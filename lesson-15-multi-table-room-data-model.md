# Lesson 15 — Multi-Table Room Data Model

In Lesson 14, we introduced **Room database**.

The app moved from simple file-based storage toward structured local storage:

```text
ResearchScreen
 ↓
ResearchViewModel
 ↓
MeasurementRepository
 ↓
MeasurementDao
 ↓
ResearchDatabase / Room
```

At that point, we only had one main table:

```text
Measurement
```

That is useful for learning, but a real research app usually needs more structure.

For example, your app may need to know:

```text
Which patient/sample does this data belong to?
Which session was this measurement collected in?
When did the session start?
When did it end?
What result or ML prediction was produced?
```

So in Lesson 15, we move from:

```text
one flat Measurement table
```

to a more realistic research-app data model with multiple tabless:

```text
Patient
 ↓
Session
 ↓
Measurement
 ↓
Result
```

---

## 1. Why data modelling matters

So far, our measurement object looked something like this:

```kotlin
data class Measurement(
    val id: Long = 0,
    val sampleId: String,
    val repetition: Int,
    val value: Double,
    val timestamp: Long,
    val status: String
)
```

This works for a small prototype.

But as the app grows, `sampleId` alone is not enough.

Imagine you collect data like this:

```text
Patient P001
 ├── Session 1: baseline measurement
 │    ├── Measurement 1
 │    ├── Measurement 2
 │    └── Result
 │
 └── Session 2: after treatment
      ├── Measurement 1
      ├── Measurement 2
      └── Result
```

If we only store `sampleId` inside every measurement, the data becomes harder to organise.

A better structure is:

```text
Patient
 ↓
has many Sessions

Session
 ↓
has many Measurements

Session
 ↓
may have one or more Results
```

This is a very common structure in research apps.

---

## 2. Patient, participant, sample, or subject?

In this tutorial, I will use the word:

```text
Patient
```

because your planned app may be related to disease detection.

But depending on your research context, the name could also be:

```text
Participant
Subject
Sample
UniformSample
Specimen
ExperimentItem
```

The technical idea is the same.

For now:

```text
Patient = the main thing/person/sample being measured
Session = one data-collection event
Measurement = one recorded value or data point
Result = processed output or ML prediction
```

If your app is not clinical, you can rename `Patient` to `Participant` or `Sample`.

---

## 3. The relationship model

The structure we want is:

```text
Patient
 └── Session
      ├── Measurement
      ├── Measurement
      ├── Measurement
      └── Result
```

In database thinking:

```text
One Patient can have many Sessions.

One Session can have many Measurements.

One Session can have one Result, or many Results depending on the app.
```

This is different from putting everything into one giant class.

A beginner mistake would be:

```kotlin
data class Patient(
    val patientId: String,
    val sessions: List<Session>
)
```

This looks natural in Kotlin, but it is not the best way to store data in Room.

In a database, we normally store them as separate tables and connect them using IDs.

---

## 4. Main idea: connect tables using IDs

Instead of nesting lists directly, we use IDs.

For example:

```text
Patient
 id = 1
 patientCode = "P001"

Session
 id = 10
 patientId = 1

Measurement
 id = 100
 sessionId = 10
 value = 2.45
```

This means:

```text
Measurement 100 belongs to Session 10.
Session 10 belongs to Patient 1.
```

The link is created by:

```text
patientId
sessionId
```

This is one of the most important database ideas.

### Why `sessionId` does not appear inside the Session table
It is noticed that inside the session table (parent table), the session's ID is just called `id`.

```kotlin
@Entity(tableName = "sessions")
data class SessionEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,

    val patientId: Long,
    val sessionName: String
)
```
That `id` is the session's own primary key.

A common question is:

```text
Now that there is no column called sessionId inside the sessions table, how do we know measurement.sessionId points to the session table's id?
```

Inside the measurement table (child table), each measurement also has its own `id`, so we need another column to store the ID of the session it belongs to:

```kotlin
@Entity(tableName = "measurements")
data class MeasurementEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,

    val sessionId: Long,
    val repetition: Int,
    val value: Double
)
```

So the two IDs mean different things:

```text
MeasurementEntity.id
    the measurement's own ID

MeasurementEntity.sessionId
    the ID of the SessionEntity that owns this measurement
```

Example:

