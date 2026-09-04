# Lesson 15 Notes - Room Database Thinking Map

This note explains how to think when building and using a Room database.

The main question is:

```text
How do I go from real app data to Room tables, DAO functions, database class, and app usage?
```

The simple mental order is:

1. What real-world things does the app store?
2. What table should each thing become?
3. What fields should each table have?
4. What is the primary key?
5. What relationships exist between tables?
6. What operations does the app need?
7. What DAO functions support those operations?
8. What entities and DAOs must the database class include?
9. How does the repository get the database and call the DAOs?
10. What IDs must be passed between steps?

Room can feel complicated because there are several files, but each file has a different job.

```text
Entity
    describes table structure

DAO
    describes database operations

Database class
    connects multiple entities and DAOs into one Room database

Repository
    gives the rest of the app a cleaner way to call the database

ViewModel
    decides when to call the repository based on user actions

UI
    shows data and sends user events to the ViewModel
```

## 1. Start from the app's real data

Before writing Room code, think about the real things your app needs to remember.

For the research app, the important things are:

```text
Patient
Session
Measurement
Result
```

Then ask:

```text
Should this thing have its own table?
```

Usually, if the thing can exist as its own record, it should probably have its own table.

For example:

```text
One patient can have many sessions.
One session can have many measurements.
One session can have one or more results.
```

That is why Lesson 15 uses multiple tables instead of one giant measurement table.

In Android project terms, this usually becomes several Kotlin files:

```text
data/local/PatientEntity.kt
data/local/SessionEntity.kt
data/local/MeasurementEntity.kt
data/local/ResultEntity.kt

data/local/PatientDao.kt
data/local/SessionDao.kt
data/local/MeasurementDao.kt
data/local/ResultDao.kt

data/local/ResearchDatabase.kt
data/MeasurementRepository.kt
ui/ResearchViewModel.kt
ui/ResearchScreen.kt
```

You do not need to create every file at once, but this is the shape you are slowly building toward.

## 2. Design the entity first

An entity answers this question:

```text
What should one row in this table look like?
```

For example, one patient row might need:

```text
id
patientCode
notes
createdAt
```

So we create:

```kotlin
import androidx.room3.Entity
import androidx.room3.PrimaryKey
```

```kotlin
@Entity(tableName = "patients")
data class PatientEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,

    val patientCode: String,
    val notes: String = "",
    val createdAt: Long = System.currentTimeMillis()
)
```

Then one session row might need:

```text
id
patientId
sessionName
startedAt
endedAt
```

So we create:

```kotlin
@Entity(tableName = "sessions")
data class SessionEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,

    val patientId: Long,
    val sessionName: String,
    val startedAt: Long = System.currentTimeMillis(),
    val endedAt: Long? = null
)
```

Then one measurement row might need:

```text
id
sessionId
repetition
value
timestamp
status
```

So we create:

```kotlin
@Entity(tableName = "measurements")
data class MeasurementEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,

    val sessionId: Long,
    val repetition: Int,
    val value: Double,
    val timestamp: Long = System.currentTimeMillis(),
    val status: String = "OK"
)
```

When designing an entity, ask:

- Which field uniquely identifies this row?
- Which fields are shown to the user?
- Which fields connect this table to another table?
- Which fields can be empty or unknown?
- Which fields are timestamps or status values?

In this tutorial:

```text
id
    internal database ID

patientCode
    readable code for the user, such as P001

patientId
    foreign-key-style value that points to PatientEntity.id

sessionId
    foreign-key-style value that points to SessionEntity.id
```

## 3. Decide the primary key

Every Room entity needs a primary key.

The primary key answers:

```text
How does the database uniquely identify one row?
```

The common beginner-friendly choice is:

```kotlin
@PrimaryKey(autoGenerate = true)
val id: Long = 0
```

This means:

```text
When I create a new object, id starts as 0.
When Room inserts it, SQLite gives it a real ID.
```

For example:

```text
Before insert:
id = 0

After insert:
id = 1
```

This internal ID is usually not what the user sees.

The user sees:

```text
patientCode = P001
```

The database uses:

```text
id = 1
```

## 4. Think about relationships between tables

A relationship answers:

```text
How does one table point to another table?
```

For example:

