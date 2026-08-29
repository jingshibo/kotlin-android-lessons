# Lesson 27 — Adding the Fake Device Source

In Lesson 26, we built the Repository layer.

The repository can now talk to Room through the DAOs:

```text
MeasurementRepository
 ↓
ResearchDatabase
 ↓
PatientDao / SessionDao / MeasurementDao / ResultDao
```

It can already do database operations such as:

```text
createPatient()
createSession()
insertMeasurement()
getMeasurementsForSession()
insertResult()
```

Now we add the next part:

```text
fake device data source
```

This corresponds to **Step A5** in Direction A:

```text
Add fake device source
```

The goal is to make the app capable of pretending to connect to a device and read measurement values.

We are still not using real Bluetooth or Wi-Fi yet.

---

## 1. Why use a fake device first?

A real device connection can involve:

```text
Bluetooth permissions
device scanning
connection errors
data streams
message parsing
Android version differences
hardware debugging
```

That is too much to handle before the app skeleton works.

So we first create:

```text
FakeDeviceDataSource
```

This lets us test the app flow:

```text
connect device
 ↓
read value
 ↓
create measurement
 ↓
save to Room
 ↓
show in UI later
```

without needing real hardware.

This follows the same tutorial strategy we have used many times:

```text
fake first
real later
```

---

## 2. Where the fake device code belongs

In Lesson 23, we planned this folder:

```text
device
 ├── DeviceDataSource.kt
 └── FakeDeviceDataSource.kt
```

So now we create real code inside:

```text
app/src/main/java/com/example/researchapp/device
```

The `device` package is for:

```text
fake device communication now
real Bluetooth/Wi-Fi/USB communication later
```

Do not put fake device code inside:

```text
MeasurementScreen.kt
```

or:

```text
ResearchViewModel.kt
```

The device source should be separate.

---

## 3. Create `DeviceDataSource.kt`

Create this file:

```text
device/DeviceDataSource.kt
```

Code:

```kotlin
package com.example.researchapp.device

interface DeviceDataSource {

    suspend fun connect()

    suspend fun disconnect()

    suspend fun readValue(): Double
}
```

This interface says:

```text
Any device data source must be able to:
connect
disconnect
read one value
```

The app will use this interface instead of directly depending on a fake or real device.

---

## 4. Why use an interface?

An interface gives us a stable doorway.

For now:

```text
DeviceDataSource
 ↓
FakeDeviceDataSource
```

Later:

```text
DeviceDataSource
 ↓
BluetoothDeviceDataSource
```

or:

```text
DeviceDataSource
 ↓
WifiDeviceDataSource
```

The rest of the app can keep calling:

```kotlin
deviceDataSource.connect()
deviceDataSource.readValue()
deviceDataSource.disconnect()
```

without caring whether the device is fake or real.

This protects the app from needing big changes later.

---

## 5. Create `FakeDeviceDataSource.kt`

Create this file:

```text
device/FakeDeviceDataSource.kt
```

Code:

```kotlin
package com.example.researchapp.device

import kotlinx.coroutines.delay
import kotlin.random.Random

class FakeDeviceDataSource : DeviceDataSource {

    private var connected: Boolean = false

    override suspend fun connect() {
        delay(1000)
        connected = true
    }

    override suspend fun disconnect() {
        connected = false
    }

    override suspend fun readValue(): Double {
        if (!connected) {
            throw IllegalStateException(
                "Device is not connected"
            )
        }

        delay(1000)

        return Random.nextDouble(
            0.0,
            5.0
        )
    }
}
```

This fake source behaves like a very simple device.

It can:

```text
connect after 1 second
disconnect immediately
return a random value between 0.0 and 5.0
```

---

## 6. Understanding `connected`

Inside the fake device, we have:

```kotlin
private var connected: Boolean = false
```

This stores whether the fake device is connected.

At the beginning:

```text
connected = false
```

After calling:

```kotlin
connect()
```

we set:

```text
connected = true
```

After calling:

```kotlin
disconnect()
```

we set:

```text
connected = false
```

This simulates real device state.

---

## 7. Why `connect()` uses `delay`

The fake connect function is:

```kotlin
override suspend fun connect() {
    delay(1000)
    connected = true
}
```