```text
sessions table

id    patientId    sessionName
1     10           Morning run
2     10           Evening run

measurements table

id     sessionId    value
101    1            2.43
102    1            2.51
103    2            3.12
```

This means:

```text
Measurements 101 and 102 belong to Session 1.
Measurement 103 belongs to Session 2.
```

The column does not need to be called `sessionsId`.

The important part is not the exact name.

The important part is:

```text
measurements.sessionId stores the same kind of value as sessions.id
```

Then a query can compare those numbers:

```sql
SELECT * FROM measurements WHERE sessionId = :sessionId
```

If `:sessionId` is `1`, Room returns all measurements where `sessionId` is `1`.

At this basic level, the relationship works because your Kotlin code saves the correct number into `sessionId`.

---

## 5. Patient entity

Let us start with the patient table.

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

The meaning is:

```text
id
 ↓
database-generated unique ID

patientCode
 ↓
human-readable patient/sample ID, for example P001

notes
 ↓
optional notes

createdAt
 ↓
when this patient record was created
```

Example:

```kotlin
val patient = PatientEntity(
    patientCode = "P001",
    notes = "Baseline test subject"
)
```

For your own app, `patientCode` could be something like:

```text
P001
S001
D1-ETO-W0-U1-S1
Uniform-01
Sample-A03
```

### Internal ID versus readable code

In this design, `id` and `patientCode` are not the same thing.

```text
id
    internal database ID
    generated by Room/SQLite
    used for relationships between tables

patientCode
    readable code shown to the user
    chosen by the researcher or app
    useful in the UI, notes, and export files
```

For example:

```text
id = 1
patientCode = "P001"
```

The user should normally see and select:

```text
P001
```

But the database should usually link tables using:

```text
patientId = 1
```

**Note that the patientCode should be unique for each patientid.** 



So when the users select a patientCode, the app flow is: 

```text
User selects patient code P001
        ->
App searches the patients table for patientCode = "P001"
        ->
Room returns the matching PatientEntity table
        ->
The app reads its corresponding internal id, for example 1
        ->
Future sessions store patientId = 1
```

Important:

```text
Room can auto-generate primary keys.
Room does not auto-fill foreign keys.
```

That means Room can generate `PatientEntity.id`, but your app must take that generated ID and put it into `SessionEntity.patientId`.

---

## 6. Session entity

A session means one data-collection event.

For example:

```text
Patient P001
Session 1: morning test
Session 2: afternoon test
Session 3: follow-up test
```

Create:

```kotlin
@Entity(tableName = "sessions")
data class SessionEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,

    val patientId: Long,
    val sessionName: String,
    val startedAt: Long = System.currentTimeMillis(),
    val endedAt: Long? = null,
    val notes: String = ""
)
```

Notice:

```kotlin
val patientId: Long
```

This connects the session to a patient.

So if:

```text
patientId = 1
```

then this session belongs to the patient whose database ID is `1`.

Also notice:

```kotlin
val endedAt: Long? = null
```

This is nullable because when a session starts, it may not have ended yet.

That connects back to Kotlin null safety:

```text
Long
 ↓
must have a value

Long?
 ↓
may have a value or may be null
```

### What if we did not auto-generate the session ID?

For learning purposes, a table can also use a primary key made from multiple columns.

This is called a composite primary key.

For example, a measurement session could theoretically be identified by:

```text
patientId + recordingDay
```

or, in a later version with devices:

```text
patientId + deviceId + recordingDay
```

In Room, that style looks like this:

```kotlin
@Entity(
    tableName = "sessions",
    primaryKeys = ["patientId", "recordingDay"]
)
data class SessionEntity(
    val patientId: Long,
    val recordingDay: String,
    val sessionName: String,
    val startedAt: Long = System.currentTimeMillis()
)
```

This says:

```text
The same patient cannot have two sessions on the same recording day.
```

But for a real research app, an auto-generated session ID is usually easier and safer:

```text
id = 10
```

Then the other fields are normal information about the session:

```text
patientId = 1
recordingDay = 2026-09-04
sessionName = "Morning test"
```

This is more flexible if:

```text
the same patient has two sessions on the same day
the recording day was entered incorrectly
the device assignment changes later
the session needs a stable identity for export or analysis
```

So this tutorial uses:

```text
SessionEntity.id
    primary key

SessionEntity.patientId
    foreign key to PatientEntity.id
```