```text
Session belongs to Patient.
Measurement belongs to Session.
Result belongs to Session.
```

So the child table stores the parent table's ID.

```text
SessionEntity.patientId stores PatientEntity.id.
MeasurementEntity.sessionId stores SessionEntity.id.
ResultEntity.sessionId stores SessionEntity.id.
```

This is the core logic:

```text
Parent row has id = 5.
Child row stores parentId = 5.
Now the child belongs to that parent.
```

The name `sessionId` helps humans understand the code, but the actual link is the matching number.

Room can also enforce the relationship with `ForeignKey`, but even then your app still needs to pass the correct ID when creating child rows.

In code, the basic version looks like this:

```kotlin
data class MeasurementEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,

    val sessionId: Long,
    val repetition: Int,
    val value: Double
)
```

The stricter Room-enforced version looks like this:

```kotlin
import androidx.room3.Entity
import androidx.room3.ForeignKey
import androidx.room3.Index
import androidx.room3.PrimaryKey
```

```kotlin
@Entity(
    tableName = "measurements",
    foreignKeys = [
        ForeignKey(
            entity = SessionEntity::class,
            parentColumns = ["id"],
            childColumns = ["sessionId"],
            onDelete = ForeignKey.CASCADE
        )
    ],
    indices = [
        Index(value = ["sessionId"])
    ]
)
data class MeasurementEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,

    val sessionId: Long,
    val repetition: Int,
    val value: Double
)
```

Read the foreign key from left to right:

```text
SessionEntity.id
    parent value

MeasurementEntity.sessionId
    child value that must match the parent value
```

## 5. Design the DAO after the entity

After the table structure is clear, ask:

```text
What does the app need to do with this table?
```

For a patient table, the app may need to:

```text
insert a new patient
get all patients
find one patient by patientCode
```

So the DAO has functions like:

```kotlin
import androidx.room3.Dao
import androidx.room3.Insert
import androidx.room3.Query
```

```kotlin
@Dao
interface PatientDao {

    @Insert
    suspend fun insertPatient(
        patient: PatientEntity
    ): Long

    @Query("SELECT * FROM patients ORDER BY createdAt DESC")
    suspend fun getAllPatients(): List<PatientEntity>

    @Query("SELECT * FROM patients WHERE patientCode = :patientCode LIMIT 1")
    suspend fun getPatientByCode(
        patientCode: String
    ): PatientEntity?
}
```

Think of the entity as the noun and the DAO as the verbs.

```text
Entity:
    What data exists?

DAO:
    What can we do with that data?
```

For the session table, the app may need to:

```text
insert a session
get all sessions for one patient
mark a session as ended
```

So the DAO might be:

```kotlin
@Dao
interface SessionDao {

    @Insert
    suspend fun insertSession(
        session: SessionEntity
    ): Long

    @Query("SELECT * FROM sessions WHERE patientId = :patientId ORDER BY startedAt DESC")
    suspend fun getSessionsForPatient(
        patientId: Long
    ): List<SessionEntity>

    @Query("UPDATE sessions SET endedAt = :endedAt WHERE id = :sessionId")
    suspend fun endSession(
        sessionId: Long,
        endedAt: Long
    )
}
```

For the measurement table, the app may need to:

```text
insert a measurement
get measurements for one session
delete measurements for one session
```

So the DAO might be:

```kotlin
@Dao
interface MeasurementDao {

    @Insert
    suspend fun insertMeasurement(
        measurement: MeasurementEntity
    ): Long

    @Query("SELECT * FROM measurements WHERE sessionId = :sessionId ORDER BY timestamp ASC")
    suspend fun getMeasurementsForSession(
        sessionId: Long
    ): List<MeasurementEntity>

    @Query("DELETE FROM measurements WHERE sessionId = :sessionId")
    suspend fun deleteMeasurementsForSession(
        sessionId: Long
    )
}
```

Notice how the function parameters match the SQL placeholders:

```text
:patientId uses the function parameter patientId
:sessionId uses the function parameter sessionId
:endedAt uses the function parameter endedAt
```

## 6. Create the database class

The database class answers:

```text
Which tables and DAOs belong to this Room database?
```

For Lesson 15:

