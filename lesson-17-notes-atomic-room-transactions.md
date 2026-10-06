# Lesson 17 Notes - Atomic Room Transactions

This companion note expands the transaction concepts behind the multi-table Room model from Lesson 17.

It uses one research session, its measurements, its result, and its completion status to explain how Room prevents partial database saves.

---

## 1. Atomic Database Work with Room

Room may need to write several related rows when the app saves one completed research session. The session, its measurements, its result, and its completion status will be the running example in this section.

**Atomic** means that several related database changes are treated as one indivisible operation:

```text
all changes succeed together
or
none of the changes are kept
```

Atomicity is the `A` in the database term **ACID transaction**. For this lesson, the most important part is the all-or-nothing guarantee.

### The partial-save problem

Suppose completing one research session requires the app to:

```text
1. save the session
2. save 100 measurements
3. save the result
4. mark the session complete
```

If those calls run as separate database operations, a failure can occur in the middle:

```text
save session                    ✓
save measurements 1-50         ✓
save measurement 51            ✗ error
save result                     not executed
mark session complete           not executed
```

The database may now contain:

```text
session exists
only half of its measurements exist
result is missing
session is not marked complete
```

That may be invalid for a research dataset because other code cannot know whether the partial records are safe to analyse.

### What a Room transaction changes

A database transaction wraps the writes in one boundary:

```text
begin transaction
 ↓
save session
save all measurements
save result
mark session complete
 ↓
did every operation succeed?
 ├── yes → commit all changes
 └── no  → roll back all changes
```

**Commit** means make every change permanent.

**Rollback** means undo the changes made inside that transaction and restore the database to its earlier state.

If measurement 51 throws an exception, the transaction result becomes:

```text
session not saved
measurements not saved
result not saved
completion status not saved
```

The database never exposes the half-completed dataset as the final result.

A bank transfer is the classic analogy:

```text
subtract £100 from account A
add £100 to account B
```

Those two writes must succeed together. Keeping only the subtraction would leave the data incorrect.

### One DAO call is not the same as one workflow transaction

Room already runs an individual `@Insert`, `@Update`, or `@Delete` operation transactionally. For example, one `insertAll(measurements)` call will not normally insert only part of that list.

However, that does not automatically group several separate DAO or repository calls:

```kotlin
sessionRepository.saveSession(session)
measurementRepository.saveMeasurements(measurements)
resultRepository.saveResult(result)
sessionRepository.markComplete(session.id)
```

Each function may finish and commit before the next function begins. If the third call fails, the earlier calls may already be permanent.

The fact that all four functions are called from the same use case or coroutine does not create a database transaction.

```text
same function
same coroutine
same use case

do not automatically mean

same database transaction
```

### Expose one atomic data-layer operation

When the writes represent one indivisible save, expose one operation for the complete unit:

```kotlin
interface SessionCompletionRepository {
    suspend fun saveCompletedSession(
        session: SessionRecord,
        measurements: List<MeasurementRecord>,
        result: ResultRecord
    )
}
```

The interface describes the data capability needed by the application:

```text
save one completed session dataset
```

The interface is a contract, not an object and not a transaction by itself. It is used in two places later in this note:

```text
RoomSessionCompletionRepository
    implements SessionCompletionRepository

CompleteSessionUseCase
    receives SessionCompletionRepository
    calls saveCompletedSession(...)
```

The relationship will become:

```text
SessionCompletionRepository interface
        ↓ implemented by
RoomSessionCompletionRepository
        ↓ supplied to
CompleteSessionUseCase
```

The Room implementation determines how to fulfil the contract atomically.

### Room 3 implementation with `withWriteTransaction`

This tutorial uses Room 3, whose package name is `androidx.room3`.

For several high-level DAO write operations, use the database's `withWriteTransaction` function:

```kotlin
import androidx.room3.withWriteTransaction

class RoomSessionCompletionRepository(
    private val database: ResearchDatabase
) : SessionCompletionRepository {

    private val sessionDao = database.sessionDao()
    private val measurementDao = database.measurementDao()
    private val resultDao = database.resultDao()

    override suspend fun saveCompletedSession(
        session: SessionRecord,
        measurements: List<MeasurementRecord>,
        result: ResultRecord
    ) {
        // Perform mapping before opening the transaction when possible.
        val sessionEntity = session.toEntity()
        val measurementEntities = measurements.map {
            it.toEntity()
        }
        val resultEntity = result.toEntity()

        database.withWriteTransaction {
            sessionDao.insertSession(sessionEntity)
            measurementDao.insertMeasurements(measurementEntities)
            resultDao.insertResult(resultEntity)

            sessionDao.markSessionComplete(
                sessionId = session.id,
                completedAt = System.currentTimeMillis()
            )
        }
    }
}
```