### Optional uniqueness rule

There is also a middle option.

We can keep the auto-generated session ID as the primary key:

```text
id = 10
```

and still tell the database:

```text
Do not allow the same patient/device/day combination twice.
```

In SQL, that idea is:

```sql
UNIQUE(patient_id, device_id, recording_day)
```

In Room, we usually express this with a unique index.

For example, in a later version with devices:

```kotlin
import androidx.room3.Index
```

```kotlin
@Entity(
    tableName = "sessions",
    indices = [
        Index(
            value = ["patientId", "deviceId", "recordingDay"],
            unique = true
        )
    ]
)
data class SessionEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,

    val patientId: Long,
    val deviceId: Long,
    val recordingDay: String,
    val sessionName: String,
    val startedAt: Long = System.currentTimeMillis()
)
```

This means:

```text
id
    stable primary key for this session

patientId + deviceId + recordingDay
    combination that must not repeat
```

So the database allows:

```text
Session 10: patientId = 1, deviceId = 3, recordingDay = 2026-09-04
Session 11: patientId = 1, deviceId = 3, recordingDay = 2026-09-05
```

But it rejects another row with exactly the same combination:

```text
patientId = 1
deviceId = 3
recordingDay = 2026-09-04
```

Only add this rule if your research design really says:

```text
One patient can only have one session per device per recording day.
```

If the same patient might be measured twice on the same device on the same day, do not add this unique rule.

---

## 7. Measurement entity

Now we update the measurement table.

Before, `Measurement` had `sampleId`.

Now the measurement should belong to a session.

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

The key change is:

```kotlin
val sessionId: Long
```

This means:

```text
This measurement belongs to a specific session.
```

But there is an important detail:

```text
The name sessionId alone does not automatically tell SQLite that it points to the sessions table.
```

Without any extra Room annotation, `sessionId` is just a normal `Long` column, not automatically refers to the Id in the session table.

The name is meaningful to us as Kotlin programmers, but SQLite does not look at the name and automatically understand:

```text
sessionId means sessions.id
```

At the basic level, the relationship is created because the app saves the same number in two places:

```text
sessions.id
measurements.sessionId
```

For example, first the app creates a session:

```kotlin
val newSessionId: Long = sessionDao.insertSession(
    SessionEntity(
        patientId = patientId,
        sessionName = "Morning test"
    )
)
```

Room inserts the new session and returns its generated ID.

For example:

```text
newSessionId = 5
```

That means the session row is now something like:

```text
sessions table

id    patientId    sessionName
5     1            Morning test
```

Then the app creates a measurement and puts that same ID into `sessionId`:

```kotlin
val measurement = MeasurementEntity(
    sessionId = newSessionId,
    repetition = 1,
    value = 2.43
)
```

Because `newSessionId` is `5`, the measurement is saved like this:

```text
measurements table

id     sessionId    repetition    value
101    5            1             2.43
```

Now the link is just a matching number:

```text
measurements.sessionId = 5
matches
sessions.id = 5
```

Later, when we ask for measurements from Session 5, the query simply looks for rows where `sessionId` is `5`:

```sql
SELECT * FROM measurements WHERE sessionId = 5
```

So SQLite is not guessing the relationship from the name `sessionId`.

Your app created the relationship by saving the correct session ID into the measurement row.

So there are two levels:

```text
Level 1:
Your Kotlin code links the rows by saving the correct ID.

Level 2:
Room can enforce that link with a foreign key constraint.
```

### Optional stricter version with `ForeignKey`

If we want Room and SQLite to enforce the relationship, we can declare a foreign key.

That means:

```text
measurement.sessionId must refer to an existing session.id
```

First import `ForeignKey` and `Index`:

```kotlin
import androidx.room3.ForeignKey
import androidx.room3.Index
```

Then write the entity like this:

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
    val value: Double,
    val timestamp: Long = System.currentTimeMillis(),
    val status: String = "OK"
)
```

This tells Room exactly how the tables connect:

```text
entity = SessionEntity::class
    the parent table is sessions

parentColumns = ["id"]
    use the id column from SessionEntity

childColumns = ["sessionId"]
    use the sessionId column from MeasurementEntity

onDelete = ForeignKey.CASCADE
    if a session is deleted, delete its measurements too