```kotlin
@Database(
    entities = [
        PatientEntity::class,
        SessionEntity::class,
        MeasurementEntity::class,
        ResultEntity::class
    ],
    version = 1
)
abstract class ResearchDatabase : RoomDatabase() {
    abstract fun patientDao(): PatientDao
    abstract fun sessionDao(): SessionDao
    abstract fun measurementDao(): MeasurementDao
    abstract fun resultDao(): ResultDao
}
```

This class does not usually contain app logic.

It mainly says:

```text
These are my tables.
These are my DAO access points.
This is my database version.
```

If you create a new entity but forget to add it to `entities`, Room does not know it belongs to the database.

If you create a new DAO but forget to expose it with an abstract function, the repository cannot access it through the database object.

In a real Android app, you often create the database as a singleton so the app does not accidentally open many database instances:

```kotlin
import android.content.Context
import androidx.room3.Database
import androidx.room3.Room
import androidx.room3.RoomDatabase
```

```kotlin
@Database(
    entities = [
        PatientEntity::class,
        SessionEntity::class,
        MeasurementEntity::class,
        ResultEntity::class
    ],
    version = 1
)
abstract class ResearchDatabase : RoomDatabase() {
    abstract fun patientDao(): PatientDao
    abstract fun sessionDao(): SessionDao
    abstract fun measurementDao(): MeasurementDao
    abstract fun resultDao(): ResultDao

    companion object {
        @Volatile
        private var INSTANCE: ResearchDatabase? = null

        fun getDatabase(context: Context): ResearchDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    ResearchDatabase::class.java,
                    "research_database"
                ).build()

                INSTANCE = instance
                instance
            }
        }
    }
}
```

This is a common Room singleton pattern.

You can mostly reuse this structure in other Room projects.

The parts that usually change are:

```text
ResearchDatabase
    your database class name

"research_database"
    your database file name
```

So if your database class is called `AppDatabase`, the pattern would use:

```kotlin
private var INSTANCE: AppDatabase? = null
```

and:

```kotlin
AppDatabase::class.java
```

### Read this part slowly: companion object

```kotlin
companion object
```

means:

```text
This belongs to the ResearchDatabase class itself,
not to one specific ResearchDatabase object.
```

So other code can call:

```kotlin
ResearchDatabase.getDatabase(context)
```

without first creating a `ResearchDatabase` manually.

This line stores the one shared database object:

```kotlin
private var INSTANCE: ResearchDatabase? = null
```

At the beginning, it is `null` because the database has not been created yet.

After Room creates the database, we save it in `INSTANCE`.

Then next time, the app reuses the same database object instead of building another one.

This part:

```kotlin
return INSTANCE ?: synchronized(this) {
```

means:

```text
If INSTANCE already exists, return it.
If INSTANCE is still null, enter the synchronized block and create it.
```

The `synchronized(this)` block prevents two threads from creating the database at the same time.

This part creates the actual Room database:

```kotlin
val instance = Room.databaseBuilder(
    context.applicationContext,
    ResearchDatabase::class.java,
    "research_database"
).build()
```

The three important arguments are:

```text
context.applicationContext
    the Android app context used to create the database safely

ResearchDatabase::class.java
    the database class Room should build

"research_database"
    the database file name stored on the device
```

This part saves the newly created database:

```kotlin
INSTANCE = instance
```

And this last line returns it:

```kotlin
instance
```

The `@Volatile` annotation helps make sure different threads see the latest value of `INSTANCE`.

You do not need to fully master `@Volatile` and `synchronized` yet.

For now, remember the practical idea:

```text
Create the Room database once.
Reuse the same database instance everywhere.
```


The important idea is:

```text
Database class
    knows which entities exist
    exposes the DAO objects
    creates or returns the Room database instance
```

## 7. Use the database through the repository

After the database class exists, the app needs to use it.

The repository usually gets the database instance:

```kotlin
private val database = ResearchDatabase.getDatabase(context)
```

Then it gets DAO objects from the database:

```kotlin
private val patientDao = database.patientDao()
private val sessionDao = database.sessionDao()
private val measurementDao = database.measurementDao()
private val resultDao = database.resultDao()
```

Then repository functions call DAO functions.

A small repository function can look like this:

