# Lesson 26 — Building the Repository Layer

In Lesson 25, we built the Room database layer.

We created:

```text
PatientDao
SessionDao
MeasurementDao
ResultDao
ResearchDatabase
```

Now we need a layer that uses those DAOs.

That layer is the:

```text
Repository layer
```

This corresponds to **Step A4** in Direction A:

```text
Build Repository
```

The goal of Lesson 26 is to create a useful `MeasurementRepository` that connects the rest of the app to the database.

---

## 1. Where the repository fits

After Lesson 25, our architecture looks like this:

```text
ResearchViewModel
 ↓
MeasurementRepository
 ↓
ResearchDatabase
 ↓
PatientDao / SessionDao / MeasurementDao / ResultDao
 ↓
PatientEntity / SessionEntity / MeasurementEntity / ResultEntity
```

The ViewModel should not directly call DAOs.

Avoid this:

```text
ResearchViewModel
 ↓
PatientDao
SessionDao
MeasurementDao
ResultDao
```

A better structure is:

```text
ResearchViewModel
 ↓
MeasurementRepository
 ↓
DAOs
```

The repository acts as the bridge.

---

## 2. Why we need the repository

The DAOs are low-level database access objects.

For example, a DAO can do:

```text
insert patient
get sessions for patient
insert measurement
get result for session
```

But the app usually needs higher-level operations.

For example:

```text
create a patient and session
start a session
save a measurement
load all measurements for a session
run inference and save result
build export data
```

These are not just simple database calls.

They are app-level data operations.

So we put them in:

```text
MeasurementRepository
```

The repository hides the lower-level details from the ViewModel.

---

## 3. Create `MeasurementRepository.kt`

In the project structure from Lesson 23, create this file:

```text
app/src/main/java/com/example/researchapp/data/MeasurementRepository.kt
```

The package should be:

```kotlin
package com.example.researchapp.data
```

Start with an empty class:

```kotlin
package com.example.researchapp.data

class MeasurementRepository {
}
```

Now we will gradually fill it.

---

## 4. Repository needs a database

The repository needs access to the Room database.

So the repository should receive a `Context` and create the database.

```kotlin
package com.example.researchapp.data

import android.content.Context
import androidx.room3.Room

class MeasurementRepository(
    context: Context
) {
    private val database = Room.databaseBuilder(
        context.applicationContext,
        ResearchDatabase::class.java,
        "research_database"
    ).build()
}
```

This creates a database called:

```text
research_database
```

Notice this:

```kotlin
context.applicationContext
```

This is better than storing an Activity context.

The repository may live longer than a screen, so using application context is safer.

---

## 5. Get DAO objects from the database

Now the repository needs access to the DAOs.

Add:

```kotlin
private val patientDao = database.patientDao()
private val sessionDao = database.sessionDao()
private val measurementDao = database.measurementDao()
private val resultDao = database.resultDao()
```

So the repository becomes:

```kotlin
package com.example.researchapp.data

import android.content.Context
import androidx.room3.Room

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
}
```

Now the repository can call:

```text
patientDao
sessionDao
measurementDao
resultDao
```

but the ViewModel does not need to know about them.

---

## 6. Add patient functions

First, add functions for patients.

```kotlin
suspend fun createPatient(
    patientCode: String,
    notes: String = ""
): Long {
    return patientDao.insertPatient(
        PatientEntity(
            patientCode = patientCode,
            notes = notes
        )
    )
}

suspend fun getAllPatients(): List<PatientEntity> {
    return patientDao.getAllPatients()
}

suspend fun getPatientById(
    patientId: Long
): PatientEntity? {
    return patientDao.getPatientById(
        patientId = patientId
    )
}
```

You need these imports:

```kotlin
import com.example.researchapp.data.entity.PatientEntity
```

The first function:

```kotlin
createPatient(...)
```

creates a new patient and returns the generated patient ID.

For example:

```text
createPatient("P001")
 ↓
returns patientId = 1
```

That ID can later be used to create a session.

---

## 7. Add session functions

Now add functions for sessions.

```kotlin
suspend fun createSession(
    patientId: Long,
    sessionName: String,
    notes: String = ""
): Long {
    return sessionDao.insertSession(
        SessionEntity(
            patientId = patientId,
            sessionName = sessionName,
            notes = notes
        )
    )
}

suspend fun getSessionsForPatient(
    patientId: Long
): List<SessionEntity> {
    return sessionDao.getSessionsForPatient(
        patientId = patientId
    )
}

suspend fun getSessionById(
    sessionId: Long
): SessionEntity? {
    return sessionDao.getSessionById(
        sessionId = sessionId
    )
}

suspend fun endSession(
    sessionId: Long,
    endedAt: Long = System.currentTimeMillis()
) {
    sessionDao.endSession(
        sessionId = sessionId,
        endedAt = endedAt
    )
}
```