indices = [Index(value = ["sessionId"])]
    helps Room search measurements by sessionId efficiently
```

Now Room knows that:

```text
measurements.sessionId points to sessions.id
```

With this constraint, Room helps protect your data. For example, it prevents you from saving a measurement with `sessionId = 999` if there is no session whose `id` is `999`.

So the flow becomes:

```text
Patient
 ↓
Session
 ↓
Measurement
```

This is cleaner than repeating `sampleId` in every measurement.

---

## 8. Result entity

A result may represent:

```text
classification result
disease detection result
model confidence
processed score
summary statistic
final decision
```

For example:

```kotlin
@Entity(tableName = "results")
data class ResultEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,

    val sessionId: Long,
    val label: String,
    val confidence: Double,
    val createdAt: Long = System.currentTimeMillis()
)
```

Example result:

```kotlin
val result = ResultEntity(
    sessionId = 10,
    label = "Positive",
    confidence = 0.92
)
```

or for a non-clinical research app:

```kotlin
val result = ResultEntity(
    sessionId = 10,
    label = "Class A",
    confidence = 0.87
)
```

The result belongs to a session through:

```kotlin
val sessionId: Long
```

This is the same idea as `MeasurementEntity.sessionId`.

The column name tells Kotlin programmers what the value means, but Room only enforces the relationship if we add a `ForeignKey` constraint.

---

## 9. Updated database structure

Now the Room database has four entities:

```text
PatientEntity
SessionEntity
MeasurementEntity
ResultEntity
```

So the database class becomes:

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

The structure is now much closer to a real research app.

---

## 10. DAOs for each table

In Lesson 14, we only had `MeasurementDao`.

Now we can create more DAOs.

### PatientDao

```kotlin
@Dao
interface PatientDao {

    @Insert
    suspend fun insertPatient(
        patient: PatientEntity
    ): Long

    @Query("SELECT * FROM patients ORDER BY createdAt DESC")
    suspend fun getAllPatients(): List<PatientEntity>
}
```

Important detail:

```kotlin
suspend fun insertPatient(...): Long
```

This returns the new patient ID generated by Room.

That is useful because we need the patient ID when creating a session.

---

### SessionDao

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

This allows us to:

```text
create a session
get all sessions for one patient
mark a session as ended
```

The `:patientId` syntax means:

```text
Use the function parameter patientId inside the SQL query.
```

---

### MeasurementDao

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

Now we no longer read all measurements from the whole app.

Instead, we normally read measurements for a specific session:

```text
getMeasurementsForSession(sessionId)
```

This is much more realistic.

---

### ResultDao

```kotlin
@Dao
interface ResultDao {

    @Insert
    suspend fun insertResult(
        result: ResultEntity
    ): Long

    @Query("SELECT * FROM results WHERE sessionId = :sessionId ORDER BY createdAt DESC")
    suspend fun getResultsForSession(
        sessionId: Long
    ): List<ResultEntity>
}
```

This lets us save and retrieve model or analysis results for a session.

---

## 11. The new research flow

Now the app logic becomes more structured.

Instead of:

```text
enter sample ID
 ↓
start collecting measurements
```

we now think:

```text
create or select patient
 ↓
create new session
 ↓
start acquisition for that session
 ↓
save measurements with sessionId
 ↓
stop session
 ↓
save result
```

This is much closer to a real research app.

---

## 12. Example: create patient and session

Imagine the user enters:

```text
Patient code: P001
Session name: Baseline
```

The repository can do:

```kotlin
suspend fun createPatientAndSession(
    patientCode: String,
    sessionName: String
): Long {
    val patientId = patientDao.insertPatient(
        PatientEntity(
            patientCode = patientCode
        )
    )

    val sessionId = sessionDao.insertSession(
        SessionEntity(
            patientId = patientId,
            sessionName = sessionName
        )
    )

    return sessionId
}
```

This returns:

```text
sessionId
```

That session ID is important.

When we collect measurements, every measurement should include that `sessionId`.

---

## 13. Store current session in UI state

The app needs to know which session is currently active.

So add this to `ResearchUiState`:

```kotlin
val currentPatientId: Long? = null
val currentSessionId: Long? = null
```

Now:

```kotlin
data class ResearchUiState(
    val patientCode: String = "",
    val sessionName: String = "",
    val currentPatientId: Long? = null,
    val currentSessionId: Long? = null,

    val deviceConnectionState: DeviceConnectionState = DeviceConnectionState.DISCONNECTED,
    val acquisitionState: AcquisitionState = AcquisitionState.IDLE,

    val measurements: List<MeasurementEntity> = emptyList(),
    val latestValue: Double? = null,

    val isLoading: Boolean = false,
    val message: String = ""
)
```

Again, these IDs are nullable because before the user creates/selects a patient and session, there is no current ID yet.

```text
currentSessionId = null
 ↓