This means:

```text
wait 1 second
then mark the fake device as connected
```

Why not connect instantly?

Because real device connection usually takes time.

For example, a real connection may need to:

```text
find the device
open a socket
start a stream
check handshake
prepare input buffer
```

So the fake version should also feel asynchronous.

This helps us test loading and connection states later.

---

## 8. Why `readValue()` checks connection

The fake read function is:

```kotlin
override suspend fun readValue(): Double {
    if (!connected) {
        throw IllegalStateException(
            "Device is not connected"
        )
    }

    delay(1000)

    return Random.nextDouble(
        0.0,
        5.0
    )
}
```

The first part is important:

```kotlin
if (!connected) {
    throw IllegalStateException(
        "Device is not connected"
    )
}
```

This means:

```text
Do not allow reading if the device is not connected.
```

This is a good habit.

A real app should not silently pretend data exists when the device is disconnected.

---

## 9. Update `MeasurementRepository`

Now we connect the fake device source to the repository.

In Lesson 26, the repository started like this:

```kotlin
class MeasurementRepository(
    context: Context
) {
    ...
}
```

Now we add a `DeviceDataSource`:

```kotlin
class MeasurementRepository(
    context: Context,
    private val deviceDataSource: DeviceDataSource = FakeDeviceDataSource()
) {
    ...
}
```

This means:

```text
By default, use FakeDeviceDataSource.
```

Later, we can replace it with:

```text
BluetoothDeviceDataSource
```

without changing the ViewModel or UI too much.

---

## 10. Add imports to repository

At the top of `MeasurementRepository.kt`, add:

```kotlin
import com.example.researchapp.device.DeviceDataSource
import com.example.researchapp.device.FakeDeviceDataSource
```

So the top of the repository file becomes:

```kotlin
package com.example.researchapp.data

import android.content.Context
import androidx.room3.Room
import com.example.researchapp.data.entity.MeasurementEntity
import com.example.researchapp.data.entity.PatientEntity
import com.example.researchapp.data.entity.ResultEntity
import com.example.researchapp.data.entity.SessionEntity
import com.example.researchapp.device.DeviceDataSource
import com.example.researchapp.device.FakeDeviceDataSource
```

Now the repository knows about the device interface and fake implementation.

---

## 11. Add device connection functions to repository

Inside `MeasurementRepository`, add:

```kotlin
suspend fun connectDevice() {
    deviceDataSource.connect()
}

suspend fun disconnectDevice() {
    deviceDataSource.disconnect()
}
```

These functions allow the ViewModel to call:

```kotlin
measurementRepository.connectDevice()
```

instead of directly calling:

```kotlin
deviceDataSource.connect()
```

The flow remains:

```text
ViewModel
 ↓
Repository
 ↓
DeviceDataSource
```

This keeps the architecture clean.

---

## 12. Create measurement from fake device value

Now we add a function that reads a value from the fake device and turns it into a `MeasurementEntity`.

```kotlin
suspend fun createMeasurementFromDevice(
    sessionId: Long,
    repetition: Int
): MeasurementEntity {
    val rawValue = deviceDataSource.readValue()

    return MeasurementEntity(
        sessionId = sessionId,
        repetition = repetition,
        rawValue = rawValue,
        processedValue = rawValue,
        status = "OK"
    )
}
```

For now, we set:

```kotlin
processedValue = rawValue
```

Why?

Because we have not connected `SignalProcessor` yet.

That will happen in Lesson 28.

For Lesson 27, the goal is only:

```text
read fake value
create MeasurementEntity
```

Processing comes next.

---

## 13. Add function to read and save measurement

The repository can also provide a convenience function:

```kotlin
suspend fun readAndSaveMeasurement(
    sessionId: Long,
    repetition: Int
): MeasurementEntity {
    val measurement = createMeasurementFromDevice(
        sessionId = sessionId,
        repetition = repetition
    )

    insertMeasurement(
        measurement = measurement
    )

    return measurement
}
```

This function does three things:

```text
read value from device
create MeasurementEntity
save it to Room
```

This is useful because the ViewModel can later call one function:

```kotlin
val measurement = measurementRepository.readAndSaveMeasurement(
    sessionId = sessionId,
    repetition = repetition
)
```

