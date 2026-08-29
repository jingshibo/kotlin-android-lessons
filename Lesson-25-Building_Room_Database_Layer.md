# Lesson 25 — Building the Room Database Layer

In Lesson 24, we created the core entity files:

```text
PatientEntity.kt
SessionEntity.kt
MeasurementEntity.kt
ResultEntity.kt
```

These files define the data structure of the app:

```text
Patient
 ↓
Session
 ↓
Measurement
 ↓
Result
```

Now we build the database layer around those entities.

This corresponds to **Step A3** in Direction A:

```text
Build Room database
```

The goal of Lesson 25 is to create:

```text
PatientDao.kt
SessionDao.kt
MeasurementDao.kt
ResultDao.kt
ResearchDatabase.kt
```

After this lesson, the app will have the files needed to insert, read, update, and query research data locally.

Room has three major parts: a database class, data entities, and data access objects, or DAOs. The database class is the main access point, entities represent database tables, and DAOs provide functions for querying, inserting, updating, and deleting data. citeturn894376view0

---

## 1. Where we are in the project structure

In Lesson 23, we planned this folder structure:

```text
data
 ├── ResearchDatabase.kt
 ├── MeasurementRepository.kt
 ├── entity
 │    ├── PatientEntity.kt
 │    ├── SessionEntity.kt
 │    ├── MeasurementEntity.kt
 │    └── ResultEntity.kt
 │
 └── dao
      ├── PatientDao.kt
      ├── SessionDao.kt
      ├── MeasurementDao.kt
      └── ResultDao.kt
```

In Lesson 24, we implemented:

```text
data/entity
```

Now we implement:

```text
data/dao
ResearchDatabase.kt
```

So the focus is:

```text
DAO files
 ↓
database class
```

---

## 2. What is a DAO?

DAO means:

```text
Data Access Object
```

A DAO defines how the app interacts with a database table.

For example:

```text
PatientDao
 ↓
insert patient
read all patients
read patient by ID

SessionDao
 ↓
insert session
read sessions for one patient
end session

MeasurementDao
 ↓
insert measurement
read measurements for one session

ResultDao
 ↓
insert result
read result for one session
```

The important idea is:

```text
The rest of the app should not write SQL everywhere.
```

Instead, SQL is collected inside DAO files.

Android’s Room documentation says DAOs provide abstract access to the database, and Room generates DAO implementations at compile time. It also notes that using DAOs helps preserve separation of concerns. citeturn894376view1

---

## 3. Create the `dao` package

Create this folder:

```text
app/src/main/java/com/example/researchapp/data/dao
```

Inside it, create:

```text
PatientDao.kt
SessionDao.kt
MeasurementDao.kt
ResultDao.kt
```

Each file should start with:

```kotlin
package com.example.researchapp.data.dao
```

---

## 4. `PatientDao.kt`

Create:

```text
PatientDao.kt
```

Code:

```kotlin
package com.example.researchapp.data.dao

import androidx.room3.Dao
import androidx.room3.Insert
import androidx.room3.Query
import com.example.researchapp.data.entity.PatientEntity

@Dao
interface PatientDao {

    @Insert
    suspend fun insertPatient(
        patient: PatientEntity
    ): Long

    @Query("SELECT * FROM patients ORDER BY createdAt DESC")
    suspend fun getAllPatients(): List<PatientEntity>

    @Query("SELECT * FROM patients WHERE id = :patientId LIMIT 1")
    suspend fun getPatientById(
        patientId: Long
    ): PatientEntity?
}
```

This DAO gives us three operations:

```text
insert one patient
read all patients
read one patient by ID
```

---

## 5. Understanding `insertPatient()`

```kotlin
@Insert
suspend fun insertPatient(
    patient: PatientEntity
): Long
```

This inserts one patient into the `patients` table.

It returns:

```kotlin
Long
```

That is the generated database row ID.

For example:

```text
insert PatientEntity(patientCode = "P001")
 ↓
Room returns 1
```

This returned ID is important because later we need it to create a session:

```text
patientId = 1
```

Room’s DAO documentation explains that when an `@Insert` function receives a single parameter, it can return a `Long`, the new row ID for the inserted item. citeturn894376view1

---

## 6. Understanding `getAllPatients()`

```kotlin
@Query("SELECT * FROM patients ORDER BY createdAt DESC")
suspend fun getAllPatients(): List<PatientEntity>
```

This reads all patient rows.

The SQL means:

```sql
SELECT * FROM patients ORDER BY createdAt DESC
```

In simple language:

```text
Get all patients.
Show newest records first.
```

The return type is:

```kotlin
List<PatientEntity>
```

because there may be many patients.

---

## 7. Understanding `getPatientById()`

```kotlin
@Query("SELECT * FROM patients WHERE id = :patientId LIMIT 1")
suspend fun getPatientById(
    patientId: Long
): PatientEntity?
```

This reads one patient by database ID.

The SQL means:

```sql
SELECT * FROM patients WHERE id = :patientId LIMIT 1
```

In simple language:

```text
Find the patient whose id matches patientId.
Return at most one row.
```

The return type is:

```kotlin
PatientEntity?
```

not:

```kotlin
PatientEntity
```

because the patient may not exist.

This connects to Kotlin null safety:

```text
PatientEntity?
 ↓
maybe a patient exists,
maybe not
```

---

## 8. `SessionDao.kt`

Create:

```text
SessionDao.kt
```

Code:

```kotlin
package com.example.researchapp.data.dao

import androidx.room3.Dao
import androidx.room3.Insert
import androidx.room3.Query
import com.example.researchapp.data.entity.SessionEntity

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

    @Query("SELECT * FROM sessions WHERE id = :sessionId LIMIT 1")
    suspend fun getSessionById(
        sessionId: Long
    ): SessionEntity?

    @Query("UPDATE sessions SET endedAt = :endedAt WHERE id = :sessionId")
    suspend fun endSession(
        sessionId: Long,
        endedAt: Long
    )
}
```

This DAO lets us:

```text
create a session
get sessions for one patient
get one session by ID
mark a session as ended
```

---

## 9. Understanding `insertSession()`

```kotlin
@Insert
suspend fun insertSession(
    session: SessionEntity
): Long
```

This inserts one session.

Example:

```kotlin
val sessionId = sessionDao.insertSession(
    SessionEntity(
        patientId = 1,
        sessionName = "Baseline"
    )
)
```

This creates a session linked to Patient 1.

The returned `sessionId` is then used when inserting measurements.

---

## 10. Understanding `getSessionsForPatient()`

```kotlin
@Query("SELECT * FROM sessions WHERE patientId = :patientId ORDER BY startedAt DESC")
suspend fun getSessionsForPatient(
    patientId: Long
): List<SessionEntity>
```

This reads all sessions belonging to one patient.

The SQL means:

```sql
SELECT * FROM sessions
WHERE patientId = :patientId
ORDER BY startedAt DESC
```

In simple language:

```text
Find all sessions for this patient.
Show newest sessions first.
```

This is useful for the Patient Detail screen.

For example:

```text
Patient P001
 ├── Baseline
 ├── Follow-up 1
 └── Follow-up 2
```

---

## 11. Understanding `getSessionById()`

```kotlin
@Query("SELECT * FROM sessions WHERE id = :sessionId LIMIT 1")
suspend fun getSessionById(
    sessionId: Long
): SessionEntity?
```

This reads one session.

It is useful for:

```text
MeasurementScreen
ResultScreen
export
```

For example, if the screen receives:

```text
sessionId = 10
```

the app can load the session metadata from Room.

---

## 12. Understanding `endSession()`

```kotlin
@Query("UPDATE sessions SET endedAt = :endedAt WHERE id = :sessionId")
suspend fun endSession(
    sessionId: Long,
    endedAt: Long
)
```

This updates a session when acquisition stops.

The SQL means:

```sql
UPDATE sessions
SET endedAt = :endedAt
WHERE id = :sessionId
```

In simple language:

```text
Find this session.
Set its endedAt time.
```

This matches our app flow:

```text
Create session
 ↓
Start acquisition
 ↓
Stop acquisition
 ↓
Set endedAt
```

---

## 13. `MeasurementDao.kt`

Create:

```text
MeasurementDao.kt
```

Code:

```kotlin
package com.example.researchapp.data.dao

import androidx.room3.Dao
import androidx.room3.Insert
import androidx.room3.Query
import com.example.researchapp.data.entity.MeasurementEntity

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

This DAO lets us:

```text
insert one measurement
read measurements for one session
delete measurements for one session
```

---

## 14. Understanding `insertMeasurement()`

```kotlin
@Insert
suspend fun insertMeasurement(
    measurement: MeasurementEntity
): Long
```

This inserts one measurement row.

Example:

```kotlin
measurementDao.insertMeasurement(
    MeasurementEntity(
        sessionId = 10,
        repetition = 1,
        rawValue = 2.438,
        processedValue = 2.238,
        status = "OK"
    )
)
```

This creates a measurement belonging to Session 10.

The important relationship is:

```text
MeasurementEntity.sessionId
 ↓