No active session yet

currentSessionId = 10
 ↓
Measurements should be saved under Session 10
```

---

## 14. Start acquisition should require a session

In Lesson 12, we checked:

```text
Is the device connected?
```

Now we also check:

```text
Is there an active session?
```

So `startAcquisition()` should include:

```kotlin
val sessionId = uiState.currentSessionId

if (sessionId == null) {
    uiState = uiState.copy(
        message = "Create a session before starting acquisition"
    )
    return
}
```

Then full logic:

```kotlin
fun startAcquisition() {
    val sessionId = uiState.currentSessionId

    if (sessionId == null) {
        uiState = uiState.copy(
            message = "Create a session before starting acquisition"
        )
        return
    }

    if (uiState.deviceConnectionState != DeviceConnectionState.CONNECTED) {
        uiState = uiState.copy(
            message = "Connect device before starting acquisition"
        )
        return
    }

    if (uiState.acquisitionState == AcquisitionState.RECORDING) {
        return
    }

    uiState = uiState.copy(
        acquisitionState = AcquisitionState.RECORDING,
        message = "Acquisition started"
    )

    viewModelScope.launch {
        while (uiState.acquisitionState == AcquisitionState.RECORDING) {
            addSimulatedMeasurement(
                sessionId = sessionId
            )
            delay(1000)
        }
    }
}
```

Now the app prevents a serious research mistake:

```text
collecting measurements without a session identity
```

---

## 15. Add measurement with session ID

Previously, a measurement used `sampleId`.

Now it should use `sessionId`.

```kotlin
private fun addSimulatedMeasurement(
    sessionId: Long
) {
    val newMeasurement =
        measurementRepository.createSimulatedMeasurement(
            sessionId = sessionId,
            repetition = uiState.measurements.size + 1
        )

    viewModelScope.launch {
        try {
            measurementRepository.insertMeasurement(newMeasurement)

            val updatedMeasurements =
                measurementRepository.getMeasurementsForSession(sessionId)

            uiState = uiState.copy(
                measurements = updatedMeasurements,
                latestValue = newMeasurement.value,
                message = "Measurement saved"
            )
        } catch (e: Exception) {
            uiState = uiState.copy(
                message = "Measurement save failed"
            )
        }
    }
}
```

And the repository function becomes:

```kotlin
fun createSimulatedMeasurement(
    sessionId: Long,
    repetition: Int
): MeasurementEntity {
    val value = Random.nextDouble(0.0, 5.0)

    return MeasurementEntity(
        sessionId = sessionId,
        repetition = repetition,
        value = value,
        timestamp = System.currentTimeMillis(),
        status = "OK"
    )
}
```

Now the measurement is clearly connected to a session.

---

## 16. End session when stopping acquisition

When the user stops acquisition, we can mark the session as ended.

```kotlin
fun stopAcquisition() {
    if (uiState.acquisitionState != AcquisitionState.RECORDING) {
        return
    }

    val sessionId = uiState.currentSessionId

    uiState = uiState.copy(
        acquisitionState = AcquisitionState.STOPPED,
        message = "Acquisition stopped"
    )

    if (sessionId != null) {
        viewModelScope.launch {
            measurementRepository.endSession(
                sessionId = sessionId,
                endedAt = System.currentTimeMillis()
            )
        }
    }
}
```

This is better than only stopping the loop.

Now the database also knows:

```text
when the session ended
```

This is useful later for:

```text
duration calculation
export metadata
experiment log
quality checking
```

---

## 17. Repository after Lesson 15

The repository now becomes more important.

It may contain functions like:

```kotlin
class MeasurementRepository(
    context: Context
) {
    private val database = Room.databaseBuilder(
        context.applicationContext,
        ResearchDatabase::class.java,
        "research_database"
    ).build()

    private val patientDao = database.patientDao()
    private val sessionDao = database.sessionDao()
    private val measurementDao = database.measurementDao()
    private val resultDao = database.resultDao()

    suspend fun createPatient(
        patientCode: String
    ): Long {
        return patientDao.insertPatient(
            PatientEntity(
                patientCode = patientCode
            )
        )
    }

    suspend fun createSession(
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

    suspend fun endSession(
        sessionId: Long,
        endedAt: Long
    ) {
        sessionDao.endSession(
            sessionId = sessionId,
            endedAt = endedAt
        )
    }

    fun createSimulatedMeasurement(
        sessionId: Long,
        repetition: Int
    ): MeasurementEntity {
        val value = Random.nextDouble(0.0, 5.0)

        return MeasurementEntity(
            sessionId = sessionId,
            repetition = repetition,
            value = value,
            timestamp = System.currentTimeMillis(),
            status = "OK"
        )
    }

    suspend fun insertMeasurement(
        measurement: MeasurementEntity
    ) {
        measurementDao.insertMeasurement(measurement)
    }

    suspend fun getMeasurementsForSession(
        sessionId: Long
    ): List<MeasurementEntity> {
        return measurementDao.getMeasurementsForSession(sessionId)
    }

    suspend fun saveResult(
        result: ResultEntity
    ) {
        resultDao.insertResult(result)
    }
}
```

This repository now handles the research data structure.

The ViewModel does not need to know the SQL queries.

---

## 18. What the UI should show now

The UI should gradually move from:

```text
Sample ID
Current value
Start Acquisition
Measurement history
```

to:

```text
Patient code
Session name
Create Session

Device status
Acquisition status

Latest value
Measurement count
Start Acquisition
Stop Acquisition

Measurement history
```

A simple layout idea:

```text
Research Measurement App

Patient code:
[ P001 ]

Session name:
[ Baseline ]

[ Create Session ]

Current session:
Session ID: 10

Device status:
Device connected

Acquisition status:
Stopped

Latest value:
2.438

Measurements:
15

[ Start Acquisition ] [ Stop Acquisition ]
```

This is much closer to a real data-collection app.

---

## 19. Important research-app rule

From now on, a measurement should not exist alone.

A measurement should belong to a session.

So the rule is:

```text
No active session
 ↓
No acquisition
```

This is very important.

Otherwise, you may collect data but later not know:

```text
who it belongs to
when it was collected
what condition it was collected under
which experiment it belongs to
```

For research data, that is dangerous.

Good app structure helps prevent bad data collection.

---

## 20. What You Learned in Lesson 15

The key data model is:

```text
Patient
 ↓
Session
 ↓
Measurement
 ↓
Result
```

You learned that:

```text
PatientEntity
```

stores the main patient/sample/participant record.

```text
SessionEntity
```

stores one data-collection event and links back to a patient using `patientId`.

```text
MeasurementEntity
```

stores one measurement and links back to a session using `sessionId`.

```text
ResultEntity
```

stores processed or ML output and links back to a session using `sessionId`.

You also learned that:

```text
Room can auto-generate primary keys when autoGenerate = true.
Room does not auto-fill foreign keys.
Readable codes like P001 are different from internal IDs like 1.
```

The most important mental model is:

```text
Do not only save values.
Save values with context.
```

For a research app, the context is often:

```text
who/what was measured
when the session started
when the session ended
which session the measurement belongs to
what result was produced
```

After Lesson 15, our architecture is:

```text
ResearchScreen
 ↓
ResearchViewModel
 ↓
MeasurementRepository
 ↓
PatientDao / SessionDao / MeasurementDao / ResultDao
 ↓
ResearchDatabase
```

And our research data structure is:

```text
Patient
 └── Session
      ├── Measurement
      ├── Measurement
      ├── Measurement
      └── Result
```

## Lesson 16 Preview

In Lesson 16, we should move from one large screen to a multi-screen app.
The app should start to look like this:

```text
Patient List Screen
 ↓
Patient Detail Screen
 ↓
Session / Measurement Screen
 ↓
Result Screen
```

So Lesson 16 should cover:

- basic Compose navigation
- patient list screen
- patient detail screen
- session screen
- passing patientId/sessionId between screens
- why multi-screen structure matters

This will make the app feel less like a single tutorial screen and more like a real tablet research application.