Import:

```kotlin
import com.example.researchapp.data.entity.SessionEntity
```

These functions support the research workflow:

```text
create patient
 ↓
create session
 ↓
collect measurements
 ↓
end session
```

---

## 8. Why repository functions are `suspend`

Most repository functions are:

```kotlin
suspend fun ...
```

because they call Room DAO functions, which are also `suspend`.

For example:

```kotlin
suspend fun getAllPatients(): List<PatientEntity> {
    return patientDao.getAllPatients()
}
```

This means the ViewModel should call repository functions from a coroutine:

```kotlin
viewModelScope.launch {
    val patients = measurementRepository.getAllPatients()
}
```

The important idea is:

```text
Database work should not block the UI.
```

This follows the coroutine idea from earlier lessons.

---

## 9. Add measurement functions

Now add functions for measurements.

```kotlin
suspend fun insertMeasurement(
    measurement: MeasurementEntity
): Long {
    return measurementDao.insertMeasurement(
        measurement = measurement
    )
}

suspend fun getMeasurementsForSession(
    sessionId: Long
): List<MeasurementEntity> {
    return measurementDao.getMeasurementsForSession(
        sessionId = sessionId
    )
}

suspend fun deleteMeasurementsForSession(
    sessionId: Long
) {
    measurementDao.deleteMeasurementsForSession(
        sessionId = sessionId
    )
}
```

Import:

```kotlin
import com.example.researchapp.data.entity.MeasurementEntity
```

These functions let the app:

```text
save one measurement
load measurements for one session
delete measurements for one session
```

The repository does not yet create measurements from a device.

That will be improved in Lesson 27 and Lesson 28.

For now, this layer only saves and loads measurement entities.

---

## 10. Add result functions

Now add functions for results.

```kotlin
suspend fun insertResult(
    result: ResultEntity
): Long {
    return resultDao.insertResult(
        result = result
    )
}

suspend fun getResultsForSession(
    sessionId: Long
): List<ResultEntity> {
    return resultDao.getResultsForSession(
        sessionId = sessionId
    )
}

suspend fun getLatestResultForSession(
    sessionId: Long
): ResultEntity? {
    return resultDao.getLatestResultForSession(
        sessionId = sessionId
    )
}
```

Import:

```kotlin
import com.example.researchapp.data.entity.ResultEntity
```

These functions support:

```text
save prediction result
load result for ResultScreen
load result for export
```

---

## 11. Repository after adding database functions

At this point, `MeasurementRepository.kt` looks like this:

```kotlin
package com.example.researchapp.data

import android.content.Context
import androidx.room3.Room
import com.example.researchapp.data.entity.MeasurementEntity
import com.example.researchapp.data.entity.PatientEntity
import com.example.researchapp.data.entity.ResultEntity
import com.example.researchapp.data.entity.SessionEntity

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
        patientCode: String,
        notes: String = ""
    ): Long {
        return patientDao.insertPatient(
            PatientEntity(
                patientCode = patientCode,
                notes = notes
            )
        )
    }

    suspend fun getAllPatients(): List<PatientEntity> {
        return patientDao.getAllPatients()
    }

    suspend fun getPatientById(
        patientId: Long
    ): PatientEntity? {
        return patientDao.getPatientById(
            patientId = patientId
        )
    }

    suspend fun createSession(
        patientId: Long,
        sessionName: String,
        notes: String = ""
    ): Long {
        return sessionDao.insertSession(
            SessionEntity(
                patientId = patientId,
                sessionName = sessionName,
                notes = notes
            )
        )
    }

    suspend fun getSessionsForPatient(
        patientId: Long
    ): List<SessionEntity> {
        return sessionDao.getSessionsForPatient(
            patientId = patientId
        )
    }

    suspend fun getSessionById(
        sessionId: Long
    ): SessionEntity? {
        return sessionDao.getSessionById(
            sessionId = sessionId
        )
    }

    suspend fun endSession(
        sessionId: Long,
        endedAt: Long = System.currentTimeMillis()
    ) {
        sessionDao.endSession(
            sessionId = sessionId,
            endedAt = endedAt
        )
    }

    suspend fun insertMeasurement(
        measurement: MeasurementEntity
    ): Long {
        return measurementDao.insertMeasurement(
            measurement = measurement
        )
    }

    suspend fun getMeasurementsForSession(
        sessionId: Long
    ): List<MeasurementEntity> {
        return measurementDao.getMeasurementsForSession(
            sessionId = sessionId
        )
    }

    suspend fun deleteMeasurementsForSession(
        sessionId: Long
    ) {
        measurementDao.deleteMeasurementsForSession(
            sessionId = sessionId
        )
    }

    suspend fun insertResult(
        result: ResultEntity
    ): Long {
        return resultDao.insertResult(
            result = result
        )
    }

    suspend fun getResultsForSession(
        sessionId: Long
    ): List<ResultEntity> {
        return resultDao.getResultsForSession(
            sessionId = sessionId
        )
    }

    suspend fun getLatestResultForSession(
        sessionId: Long
    ): ResultEntity? {
        return resultDao.getLatestResultForSession(
            sessionId = sessionId
        )
    }
}
```