```kotlin
suspend fun createSessionForPatient(
    patientId: Long,
    sessionName: String
): Long {
    return sessionDao.insertSession(
        SessionEntity(
            patientId = patientId,
            sessionName = sessionName
        )
    )
}
```

A more complete repository flow can show the real multi-table logic:

```kotlin
class MeasurementRepository(
    context: Context
) {
    private val database = ResearchDatabase.getDatabase(context)

    private val patientDao = database.patientDao()
    private val sessionDao = database.sessionDao()
    private val measurementDao = database.measurementDao()

    suspend fun getPatientByCode(
        patientCode: String
    ): PatientEntity? {
        return patientDao.getPatientByCode(patientCode)
    }

    suspend fun createSessionForPatient(
        patientId: Long,
        sessionName: String
    ): Long {
        return sessionDao.insertSession(
            SessionEntity(
                patientId = patientId,
                sessionName = sessionName
            )
        )
    }

    suspend fun saveMeasurement(
        sessionId: Long,
        repetition: Int,
        value: Double
    ): Long {
        return measurementDao.insertMeasurement(
            MeasurementEntity(
                sessionId = sessionId,
                repetition = repetition,
                value = value
            )
        )
    }

    suspend fun getMeasurementsForSession(
        sessionId: Long
    ): List<MeasurementEntity> {
        return measurementDao.getMeasurementsForSession(sessionId)
    }
}
```

This repository hides the DAO details from the ViewModel.

The ViewModel does not need to know the SQL string:

```sql
SELECT * FROM measurements WHERE sessionId = :sessionId
```

It only needs to call:

```kotlin
repository.getMeasurementsForSession(sessionId)
```

So the chain is:

```text
ViewModel
    calls repository function

Repository
    calls DAO function

DAO
    runs SQL through Room

Room/SQLite
    reads or writes the database
```

## 8. Follow the ID flow in multi-table

In multi-table Room work, IDs are the important values that travel between steps.

Example: create or select patient, then create session.

```text
User selects patient code P001
        ->
App finds PatientEntity(id = 1, patientCode = "P001")
        ->
App creates SessionEntity(patientId = 1, sessionName = "Baseline")
        ->
Room inserts the session
        ->
Room returns sessionId = 10
```

Then measurements use `sessionId = 10`:

```text
App creates MeasurementEntity(sessionId = 10, value = 2.43)
        ->
Room inserts the measurement
        ->
Later, app queries measurements WHERE sessionId = 10
```

The important rule is:

```text
Create or find the parent first.
Get the parent's ID.
Put that ID into the child row.
```

For this app:

```text
Find or create Patient first.
Use patientId to create Session.
Use sessionId to create Measurement.
Use sessionId to create Result.
```

In ViewModel code, the same ID flow might look like this:

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.launch
```

```kotlin
fun onPatientCodeChange(value: String) {
    uiState = uiState.copy(
        patientCode = value
    )
}

fun onSessionNameChange(value: String) {
    uiState = uiState.copy(
        sessionName = value
    )
}

fun createSessionForSelectedPatient() {
    viewModelScope.launch { // necessary to use viewModelScope.launch because createSessionForPatient is a suspend function
        val patient = repository.getPatientByCode(
            uiState.patientCode
        )

        if (patient == null) {
            uiState = uiState.copy(
                message = "Patient not found"
            )
            return@launch
        }

        val sessionId = repository.createSessionForPatient(
            patientId = patient.id,
            sessionName = uiState.sessionName
        )

        uiState = uiState.copy(
            currentPatientId = patient.id,
            currentSessionId = sessionId,
            message = "Session created"
        )
    }
}
```

Then saving a measurement uses `currentSessionId`:

```kotlin
fun saveCurrentMeasurement(value: Double) {
    val sessionId = uiState.currentSessionId

    if (sessionId == null) {
        uiState = uiState.copy(
            message = "Create a session first"
        )
        return
    }

    viewModelScope.launch {
        repository.saveMeasurement(
            sessionId = sessionId,
            repetition = uiState.measurements.size + 1,
            value = value
        )

        val updatedMeasurements =
            repository.getMeasurementsForSession(sessionId)

        uiState = uiState.copy(
            measurements = updatedMeasurements,
            latestValue = value,
            message = "Measurement saved"
        )
    }
}
```

This is the practical Android version of:

```text
Find parent.
Get parent ID.
Create child with parent ID.
```

### Why check `sessionId == null` here?

This check is a ViewModel/UI-state check:

```kotlin
if (sessionId == null) {
    uiState = uiState.copy(
        message = "Create a session first"
    )
    return
}
```

It answers this question:

```text
Does the app currently know which session is active?
```

It does not answer this deeper database question:

```text
Does this sessionId definitely exist in the sessions table?
```

In the normal app flow, `currentSessionId` is set only after Room successfully creates or selects a session:

```kotlin
val sessionId = repository.createSessionForPatient(
    patientId = patient.id,
    sessionName = uiState.sessionName
)