instead of manually doing each step.

---

## 14. Updated repository with device functions

Now the repository has database functions and device functions.

A simplified version looks like this:

```kotlin
package com.example.researchapp.data

import android.content.Context
import androidx.room3.Room
import com.example.researchapp.data.entity.MeasurementEntity
import com.example.researchapp.data.entity.PatientEntity
import com.example.researchapp.data.entity.ResultEntity
import com.example.researchapp.data.entity.SessionEntity
import com.example.researchapp.device.DeviceDataSource
import com.example.researchapp.device.FakeDeviceDataSource

class MeasurementRepository(
    context: Context,
    private val deviceDataSource: DeviceDataSource = FakeDeviceDataSource()
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

    suspend fun connectDevice() {
        deviceDataSource.connect()
    }

    suspend fun disconnectDevice() {
        deviceDataSource.disconnect()
    }

    suspend fun createMeasurementFromDevice(
        sessionId: Long,
        repetition: Int
    ): MeasurementEntity {
        val rawValue = deviceDataSource.readValue()

        return MeasurementEntity(
            sessionId = sessionId,
            repetition = repetition,
            rawValue = rawValue,
            processedValue = rawValue,
            status = "OK"
        )
    }

    suspend fun readAndSaveMeasurement(
        sessionId: Long,
        repetition: Int
    ): MeasurementEntity {
        val measurement = createMeasurementFromDevice(
            sessionId = sessionId,
            repetition = repetition
        )

        insertMeasurement(
            measurement = measurement
        )

        return measurement
    }

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
}
```

This is becoming a real app data layer.

---

## 15. What changed from Lesson 26?

In Lesson 26, the repository only worked with Room.

```text
Repository
 ↓
Room database
```

After Lesson 27, the repository also works with a device source.

```text
Repository
 ├── Room database
 └── DeviceDataSource
```

So the repository can now:

```text
connect to fake device
read fake value
create measurement
save measurement
```

This is an important step.

The app is no longer only a database skeleton.

It is becoming a fake acquisition app.

---

## 16. What should the ViewModel call later?

Later, the ViewModel can call:

```kotlin
measurementRepository.connectDevice()
```

to connect the fake device.

It can call:

```kotlin
measurementRepository.disconnectDevice()
```

to disconnect.

It can call:

```kotlin
measurementRepository.readAndSaveMeasurement(
    sessionId = sessionId,
    repetition = repetition
)
```

to read and save one measurement.

This means the ViewModel does not need to know:

```text
where the value came from
how it was generated
how it was saved
```

It only calls repository functions.

---

## 17. Example usage flow

Conceptually, the app can now do:

```kotlin
val ids = repository.createPatientAndSession(
    patientCode = "P001",
    sessionName = "Baseline"
)

val patientId = ids.first
val sessionId = ids.second

repository.connectDevice()

repository.readAndSaveMeasurement(
    sessionId = sessionId,
    repetition = 1
)

repository.disconnectDevice()
```

This creates:

```text
Patient P001
 ↓
Session Baseline
 ↓
Fake measurement 1
```

This is almost the core fake-data workflow.

The UI is not connected yet, but the data layer is ready.

---

## 18. Why `readAndSaveMeasurement()` returns the measurement

The function returns:

```kotlin
MeasurementEntity
```

because the ViewModel may want to update the UI immediately.

For example, later:

```kotlin
val measurement =
    measurementRepository.readAndSaveMeasurement(
        sessionId = sessionId,
        repetition = repetition
    )

uiState = uiState.copy(
    latestRawValue = measurement.rawValue,
    latestProcessedValue = measurement.processedValue,
    latestStatus = measurement.status
)
```

So the repository saves the measurement and also returns it for display.

This is convenient.

---

## 19. Why `processedValue` is not really processed yet

In Lesson 27, we set:

```kotlin
processedValue = rawValue
```

This is temporary.

The real processing will come in Lesson 28.

For now:

```text
rawValue = fake device value
processedValue = same fake device value
```

After Lesson 28:

```text
rawValue = fake device value
processedValue = baseline-corrected or processed value
```

This step-by-step approach keeps the app understandable.

---