This part connects the concrete class to the interface:

```kotlin
) : SessionCompletionRepository
```

This part implements the function promised by the interface:

```kotlin
override suspend fun saveCompletedSession(...)
```

Code using the interface does not need to know that this implementation uses Room. A test implementation could satisfy the same interface without using a real database.

Room commits the transaction only when the block finishes successfully. If an exception escapes the block, or the coroutine is cancelled, Room rolls the transaction back.

The DAOs may belong to separate repository areas, but they use the same `ResearchDatabase` transaction here. This is why the atomic operation needs access to the database boundary rather than merely calling unrelated repository methods one after another.

If the database generates the session ID, that ID can be used by the later inserts inside the same transaction:

```kotlin
database.withWriteTransaction {
    val sessionId = sessionDao.insertSession(sessionEntity)

    val linkedMeasurements = measurementEntities.map {
        it.copy(sessionId = sessionId)
    }

    measurementDao.insertMeasurements(linkedMeasurements)
    resultDao.insertResult(
        resultEntity.copy(sessionId = sessionId)
    )
    sessionDao.markSessionComplete(sessionId)
}
```

If a later insert fails, the newly generated session row is also rolled back.

## 2. Choosing the Narrowest Constructor Dependency

It is not generally better to inject `ResearchDatabase` into every repository for flexibility. A class should normally receive the smallest dependency that provides everything it legitimately needs.

For an ordinary repository that works with one table, inject the corresponding DAO:

```kotlin
class RoomSessionRepository(
    private val sessionDao: SessionDao
) : SessionRepository {
    // Ordinary session operations that use SessionDao.
}
```

This constructor immediately communicates the class's responsibility:

```text
RoomSessionRepository can perform SessionDao operations
RoomSessionRepository does not have unrestricted access to every table
```

This design provides several benefits:

- the dependency is easy to understand
- the repository cannot accidentally start using unrelated DAOs
- the class is easier to test with a fake DAO
- unrelated database changes are less likely to affect it
- the compiler helps preserve the repository's boundary

Giving every repository the complete database may look more flexible:

```kotlin
class RoomSessionRepository(
    private val database: ResearchDatabase
) : SessionRepository
```

However, that class can now reach every DAO exposed by the database:

```kotlin
database.sessionDao()
database.patientDao()
database.measurementDao()
database.resultDao()
```

That is broader access, not necessarily better design. It makes it easier for a small repository to accumulate unrelated responsibilities.

The multi-table transaction implementation is different. It must establish one transaction boundary shared by several DAOs, so receiving `ResearchDatabase` is appropriate:

```kotlin
class RoomSessionCompletionRepository(
    private val database: ResearchDatabase
) : SessionCompletionRepository {

    private val sessionDao = database.sessionDao()
    private val measurementDao = database.measurementDao()
    private val resultDao = database.resultDao()

    override suspend fun saveCompletedSession(
        session: SessionRecord,
        measurements: List<MeasurementRecord>,
        result: ResultRecord
    ) {
        database.withWriteTransaction {
            sessionDao.insertSession(session.toEntity())
            measurementDao.insertMeasurements(
                measurements.map { it.toEntity() }
            )
            resultDao.insertResult(result.toEntity())
            sessionDao.markSessionComplete(session.id)
        }
    }
}
```

All three DAOs are obtained from the same database instance, and that database controls their shared transaction.

It is usually unnecessary to inject both the database and every DAO into this class:

```kotlin
// Usually unnecessarily repetitive.
class RoomSessionCompletionRepository(
    private val database: ResearchDatabase,
    private val sessionDao: SessionDao,
    private val measurementDao: MeasurementDao,
    private val resultDao: ResultDao
)
```

The database is already required to open the transaction and can supply its own DAOs. Using it as the single constructor dependency also makes it clear that all participating DAOs belong to the same Room database.

Use this decision rule:

```text
One DAO is sufficient
    -> inject that specific DAO

Several specific DAOs are needed, but the class does not control a transaction
    -> inject those specific DAOs

The class must control one transaction across several DAOs
    -> inject ResearchDatabase and obtain its DAOs from it
```

For the running example, the dependencies would therefore be:

| Implementation | Constructor dependency |
|---|---|
| `RoomSessionRepository` | `SessionDao` |
| `RoomPatientRepository` | `PatientDao` |
| `RoomMeasurementRepository` | `MeasurementDao` |
| `RoomResultRepository` | `ResultDao` |
| `RoomSessionCompletionRepository` | `ResearchDatabase` |