uiState = uiState.copy(
    currentSessionId = sessionId
)
```

So if `currentSessionId` is not null, we usually trust that it came from the database.

There are two different safety levels:

```text
Level 1: ViewModel safety check
    Do we have an active session ID in UI state?

Level 2: Database integrity check
    Does that session ID really exist in the sessions table?
```

The ViewModel check is:

```kotlin
if (sessionId == null)
```

The database integrity check is the `ForeignKey` constraint:

```kotlin
ForeignKey(
    entity = SessionEntity::class,
    parentColumns = ["id"],
    childColumns = ["sessionId"],
    onDelete = ForeignKey.CASCADE
)
```

With the foreign key, Room/SQLite can reject a measurement if its `sessionId` does not match an existing `sessions.id`.

So the short version is:

```text
ViewModel checks whether a session is currently selected.
ForeignKey checks whether the selected session really exists in the database.
```

You could manually query the session table before every measurement insert, but usually this is unnecessary if:

```text
currentSessionId came from Room
and
the child table uses a ForeignKey constraint
```


## 9. The flow converting Readable Codes into Internal IDs

The most important transferred values are:

```text
patientCode
    user-facing code
    used to search/select a patient

patientId
    internal ID from PatientEntity.id
    used to create or query sessions

sessionId
    internal ID from SessionEntity.id
    used to create or query measurements and results

measurementId
    internal ID from MeasurementEntity.id
    useful if editing or deleting one measurement

resultId
    internal ID from ResultEntity.id
    useful if editing or deleting one result
```

A useful way to think is using this three-step mental model:

```text
Readable codes are for humans.
The app convert from the readable code to the internal ID.
Internal IDs are for database relationships.
```

### 1. Readable codes are for humans

Readable codes are the values the user can see, type, select, and understand.

For example:

```text
patientCode = "P001"
sessionName = "Baseline"
deviceCode = "D003"
```

The UI stores and displays these readable values:

```kotlin
// UI state stores both readable input and internal selected IDs.
data class ResearchUiState(
    val patientCode: String = "",
    val sessionName: String = "",
    val currentPatientId: Long? = null,
    val currentSessionId: Long? = null,
    val measurements: List<MeasurementEntity> = emptyList(),
    val latestValue: Double? = null,
    val message: String = ""
)
```

```kotlin
import androidx.compose.material3.Button
import androidx.compose.material3.Text
import androidx.compose.material3.TextField
```

```kotlin
// UI shows readable values and passes user intent to the ViewModel.
TextField(
    value = uiState.patientCode,
    onValueChange = viewModel::onPatientCodeChange,
    label = {
        Text("Patient code")
    }
)

TextField(
    value = uiState.sessionName,
    onValueChange = viewModel::onSessionNameChange,
    label = {
        Text("Session name")
    }
)

Button(
    onClick = {
        viewModel.createSessionForSelectedPatient()
    }
) {
    Text("Create Session")
}
```

At this stage, the user is thinking:

```text
I selected patient P001.
I want to create a Baseline session.
```

### 2. The app converts readable code to internal ID

The conversion does not happen inside the UI alone.

It happens across these Android layers:

```text
UI
    captures patientCode = "P001"

ViewModel
    receives the user action
    asks the repository to find the patient

Repository
    calls the DAO

DAO
    runs a query using patientCode

Room/SQLite
    returns the matching PatientEntity