This is already useful.

It gives the ViewModel one clear object for database operations.

---

## 12. Add a helper function: create patient and session together

In the app, the user may enter:

```text
patientCode = "P001"
sessionName = "Baseline"
```

Then the app needs to:

```text
insert patient
 ↓
get patientId
 ↓
insert session with patientId
 ↓
get sessionId
```

So the repository can provide a higher-level function:

```kotlin
suspend fun createPatientAndSession(
    patientCode: String,
    sessionName: String,
    patientNotes: String = "",
    sessionNotes: String = ""
): Pair<Long, Long> {
    val patientId = createPatient(
        patientCode = patientCode,
        notes = patientNotes
    )

    val sessionId = createSession(
        patientId = patientId,
        sessionName = sessionName,
        notes = sessionNotes
    )

    return Pair(
        patientId,
        sessionId
    )
}
```

This returns:

```kotlin
Pair<Long, Long>
```

where:

```text
first = patientId
second = sessionId
```

Example:

```text
createPatientAndSession("P001", "Baseline")
 ↓
returns Pair(1, 10)
```

This is a good example of why repositories are useful.

The ViewModel does not need to manually coordinate multiple DAO calls.

---

## 13. Should the repository be called `MeasurementRepository`?

At this point, the repository does more than measurements.

It handles:

```text
patients
sessions
measurements
results
```

So you may wonder:

```text
Should it still be called MeasurementRepository?
```

For a beginner project, it is okay.

But a more precise name might be:

```text
ResearchRepository
```

or:

```text
SessionRepository
```

For this tutorial, we can continue using:

```text
MeasurementRepository
```

because that is the name we introduced earlier.

But if starting a real project from scratch, I would probably use:

```text
ResearchRepository
```

because it handles the whole research data workflow.

So you may choose either:

```text
MeasurementRepository
```

or:

```text
ResearchRepository
```

The important thing is consistency.

---

## 14. Should there be multiple repositories?

In a larger app, you might split repositories:

```text
PatientRepository
SessionRepository
MeasurementRepository
ResultRepository
DeviceRepository
```

That can be useful later.

But for this Direction A skeleton, one repository is simpler.

Use one repository first:

```text
MeasurementRepository
```

Later, if the file becomes too large, split it.

The beginner rule is:

```text
Do not split too early.
Do not put everything in ViewModel either.
```

So one repository is a good middle point for now.

---

## 15. Add a simple manual test function idea

Before connecting the UI, it is useful to understand how this repository would be used.

Conceptually:

```kotlin
val patientId = repository.createPatient(
    patientCode = "P001"
)

val sessionId = repository.createSession(
    patientId = patientId,
    sessionName = "Baseline"
)

repository.insertMeasurement(
    MeasurementEntity(
        sessionId = sessionId,
        repetition = 1,
        rawValue = 2.438,
        processedValue = 2.238,
        status = "OK"
    )
)

val measurements = repository.getMeasurementsForSession(
    sessionId = sessionId
)
```

This creates:

```text
Patient P001
 ↓
Session Baseline
 ↓
Measurement 1
```

This is the core data workflow.

---

## 16. Repository should not contain UI code

The repository should not do things like:

```kotlin
Text("Patient saved")
```

or:

```kotlin
Button(...)
```

That belongs to the UI.

The repository should return data or perform data operations.

Good repository functions:

```text
createPatient()
createSession()
getMeasurementsForSession()
insertResult()
```

Bad repository responsibilities:

```text
show snackbar
navigate to another screen
enable or disable a button
draw measurement chart
```

Those belong to UI/ViewModel.

---

## 17. Repository should not directly decide navigation

Avoid this idea:

```text
Repository creates session
 ↓
Repository navigates to MeasurementScreen
```

That is not the repository’s job.

A better flow is:

```text
ViewModel asks repository to create session
 ↓
repository returns sessionId
 ↓
ViewModel updates UI state
 ↓
UI/navigation layer moves to MeasurementScreen
```

So the repository provides data.

The ViewModel and UI decide what to do with it.

---