The guiding principle is:

> Inject the narrowest dependency that provides everything the class needs. Inject the database when the class genuinely owns a multi-DAO Room transaction boundary.

## 3. Applying the Transaction Across the Application

### Room 2 compatibility note

Room 2 uses a similar idea with a different package and helper:

```kotlin
import androidx.room.withTransaction

database.withTransaction {
    sessionDao.insertSession(sessionEntity)
    measurementDao.insertMeasurements(measurementEntities)
    resultDao.insertResult(resultEntity)
}
```

Use the API that matches the Room version configured in the project. Do not mix `androidx.room3` and `androidx.room` imports.

### `@Transaction` is another option

When all related operations naturally belong to one DAO, Room's `@Transaction` annotation can wrap a concrete DAO function:

```kotlin
import androidx.room3.Dao
import androidx.room3.Insert
import androidx.room3.Transaction

@Dao
abstract class CompletedSessionDao {

    @Insert
    protected abstract suspend fun insertSession(
        session: SessionEntity
    )

    @Insert
    protected abstract suspend fun insertMeasurements(
        measurements: List<MeasurementEntity>
    )

    @Insert
    protected abstract suspend fun insertResult(
        result: ResultEntity
    )

    @Transaction
    open suspend fun insertCompletedSession(
        session: SessionEntity,
        measurements: List<MeasurementEntity>,
        result: ResultEntity
    ) {
        insertSession(session)
        insertMeasurements(measurements)
        insertResult(result)
    }
}
```

Use `withWriteTransaction` when a data-layer operation needs to coordinate several existing DAOs. Use `@Transaction` when the complete operation fits naturally inside one DAO.

### The use case requests atomic work but does not implement Room

The domain use case may decide that completing a session is one business operation:

```kotlin
class CompleteSessionUseCase(
    private val completionRepository:
        SessionCompletionRepository
) {
    suspend operator fun invoke(
        session: SessionRecord,
        measurements: List<MeasurementRecord>,
        result: ResultRecord
    ) {
        require(measurements.isNotEmpty()) {
            "A completed session must contain measurements"
        }

        completionRepository.saveCompletedSession(
            session = session,
            measurements = measurements,
            result = result
        )
    }
}
```

#### Read `suspend operator fun invoke`

These keywords have separate meanings:

```kotlin
suspend operator fun invoke(...)
│       │        │   └── special function name
│       │        └── declares a function
│       └── enables special call syntax
└── allows suspension and calls to other suspend functions
```

`saveCompletedSession()` is declared as a suspending function:

```kotlin
suspend fun saveCompletedSession(...)
```

Because `invoke()` calls it directly, `invoke()` must also be `suspend` unless it starts another coroutine. `suspend` does not automatically move work to a background thread; it means the function can suspend and must be called from another suspending function or a coroutine.

`operator` is optional. It allows an object containing `invoke()` to be called like a function:

```kotlin
completeSessionUseCase(
    session,
    measurements,
    result
)
```

Kotlin interprets that as:

```kotlin
completeSessionUseCase.invoke(
    session,
    measurements,
    result
)
```

The repository does not need `operator` because it is called through its normal named function:

```kotlin
completionRepository.saveCompletedSession(...)
```

The use case could avoid `operator` and use an ordinary function name instead:

```kotlin
class CompleteSessionUseCase(
    private val completionRepository:
        SessionCompletionRepository
) {
    suspend fun complete(
        session: SessionRecord,
        measurements: List<MeasurementRecord>,
        result: ResultRecord
    ) {
        completionRepository.saveCompletedSession(
            session,
            measurements,
            result
        )
    }
}
```

That version is called with:

```kotlin
completeSessionUseCase.complete(
    session,
    measurements,
    result
)
```

Both styles are valid. `operator fun invoke` is a common use-case convention, not a requirement for calling a repository.

The responsibilities are:

```text
CompleteSessionUseCase
    checks business rules
    requests one completed-session save

RoomSessionCompletionRepository
    maps domain records to Room entities
    opens the transaction
    calls the required DAOs
    guarantees commit or rollback
```

The use case should not import `RoomDatabase`, `@Transaction`, or `withWriteTransaction`. Those are data-layer implementation details.

### Initialize and connect the implementation

The interface, implementation, and use case must be connected when the application creates its dependencies:

```kotlin
val completionRepository: SessionCompletionRepository =
    RoomSessionCompletionRepository(
        database = database
    )

val completeSessionUseCase =
    CompleteSessionUseCase(
        completionRepository = completionRepository
    )
```