## 20. Should the fake device source save data?

No.

The fake device source should only provide data.

It should not save to Room.

Avoid this structure:

```text
FakeDeviceDataSource
 ↓
writes directly to Room
```

That would mix responsibilities.

Correct structure:

```text
FakeDeviceDataSource
 ↓
provides raw value

MeasurementRepository
 ↓
creates MeasurementEntity
saves to Room
```

The device source should not know about:

```text
PatientEntity
SessionEntity
MeasurementEntity
Room database
```

It should only know how to connect, disconnect, and read a value.

---

## 21. Should the fake device source know about sessions?

No.

A device source should not know which session is active.

Avoid this:

```kotlin
deviceDataSource.readValue(sessionId)
```

Better:

```kotlin
val rawValue = deviceDataSource.readValue()
```

Then the repository adds session context:

```kotlin
MeasurementEntity(
    sessionId = sessionId,
    rawValue = rawValue,
    ...
)
```

This keeps device communication separate from research data modelling.

---

## 22. Error behaviour

If the repository calls:

```kotlin
deviceDataSource.readValue()
```

before connection, the fake source throws:

```text
IllegalStateException("Device is not connected")
```

This is good.

Later, the ViewModel should catch this:

```kotlin
try {
    val measurement =
        measurementRepository.readAndSaveMeasurement(
            sessionId = sessionId,
            repetition = repetition
        )
} catch (e: Exception) {
    uiState = uiState.copy(
        message = "Failed to read measurement"
    )
}
```

The repository does not need to hide every error.

The ViewModel can decide what message to show.

---

## 23. Current architecture after Lesson 27

After Lesson 27, the app architecture becomes:

```text
ResearchViewModel
 ↓
MeasurementRepository
 ├── DeviceDataSource
 │    └── FakeDeviceDataSource
 │
 └── ResearchDatabase
      ├── PatientDao
      ├── SessionDao
      ├── MeasurementDao
      └── ResultDao
```

The fake acquisition path is:

```text
FakeDeviceDataSource.readValue()
 ↓
rawValue
 ↓
MeasurementRepository.createMeasurementFromDevice()
 ↓
MeasurementEntity
 ↓
MeasurementRepository.insertMeasurement()
 ↓
Room database
```

This is the first working acquisition structure.

---

## 24. Current files after Lesson 27

After this lesson, the project should include:

```text
device
 ├── DeviceDataSource.kt
 └── FakeDeviceDataSource.kt
```

And the repository should now use:

```text
DeviceDataSource
FakeDeviceDataSource
```

Your project structure is now:

```text
com.example.researchapp
 ├── data
 │    ├── ResearchDatabase.kt
 │    ├── MeasurementRepository.kt
 │    ├── entity
 │    └── dao
 │
 ├── device
 │    ├── DeviceDataSource.kt
 │    └── FakeDeviceDataSource.kt
 │
 ├── processing
 ├── ml
 ├── ui
 ├── viewmodel
 └── export
```

---

## 25. What you learned in Lesson 27

You created:

```text
DeviceDataSource
FakeDeviceDataSource
```

You learned that:

```text
DeviceDataSource
is the interface for fake or real device input.
```

```text
FakeDeviceDataSource
simulates a device by connecting, disconnecting, and returning random values.
```

You updated the repository so it can:

```text
connectDevice()
disconnectDevice()
createMeasurementFromDevice()
readAndSaveMeasurement()
```

The most important mental model is:

```text
The device source provides raw data.
The repository adds research context and saves it.
```

So the correct flow is:

```text
DeviceDataSource
 ↓
raw value

MeasurementRepository
 ↓
sessionId + repetition + raw value
 ↓
MeasurementEntity
 ↓
Room database
```

This keeps device communication separate from research data storage.

---

## 26. Lesson 28 preview

In Lesson 28, we will connect the signal-processing layer.

Right now:

```text
processedValue = rawValue
```

That is temporary.

In Lesson 28, we will use:

```text
SignalProcessor
```

so the repository can do:

```text
rawValue
 ↓
baseline correction
 ↓
processedValue
 ↓
validity check
 ↓
MeasurementEntity
```

This will turn the fake acquisition app into a simple processed-data acquisition app.