## 18. How the ViewModel will use this repository later

Later, the ViewModel may do something like:

```kotlin
fun createPatientAndSession() {
    viewModelScope.launch {
        val result = measurementRepository.createPatientAndSession(
            patientCode = uiState.patientCode,
            sessionName = uiState.sessionName
        )

        uiState = uiState.copy(
            currentPatientId = result.first,
            currentSessionId = result.second,
            message = "Session created"
        )
    }
}
```

This function is not the focus of Lesson 26 yet.

But this shows the relationship:

```text
ViewModel
 ↓
calls repository
 ↓
updates UI state
```

The repository does not update Compose state directly.

---

## 19. Add basic error handling in ViewModel, not repository

The repository can throw errors if something fails.

The ViewModel should catch them and show messages.

For example:

```kotlin
viewModelScope.launch {
    try {
        val patientId = measurementRepository.createPatient(
            patientCode = uiState.patientCode
        )

        uiState = uiState.copy(
            currentPatientId = patientId,
            message = "Patient created"
        )
    } catch (e: Exception) {
        uiState = uiState.copy(
            message = "Could not create patient"
        )
    }
}
```

This keeps the repository clean.

The repository performs the operation.

The ViewModel decides what message the user sees.

---

## 20. Repository and future fake device source

In Lesson 27, we will connect the repository to:

```text
DeviceDataSource
FakeDeviceDataSource
```

Then the repository will be able to do:

```text
connect fake device
read fake value
create MeasurementEntity
save measurement
```

The future flow will be:

```text
ViewModel
 ↓
MeasurementRepository
 ↓
FakeDeviceDataSource
 ↓
MeasurementEntity
 ↓
Room
```

But in Lesson 26, we first made the repository work with Room.

That is the correct order.

---

## 21. Repository and future processing

In Lesson 28, we will connect:

```text
SignalProcessor
```

Then the repository will be able to do:

```text
rawValue
 ↓
baseline correction
 ↓
processedValue
 ↓
MeasurementEntity
```

So the future function may look like:

```kotlin
suspend fun createMeasurementFromDevice(
    sessionId: Long,
    repetition: Int
): MeasurementEntity {
    val rawValue = deviceDataSource.readValue()
    val processedValue = signalProcessor.baselineCorrect(
        rawValue = rawValue,
        baseline = 0.2
    )

    return MeasurementEntity(
        sessionId = sessionId,
        repetition = repetition,
        rawValue = rawValue,
        processedValue = processedValue,
        status = "OK"
    )
}
```

But we will add this later.

For now, the repository is the database bridge.

---

## 22. Repository and future ML

In Lesson 29, we will connect:

```text
ModelRunner
FakeModelRunner
```

Then the repository will be able to do:

```text
load measurements for session
 ↓
extract features
 ↓
run fake model
 ↓
save result
```

That function may look like:

```kotlin
suspend fun runAndSaveInferenceForSession(
    sessionId: Long
): ResultEntity? {
    ...
}
```

But again, not yet.

We build gradually.

---

## 23. Repository and future export

In Lesson 32, we will connect export logic.

Then the repository may be able to do:

```text
load patient
load session
load measurements
load latest result
build CSV text
```

A future function may be:

```kotlin
suspend fun buildSessionCsvExport(
    sessionId: Long
): String {
    ...
}
```

But for Lesson 26, we only prepare the repository base.

---

## 24. Current files after Lesson 26

After Lesson 26, the important file is:

```text
data/MeasurementRepository.kt
```

Your data folder should now look like:

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

At this stage, the app has:

```text
entities
DAOs
database
repository
```

That means the data layer is now taking shape.

---

## 25. What you learned in Lesson 26

You learned that the repository is the bridge between:

```text
ViewModel
and:
Room database / DAOs
```

You created functions such as:

```text
createPatient()
getAllPatients()
getPatientById()
createSession()
getSessionsForPatient()
getSessionById()
endSession()
insertMeasurement()
getMeasurementsForSession()
insertResult()
getLatestResultForSession()
```

The most important mental model is:

```text
DAO = low-level database operation

Repository = app-level data operation
```

So instead of this:

```text
ViewModel directly talks to many DAOs
```

we use this:

```text
ViewModel
 ↓
Repository
 ↓
DAOs
```

This keeps the ViewModel cleaner and prepares the app for future device, processing, ML, and export logic.

---

## 26. Lesson 27 preview

In Lesson 27, we will add the fake device source into the real project.

We will implement:

```text
DeviceDataSource
FakeDeviceDataSource
```

and then update the repository so it can:

```text
connect fake device
disconnect fake device
read fake values
create measurement entities from fake device data
```

This will start turning the app from a database skeleton into a working fake-data acquisition app.