SessionEntity.id
```

---

## 15. Understanding `getMeasurementsForSession()`

```kotlin
@Query("SELECT * FROM measurements WHERE sessionId = :sessionId ORDER BY timestamp ASC")
suspend fun getMeasurementsForSession(
    sessionId: Long
): List<MeasurementEntity>
```

This reads measurements for one session.

The SQL means:

```sql
SELECT * FROM measurements
WHERE sessionId = :sessionId
ORDER BY timestamp ASC
```

In simple language:

```text
Find all measurements for this session.
Show oldest measurements first.
```

This is useful for:

```text
showing measurement history
building CSV export
extracting features for ML inference
```

---

## 16. Understanding `deleteMeasurementsForSession()`

```kotlin
@Query("DELETE FROM measurements WHERE sessionId = :sessionId")
suspend fun deleteMeasurementsForSession(
    sessionId: Long
)
```

This deletes measurements for one session only.

This is safer than:

```sql
DELETE FROM measurements
```

because we do not want to accidentally delete the whole app’s data.

For a research app, destructive operations should usually be narrow and controlled.

---

## 17. `ResultDao.kt`

Create:

```text
ResultDao.kt
```

Code:

```kotlin
package com.example.researchapp.data.dao

import androidx.room3.Dao
import androidx.room3.Insert
import androidx.room3.Query
import com.example.researchapp.data.entity.ResultEntity

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

    @Query("SELECT * FROM results WHERE sessionId = :sessionId ORDER BY createdAt DESC LIMIT 1")
    suspend fun getLatestResultForSession(
        sessionId: Long
    ): ResultEntity?
}
```

This DAO lets us:

```text
insert result
read all results for a session
read the latest result for a session
```

---

## 18. Understanding `insertResult()`

```kotlin
@Insert
suspend fun insertResult(
    result: ResultEntity
): Long
```

This inserts one model or analysis result.

Example:

```kotlin
resultDao.insertResult(
    ResultEntity(
        sessionId = 10,
        label = "Positive",
        confidence = 0.92
    )
)
```

This means:

```text
Session 10 produced a Positive result with confidence 0.92.
```

---

## 19. Understanding `getResultsForSession()`

```kotlin
@Query("SELECT * FROM results WHERE sessionId = :sessionId ORDER BY createdAt DESC")
suspend fun getResultsForSession(
    sessionId: Long
): List<ResultEntity>
```

This reads all results for one session.

Why might there be more than one result?

Because the user may run inference multiple times.

For example:

```text
Result 1: before enough data
Result 2: after 10 measurements
Result 3: after changing model threshold
```

For now, multiple results are allowed.

---

## 20. Understanding `getLatestResultForSession()`

```kotlin
@Query("SELECT * FROM results WHERE sessionId = :sessionId ORDER BY createdAt DESC LIMIT 1")
suspend fun getLatestResultForSession(
    sessionId: Long
): ResultEntity?
```

This returns the newest result for one session.

It is useful for:

```text
ResultScreen
CSV export
patient/session summary
```

The return type is nullable:

```kotlin
ResultEntity?
```

because a session may not have a result yet.

---

## 21. Why DAO functions are `suspend`

Most DAO functions are written as:

```kotlin
suspend fun ...
```

because database work should not block the UI.

This connects back to our earlier coroutine lessons.

The app should avoid doing heavy database, file, device, or ML work directly on the UI thread.

The DAO functions can be called from:

```kotlin
viewModelScope.launch {
    ...
}
```

or from suspend repository functions.

Room 3.0 is coroutine-based, Kotlin-only for code generation, and requires KSP. citeturn894376view2

---

## 22. Create `ResearchDatabase.kt`

Now create this file:

```text
app/src/main/java/com/example/researchapp/data/ResearchDatabase.kt
```

Code:

```kotlin
package com.example.researchapp.data

import androidx.room3.Database
import androidx.room3.RoomDatabase
import com.example.researchapp.data.dao.MeasurementDao
import com.example.researchapp.data.dao.PatientDao
import com.example.researchapp.data.dao.ResultDao
import com.example.researchapp.data.dao.SessionDao
import com.example.researchapp.data.entity.MeasurementEntity
import com.example.researchapp.data.entity.PatientEntity
import com.example.researchapp.data.entity.ResultEntity
import com.example.researchapp.data.entity.SessionEntity

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