```

The bottom-level DAO query from the database is:

```kotlin
@Query("SELECT * FROM patients WHERE patientCode = :patientCode LIMIT 1")
suspend fun getPatientByCode(
    patientCode: String
): PatientEntity?
```

The repository exposes that query to ViewModel as:

```kotlin
suspend fun getPatientByCode(
    patientCode: String
): PatientEntity? {
    return patientDao.getPatientByCode(patientCode)
}
```

The ViewModel performs the conversion during the user action:

```kotlin
val patient = repository.getPatientByCode(
    patientCode = uiState.patientCode
)

if (patient == null) {
    uiState = uiState.copy(
        message = "Patient not found"
    )
    return
}

val patientId = patient.id
```

If the user selected:

```text
patientCode = "P001"
```

Room might return:

```text
PatientEntity(
    id = 1,
    patientCode = "P001"
)
```

Then this line extracts the internal ID:

```kotlin
val patientId = patient.id
```

So the conversion is:

```text
Readable code from UI:
    P001

Database row found by Room:
    PatientEntity(id = 1, patientCode = "P001")

Internal ID extracted by app code:
    patientId = 1
```

### 3. Internal IDs are for database relationships

After the app has the internal ID, it uses that ID to connect tables.

For example, a session belongs to a patient, so the new `SessionEntity` must store `patientId`:

```kotlin
// ViewModel passes internal IDs to the repository.
repository.createSessionForPatient(
    patientId = patientId,
    sessionName = uiState.sessionName
)
```

```kotlin
// Repository passes entities to the DAO.
sessionDao.insertSession(
    SessionEntity(
        patientId = patientId,
        sessionName = sessionName
    )
)
```

The saved session row might look like:

```text
SessionEntity(
    id = 10,
    patientId = 1,
    sessionName = "Baseline"
)
```

Now the relationship is:

```text
PatientEntity.id = 1
matches
SessionEntity.patientId = 1
```

The same logic continues when saving measurements:

```text
SessionEntity.id = 10
matches
MeasurementEntity.sessionId = 10
```

So the full logic is:

```text
Readable code is for the human:
    patientCode = P001

The app converts it:
    find PatientEntity where patientCode = P001
    read PatientEntity.id

Internal ID is for relationships:
    patientId = 1
```

## 10. Single-table thinking versus multi-table thinking

With one table, the thinking is simple:

```text
Create MeasurementEntity.
Insert MeasurementEntity.
Read all measurements.
```

With multiple tables, the thinking becomes:

```text
Which parent row does this child row belong to?
Do I already have the parent ID?
If not, should I create the parent or select an existing parent?
After I insert a parent, do I need the returned ID?
When I query child rows, which parent ID should I filter by?
```

Example:

```text
Do not just insert a measurement.
Insert a measurement for a specific session.
```

That means this is incomplete:

```kotlin
MeasurementEntity(
    value = 2.43
)
```

This is meaningful:

```kotlin
val sessionId = currentSessionId ?: return

MeasurementEntity(
    sessionId = sessionId,
    repetition = 1,
    value = 2.43
)
```

## 11. The full Room thinking loop

When adding a new table, use this checklist:

```text
1. Name the real-world thing.
2. Decide if it needs its own table.
3. Create the Entity.
4. Choose the primary key.
5. Add normal fields.
6. Add foreign-key-style fields if it belongs to another table.
7. Decide whether to add real ForeignKey constraints.
8. Decide whether any field or field combination should be unique.
9. Create DAO functions for insert, read, update, and delete.
10. Add the entity and DAO to the database class.
11. Get the DAO inside the repository.
12. Create repository functions that the ViewModel can call.
13. Pass the correct IDs between parent and child operations.
14. Query child rows using the parent ID.
```

The short version:

```text
Entity defines the table.
DAO defines what can happen to the table.
Database class connects the Room pieces.
Repository uses the database.
ViewModel decides when to use the repository.
UI lets the user trigger the flow.
```

## 12. Most important mental model

Room database work is not only about writing annotations.

It is about answering this chain of questions:

```text
What data exists?
How is each row identified?
How do rows belong to other rows?
What operations does the app need?
Which IDs must move from one operation to the next?
```

For a research app, this is the main shape:

```text
Patient
    has many Sessions

Session
    has many Measurements
    has Results

Measurement
    must know which Session it belongs to

Result
    must know which Session it belongs to
```

If you understand the ID flow, the Room files become much less mysterious.