Read the first declaration from right to left:

```text
RoomSessionCompletionRepository(...)
    creates the concrete Room object

: SessionCompletionRepository
    stores that object using the interface type
```

Dependency injection libraries such as Hilt can perform this wiring later, but the relationship is the same.

The ViewModel can call the suspending use case from `viewModelScope`:

```kotlin
viewModelScope.launch {
    try {
        completeSessionUseCase(
            session = session,
            measurements = measurements,
            result = result
        )

        updateSaveSuccessfulState()
    } catch (exception: Exception) {
        updateSaveFailedState(exception.message)
    }
}
```

The call path is now explicit:

```text
ViewModel coroutine
 ↓
CompleteSessionUseCase.invoke(...)
 ↓
SessionCompletionRepository.saveCompletedSession(...)
 ↓ actual object is RoomSessionCompletionRepository
RoomSessionCompletionRepository.saveCompletedSession(...)
 ↓
database.withWriteTransaction { DAO calls }
```

### Which layer and folder should contain this code?

The technical Room transaction belongs to the data layer because it depends on `ResearchDatabase`, Room entities, DAOs, and `withWriteTransaction`.

A stricter Clean Architecture package structure can separate the contract from its implementation:

```text
domain/
├── repository/
│   └── SessionCompletionRepository.kt // interface
└── usecase/
    └── CompleteSessionUseCase.kt

data/
├── repository/
│   └── RoomSessionCompletionRepository.kt // implementation
└── local/
    ├── ResearchDatabase.kt
    └── dao/
        ├── SessionDao.kt
        ├── MeasurementDao.kt
        └── ResultDao.kt
```

`RoomSessionCompletionRepository` is the implementation of the `SessionCompletionRepository` interface. They have closely related names because they represent the implementation and contract for the same capability.

In this structure:

| File | Layer responsibility |
|---|---|
| `CompleteSessionUseCase.kt` | Domain/application business rules |
| `SessionCompletionRepository.kt` | Storage-independent contract used by the domain |
| `RoomSessionCompletionRepository.kt` | Data-layer Room implementation |
| `ResearchDatabase.kt` and DAOs | Local Room database access |

A simpler Android project can keep the interface and implementation together:

```text
data/repository/
├── SessionCompletionRepository.kt
└── RoomSessionCompletionRepository.kt
```

Both structures are acceptable. The essential rule is:

```text
ViewModel and use case
    do not contain withWriteTransaction

Room repository implementation
    contains withWriteTransaction
```

You also do not need a separate repository class for every atomic function. If completing a session is naturally part of `SessionRepository`, add the operation there:

```kotlin
interface SessionRepository {
    suspend fun findSession(id: Long): SessionRecord?

    suspend fun saveCompletedSession(
        session: SessionRecord,
        measurements: List<MeasurementRecord>,
        result: ResultRecord
    )
}
```

Its Room implementation can receive `ResearchDatabase`, obtain the required DAOs from that database, and control their shared transaction. Create a separate `SessionCompletionRepository` only when completion is a sufficiently distinct data responsibility.

### Let exceptions escape the transaction block

Room knows it must roll back when the transaction block fails. Be careful not to catch an exception inside the block and pretend that the operation succeeded:

```kotlin
// Avoid this.
database.withWriteTransaction {
    try {
        sessionDao.insertSession(sessionEntity)
        measurementDao.insertMeasurements(measurementEntities)
        resultDao.insertResult(resultEntity)
    } catch (exception: Exception) {
        // Swallowing the exception can let the block finish normally.
    }
}
```

Instead, let the exception escape, or rethrow it:

```kotlin
try {
    database.withWriteTransaction {
        sessionDao.insertSession(sessionEntity)
        measurementDao.insertMeasurements(measurementEntities)
        resultDao.insertResult(resultEntity)
    }
} catch (exception: Exception) {
    // The transaction has rolled back.
    // Convert or report the error outside the transaction.
    throw SessionSaveException(
        message = "Could not save the completed session",
        cause = exception
    )
}
```

### Keep a Room transaction short

A transaction should contain only the database work that must commit together.

Do not keep a Room transaction open while the app:

- scans for a Bluetooth device
- waits for sensor packets
- records a ten-minute session
- performs network requests
- runs long signal processing
- waits for user input

Bad structure:

```text
begin database transaction
 ↓
connect Bluetooth device
 ↓
record for ten minutes
 ↓
process signal
 ↓
save database rows
 ↓
commit transaction
```