This file connects all entities and DAOs together.

The Room database class must be annotated with `@Database`, list its entities, extend `RoomDatabase`, and define abstract zero-argument functions that return each DAO associated with the database. citeturn894376view0

---

## 23. Understanding `@Database`

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
```

This tells Room:

```text
This database contains four tables:
patients
sessions
measurements
results
```

The line:

```kotlin
version = 1
```

means this is database schema version 1.

Later, if we change entity fields, we will need to increase this version and handle migration.

But for the beginner Direction A skeleton, version 1 is fine.

---

## 24. Understanding DAO access functions

Inside `ResearchDatabase`, we wrote:

```kotlin
abstract fun patientDao(): PatientDao

abstract fun sessionDao(): SessionDao

abstract fun measurementDao(): MeasurementDao

abstract fun resultDao(): ResultDao
```

These functions give the repository access to each DAO.

The future repository will use them like:

```kotlin
val patientDao = database.patientDao()
val sessionDao = database.sessionDao()
val measurementDao = database.measurementDao()
val resultDao = database.resultDao()
```

Then the repository can do things like:

```kotlin
patientDao.insertPatient(...)
sessionDao.insertSession(...)
measurementDao.insertMeasurement(...)
resultDao.insertResult(...)
```

So the database class is the central Room object.

---

## 25. Room package note: `androidx.room3` vs `androidx.room`

In Lesson 24 and Lesson 25, I used imports like:

```kotlin
import androidx.room3.Dao
import androidx.room3.Entity
import androidx.room3.Database
```

That follows Room 3.

Older tutorials may show:

```kotlin
import androidx.room.Dao
import androidx.room.Entity
import androidx.room.Database
```

That is Room 2.x style.

Room 3.0 uses the new `androidx.room3` package and new artifact names such as `androidx.room3:room3-runtime`; Android’s Room 3.0 release notes say the package changed from `androidx.room` to `androidx.room3`, with KSP required. citeturn894376view2

So if your Android Studio project is using Room 2.x, your imports will be different.

For this Direction A tutorial, we will continue using:

```text
Room 3 style
```

unless your actual project is set up differently.

---

## 26. Gradle dependency idea

In a real project, Room also needs Gradle dependencies.

The exact version may change, but conceptually you need:

```kotlin
implementation("androidx.room3:room3-runtime:<room_version>")
ksp("androidx.room3:room3-compiler:<room_version>")
```

You also need the KSP plugin configured in your Gradle files.

Do not worry if this is not fully clear yet.

For Lesson 25, the main focus is the Kotlin database-layer code.

Gradle setup can be handled when we actually compile the project.

---

## 27. Current files after Lesson 25

After this lesson, your `data` folder should contain:

```text
data
 ├── ResearchDatabase.kt
 ├── MeasurementRepository.kt
 │
 ├── entity
 │    ├── PatientEntity.kt
 │    ├── SessionEntity.kt
 │    ├── MeasurementEntity.kt
 │    └── ResultEntity.kt
 │
 └── dao
      ├── PatientDao.kt
      ├── SessionDao.kt
      ├── MeasurementDao.kt
      └── ResultDao.kt
```

The database layer is now defined.

The app still does not use the database yet.

That will happen through the repository in the next lesson.

---

## 28. What you learned in Lesson 25

You created four DAO files:

```text
PatientDao
SessionDao
MeasurementDao
ResultDao
```

You learned that DAOs define database operations such as:

```text
insert
query
update
delete
```

You created:

```text
ResearchDatabase
```

which connects:

```text
PatientEntity
SessionEntity
MeasurementEntity
ResultEntity
```

with:

```text
PatientDao
SessionDao
MeasurementDao
ResultDao
```

The most important mental model is:

```text
Entity = what data is stored

DAO = how data is accessed

Database = main Room object that connects entities and DAOs
```

The app architecture is now:

```text
ViewModel
 ↓
Repository
 ↓
ResearchDatabase
 ↓
DAOs
 ↓
Entities / tables
```

But at this stage, the repository is still mostly empty.

---

## Lesson 26 preview
In Lesson 26, we will build the Repository layer.
We will create real repository functions such as:
- createPatient()
- createSession()
- endSession()
- insertMeasurement()
- getMeasurementsForSession()
- saveResult()
- buildSessionCsvExport()

This will connect the database layer to the rest of the app.