This holds a limited database write connection for far too long and can block other database work.

Better structure:

```text
collect sensor data
 ↓
stop acquisition
 ↓
process and validate the dataset
 ↓
map the final records to entities
 ↓
open a short database transaction
 ↓
perform only the related writes
 ↓
commit
```

### Live acquisition may intentionally save incrementally

Not every research session should wait until the end before saving anything. Saving measurements incrementally can protect data if the app or tablet stops unexpectedly.

One possible design is:

```text
create session with status IN_PROGRESS
 ↓
save each measurement or small batch safely during acquisition
 ↓
when acquisition finishes, use a short transaction to:
    save final result
    save final statistics
    mark session COMPLETE
```

If the app fails halfway through, the database contains an explicitly `IN_PROGRESS` or `INTERRUPTED` session rather than pretending that the dataset is complete.

This is often better than holding one transaction for the entire acquisition.

Atomicity does not mean that the whole user workflow must always be one enormous transaction. It means choosing the correct database boundary for the invariants the app must protect.

### Room transactions cannot roll back external hardware

A Room transaction controls only Room database changes. It cannot undo:

- a Bluetooth command already sent to the sensor
- a file already exported
- a network request already accepted by a server
- a physical measurement already performed

For example:

```text
send "clear device memory" over Bluetooth
 ↓
database save fails
```

Rolling back the database cannot restore the cleared device memory.

Workflows involving both the database and an external system may require:

- careful ordering
- retryable operations
- idempotent commands
- an `IN_PROGRESS`, `FAILED`, or `PENDING_SYNC` status
- a compensating action

Do not assume one Room transaction makes Bluetooth, files, and network work atomic together.

### When should Room writes be atomic?

Use one transaction when partial completion would violate a data rule.

Examples:

```text
session row and its required result must appear together
all measurements in one imported file must be accepted or rejected together
result and "session complete" status must agree
replacing calibration records must not expose half of the new calibration
```

Separate operations may be better when partial progress is useful or intentional.

Examples:

```text
live measurements saved incrementally
an optional note saved after the main session
independent user preferences
retryable export status updated separately
```

Ask:

```text
If operation 3 fails after operations 1 and 2 succeed,
is the remaining database state still valid and understandable?

No  → use one transaction
Yes → separate operations may be appropriate
```

### Test the Room rollback behavior

Test both the success and failure paths of an atomic operation.

Success test:

```text
call saveCompletedSession()
 ↓
session exists
all measurements exist
result exists
session status is COMPLETE
```

Failure test:

```text
force one DAO operation to fail
 ↓
call saveCompletedSession()
 ↓
verify that earlier writes from the transaction do not remain
```

For Room integration tests, use a test database and deliberately supply invalid data or another controlled failure that causes one operation inside the transaction to throw.

### Atomic Room-work mental model

```text
CompleteSessionUseCase
    decides what business operation must happen

saveCompletedSession(...)
    is the app-defined data-layer atomic operation
    defines which Room writes belong together

database.withWriteTransaction { ... }
    is Room's technical transaction mechanism
    supplies begin, commit, and rollback

DAO functions
    perform the individual table operations
```

The data-layer atomic operation and the Room transaction are not the same thing:

| Part | Question it answers |
|---|---|
| `saveCompletedSession(...)` | Which application data must be saved together? |
| `withWriteTransaction { ... }` | How will Room guarantee all-or-nothing execution? |
| DAO functions | Which individual table writes must be executed? |

`saveCompletedSession(...)` has application meaning. It groups session, measurement, result, and completion writes into one meaningful operation.

`withWriteTransaction { ... }` has database meaning. It does not understand what a completed research session is; it only guarantees that the DAO calls inside its block commit or roll back together.

The interface declaration alone cannot technically guarantee atomicity:

```kotlin
interface SessionCompletionRepository {
    suspend fun saveCompletedSession(...)
}
```

Its Room implementation must fulfil that promise using the transaction mechanism:

```kotlin
override suspend fun saveCompletedSession(...) {
    database.withWriteTransaction {
        // All required DAO writes
    }
}
```

This separation is useful because another implementation could use a different storage system and provide its own atomic mechanism while preserving the same application operation.

The concise rule is:

> If related Room writes would leave invalid data when only some succeed, expose one data-layer operation and execute those writes inside one short Room transaction.

For current Room transaction APIs, see [Access data using Room DAOs](https://developer.android.com/training/data-storage/room/accessing-data) and the [`RoomDatabase` transaction helpers](https://developer.android.com/reference/androidx/room3/RoomDatabaseKt).

