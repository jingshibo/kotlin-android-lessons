# Lesson 18 — Connecting a Real Data Source

In Lesson 17, we prepared the app for real device communication.

We discussed:

```text
permissions
Bluetooth/Wi-Fi/USB communication idea
device connection state
where real device code should belong
```

The most important design rule was:

```text
Do not put hardware communication directly inside the UI.
## 22. Real Data May Need Parsing


The UI should not directly talk to Bluetooth, Wi-Fi, USB, or sensors.

Instead, we want this structure:

```text
MeasurementScreen
 ↓
ResearchViewModel
 ↓
MeasurementRepository
 ↓
DeviceDataSource
 ↓
Fake / Bluetooth / Wi-Fi / USB device
```

This follows the same practical direction as the original tutorial: first build the app with fake/random data, then later replace fake data with real device input. The file explicitly introduced this idea earlier by generating random values first and replacing them with real device input after the app works. fileciteturn0file0L140-L144

---

## 1. What problem are we solving?

Until now, our measurement value came from this kind of code:

```kotlin
val value = Random.nextDouble(0.0, 5.0)
```

This was useful for learning.

But a real research app should eventually get data from something like:

```text
Bluetooth device
Wi-Fi device
USB device
tablet sensor
external acquisition system
```

The problem is that we do not want to rewrite the whole app when we switch from fake data to real data.

Bad structure:
## 24. Where Should the Parser Live?

```text
UI directly generates fake data
 ↓
later UI directly talks to Bluetooth
 ↓
later UI directly parses messages
 ↓
UI becomes messy
```

Better structure:

```text
UI asks ViewModel to start acquisition
 ↓
ViewModel asks Repository for measurement
 ↓
Repository asks DeviceDataSource for value
 ↓
DeviceDataSource decides where value comes from
```

This means the app can first use:

```text
FakeDeviceDataSource
```

and later use:

```text
BluetoothDeviceDataSource
```

without changing the screen much.

---

## 2. Main idea of Lesson 18

We will introduce this interface:

```kotlin
interface DeviceDataSource {
    suspend fun connect()
    suspend fun disconnect()
    suspend fun readValue(): Double
}
```

This interface means:

```text
Any device data source must know how to:
connect
disconnect
read one value
```

The ViewModel and Repository do not need to know whether the value comes from:

```text
random number
Bluetooth
Wi-Fi
USB
file replay
```

They only know:

```kotlin
deviceDataSource.readValue()
```

This is a very important software design idea.

---

## 3. Why use an interface?

An interface defines a contract.

For example:

```kotlin
interface DeviceDataSource {
    suspend fun connect()
    suspend fun disconnect()
    suspend fun readValue(): Double
}
```

This says:

```text
I do not care how you connect.
I do not care how you read the value.
But if you are a DeviceDataSource, you must provide these functions.
```

Then we can create different implementations.

Fake version:

```text
FakeDeviceDataSource
```

Real Bluetooth version later:

```text
BluetoothDeviceDataSource
```

Real Wi-Fi version later:

```text
WifiDeviceDataSource
```

The rest of the app can use the same interface.

---

## 4. Create `DeviceDataSource.kt`

Create a new Kotlin file:

```text
DeviceDataSource.kt
```

Add:

```kotlin
interface DeviceDataSource {
    suspend fun connect()
    suspend fun disconnect()
    suspend fun readValue(): Double
}
```

This file is small but important.

It gives the app a stable boundary between:

```text
app logic
```

and:

```text
device communication
```

---

## 5. Create fake device data source

Now create another file:

```text
FakeDeviceDataSource.kt
```

Add:

```kotlin
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
            throw IllegalStateException("Device is not connected")
        }

        delay(1000)

        return Random.nextDouble(0.0, 5.0)
    }
}
```

Let us understand this.

---

## 6. `connected` variable

Inside `FakeDeviceDataSource`, we have:

```kotlin
private var connected: Boolean = false
```

This stores whether the fake device is connected.

At first:

```text
connected = false
```

When `connect()` runs:

```kotlin
connected = true
```

When `disconnect()` runs:

```kotlin
connected = false
```

This simulates a real device connection.

---

## 7. Fake connect

```kotlin
override suspend fun connect() {
    delay(1000)
    connected = true
}
```

This means:

```text
wait 1 second
then mark fake device as connected
```

Why delay?

Because real device connection usually takes time.

For example:

```text
scan for device
connect socket
open stream
check handshake
```

So this fake version behaves more realistically than instantly connecting.

---

## 8. Fake disconnect

```kotlin
override suspend fun disconnect() {
    connected = false
}
```

This is simple.

In a real Bluetooth or Wi-Fi data source, this might close a socket, stream, or connection.

But for fake data, setting `connected = false` is enough.

---

## 9. Fake read value

```kotlin
override suspend fun readValue(): Double {
    if (!connected) {
        throw IllegalStateException("Device is not connected")
    }

    delay(1000)

    return Random.nextDouble(0.0, 5.0)
}
```

This function does three things:

```text
check device is connected
wait 1 second
return a fake measurement value
```

The check is important:

```kotlin
if (!connected) {
    throw IllegalStateException("Device is not connected")
}
```

This prevents the app from pretending it can read data when no device is connected.

This is a good habit for real research apps.

---

## 10. Update repository to use `DeviceDataSource`

In Lesson 15, our repository created fake measurements directly using `Random`.

Now we change that.

Before:

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

Now the repository should ask the device data source:

```kotlin
suspend fun createMeasurementFromDevice(
    sessionId: Long,
    repetition: Int
): MeasurementEntity {
    val value = deviceDataSource.readValue()

    return MeasurementEntity(
        sessionId = sessionId,
        repetition = repetition,
        value = value,
        timestamp = System.currentTimeMillis(),
        status = "OK"
    )
}
```

The key change is:

```kotlin
val value = deviceDataSource.readValue()
```

This means the measurement value no longer has to come from `Random` inside the repository.

It comes from the data source.

---

## 11. Updated repository constructor

The repository can receive a `DeviceDataSource`.

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

```kotlin
BluetoothDeviceDataSource()
```

or:

```kotlin
WifiDeviceDataSource()
```

without changing the main ViewModel and UI structure.

---

## 12. Add connect/disconnect functions to repository

The repository can expose device functions:

```kotlin
suspend fun connectDevice() {
    deviceDataSource.connect()
}

suspend fun disconnectDevice() {
    deviceDataSource.disconnect()
}
```

So the ViewModel does not call the data source directly.

The flow remains:

```text
ViewModel
 ↓
Repository
 ↓
DeviceDataSource
```

This keeps the architecture consistent.

---

## 13. Updated repository pattern

A simplified repository now looks like this:

```kotlin
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
        val value = deviceDataSource.readValue()

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

    suspend fun endSession(
        sessionId: Long,
        endedAt: Long
    ) {
        sessionDao.endSession(
            sessionId = sessionId,
            endedAt = endedAt
        )
    }

    suspend fun saveResult(
        result: ResultEntity
    ) {
        resultDao.insertResult(result)
    }
}
```

The important new functions are:

```kotlin
connectDevice()
disconnectDevice()
createMeasurementFromDevice()
```

---

## 14. Update ViewModel `connectDevice()`

In Lesson 17, `connectDevice()` simulated connection directly inside the ViewModel:

```kotlin
viewModelScope.launch {
    delay(1000)

    uiState = uiState.copy(
        deviceConnectionState = DeviceConnectionState.CONNECTED,
        message = "Device connected"
    )
}
```

Now the ViewModel should ask the repository:

```kotlin
measurementRepository.connectDevice()
```

Updated version:

```kotlin
fun connectDevice() {
    if (uiState.bluetoothPermissionState != PermissionState.GRANTED) {
        uiState = uiState.copy(
            message = "Bluetooth permission is required before connecting"
        )
        return
    }

    uiState = uiState.copy(
        deviceConnectionState = DeviceConnectionState.CONNECTING,
        message = "Connecting to device..."
    )

    viewModelScope.launch {
        try {
            measurementRepository.connectDevice()

            uiState = uiState.copy(
                deviceConnectionState = DeviceConnectionState.CONNECTED,
                message = "Device connected"
            )
        } catch (e: Exception) {
            uiState = uiState.copy(
                deviceConnectionState = DeviceConnectionState.ERROR,
                message = "Device connection failed"
            )
        }
    }
}
```

Now the ViewModel does not know whether connection is fake or real.

It only knows:

```text
repository, please connect the device
```

---

## 15. Update ViewModel `disconnectDevice()`

Before, disconnection was mostly state change.

Now we ask repository to disconnect:

```kotlin
fun disconnectDevice() {
    if (uiState.acquisitionState == AcquisitionState.RECORDING) {
        stopAcquisition()
    }

    viewModelScope.launch {
        try {
            measurementRepository.disconnectDevice()

            uiState = uiState.copy(
                deviceConnectionState = DeviceConnectionState.DISCONNECTED,
                acquisitionState = AcquisitionState.IDLE,
                message = "Device disconnected"
            )
        } catch (e: Exception) {
            uiState = uiState.copy(
                deviceConnectionState = DeviceConnectionState.ERROR,
                message = "Device disconnection failed"
            )
        }
    }
}
```

The structure is:

```text
if recording, stop first
 ↓
ask repository to disconnect
 ↓
update state
```

This is safer than simply changing the UI state without actually disconnecting the device.

---

## 16. Update measurement acquisition

In Lesson 15, we used:

```kotlin
measurementRepository.createSimulatedMeasurement(...)
```

Now we use:

```kotlin
measurementRepository.createMeasurementFromDevice(...)
```

Updated function:

```kotlin
private fun addMeasurementFromDevice(
    sessionId: Long
) {
    viewModelScope.launch {
        try {
            val newMeasurement =
                measurementRepository.createMeasurementFromDevice(
                    sessionId = sessionId,
                    repetition = uiState.measurements.size + 1
                )

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
                acquisitionState = AcquisitionState.ERROR,
                message = "Failed to read measurement from device"
            )
        }
    }
}
```

The important change is conceptual:

```text
Before:
create fake measurement

Now:
read value from DeviceDataSource
create measurement
save to Room
```

Even though the current data source is still fake, the architecture now looks real.

---

## 17. Update acquisition loop

In Lesson 15, `startAcquisition()` called:

```kotlin
addSimulatedMeasurement(sessionId)
```

Now it should call:

```kotlin
addMeasurementFromDevice(sessionId)
```

So:

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
            addMeasurementFromDevice(
                sessionId = sessionId
            )
        }
    }
}
```

Notice something important.

In Lesson 11, we had:

```kotlin
delay(1000)
```

inside the acquisition loop.

Now `readValue()` already contains a delay in the fake data source:

```kotlin
override suspend fun readValue(): Double {
    delay(1000)
    return Random.nextDouble(0.0, 5.0)
}
```

So we do not need another `delay(1000)` in the ViewModel loop.

This is better because the data source controls how quickly values arrive.

---

## 18. Important problem: nested coroutines

Look carefully at this version:

```kotlin
viewModelScope.launch {
    while (uiState.acquisitionState == AcquisitionState.RECORDING) {
        addMeasurementFromDevice(sessionId)
    }
}
```

But `addMeasurementFromDevice()` itself also starts:

```kotlin
viewModelScope.launch {
    ...
}
```

That means we may accidentally start many coroutines.

For Lesson 18, a cleaner version is to make `addMeasurementFromDevice()` a `suspend` function.

So instead of:

```kotlin
private fun addMeasurementFromDevice(sessionId: Long) {
    viewModelScope.launch {
        ...
    }
}
```

write:

```kotlin
private suspend fun addMeasurementFromDevice(
    sessionId: Long
) {
    val newMeasurement =
        measurementRepository.createMeasurementFromDevice(
            sessionId = sessionId,
            repetition = uiState.measurements.size + 1
        )

    measurementRepository.insertMeasurement(newMeasurement)

    val updatedMeasurements =
        measurementRepository.getMeasurementsForSession(sessionId)

    uiState = uiState.copy(
        measurements = updatedMeasurements,
        latestValue = newMeasurement.value,
        message = "Measurement saved"
    )
}
```

Then the loop calls it directly:

```kotlin
viewModelScope.launch {
    while (uiState.acquisitionState == AcquisitionState.RECORDING) {
        try {
            addMeasurementFromDevice(sessionId)
        } catch (e: Exception) {
            uiState = uiState.copy(
                acquisitionState = AcquisitionState.ERROR,
                message = "Failed to read measurement from device"
            )
        }
    }
}
```

This is cleaner.

The rule is:

```text
One coroutine loop controls acquisition.
The function inside the loop can be suspend.
Do not create a new coroutine for every single value unless you need to.
```

---

## 19. Cleaner `startAcquisition()` version

So the better Lesson 18 version is:

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
            try {
                addMeasurementFromDevice(
                    sessionId = sessionId
                )
            } catch (e: Exception) {
                uiState = uiState.copy(
                    acquisitionState = AcquisitionState.ERROR,
                    message = "Failed to read measurement from device"
                )
            }
        }
    }
}
```

And:

```kotlin
private suspend fun addMeasurementFromDevice(
    sessionId: Long
) {
    val newMeasurement =
        measurementRepository.createMeasurementFromDevice(
            sessionId = sessionId,
            repetition = uiState.measurements.size + 1
        )

    measurementRepository.insertMeasurement(newMeasurement)

    val updatedMeasurements =
        measurementRepository.getMeasurementsForSession(sessionId)

    uiState = uiState.copy(
        measurements = updatedMeasurements,
        latestValue = newMeasurement.value,
        message = "Measurement saved"
    )
}
```

This is a good beginner-friendly acquisition structure.

---

## 20. Stop Acquisition

stopAcquisition() can stay similar:

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

When acquisitionState becomes STOPPED, the loop condition becomes false:

```kotlin
while (uiState.acquisitionState == AcquisitionState.RECORDING)
```

So the acquisition loop stops.

## 21. What About Real Bluetooth Data?

A real Bluetooth data source may look conceptually like this:

```kotlin
class BluetoothDeviceDataSource : DeviceDataSource {

    override suspend fun connect() {
        // later:
        // find paired device
        // open Bluetooth socket
        // open input stream
    }

    override suspend fun disconnect() {
        // later:
        // close input stream
        // close socket
    }

    override suspend fun readValue(): Double {
        // later:
        // read raw message from input stream
        // parse it into Double
        return 0.0
    }
}
```

Do not implement this yet.
The point of Lesson 18 is to prepare the app so that this class can be added later.
The rest of the app should not need to know the Bluetooth details.

## 22. Real Data May Need Parsing

A fake data source returns a clean Double:

```kotlin
return Random.nextDouble(0.0, 5.0)
```

But a real device may send:

```text
"2.438\n"
or:
"S001,2.438,OK\n"
or:
"VALUE:2.438;STATUS:OK"
```

So later we may need a parser.
Simple parser example:

```kotlin
fun parseSimpleValue(
    rawMessage: String
): Double {
    return rawMessage.trim().toDouble()
}
```

If the device sends comma-separated data:

```kotlin
fun parseCsvValue(
    rawMessage: String
): Double {
    val parts = rawMessage.trim().split(",")
    return parts[1].toDouble()
}
```

Example:

```kotlin
val value = parseCsvValue("S001,2.438,OK")
```

This returns:

```text
2.438
```

But this parser assumes the message format is correct.
A safer parser would check the format and handle errors.

## 23. Safer Parser Example

```kotlin
fun parseCsvValueOrNull(
    rawMessage: String
): Double? {
    val parts = rawMessage.trim().split(",")

    if (parts.size < 2) {
        return null
    }

    return parts[1].toDoubleOrNull()
}
```

This returns:

- `Double` value if parsing works
- `null` if parsing fails

Example:

```kotlin
val value = parseCsvValueOrNull("S001,2.438,OK")
```

returns:

```text
2.438
```

But:

```kotlin
val value = parseCsvValueOrNull("bad message")
```

returns:

```text
null
```

This is safer for real device communication.

## 24. Where Should the Parser Live?

Do not put parsing code in the UI.
A better structure is:

```text
BluetoothDeviceDataSource
 ↓
reads raw message

SensorMessageParser
 ↓
converts raw message to value

MeasurementRepository
 ↓
creates MeasurementEntity

Room
 ↓
stores measurement
```

For example, later:

```kotlin
class SensorMessageParser {

    fun parseValueOrNull(
        rawMessage: String
    ): Double? {
        val parts = rawMessage.trim().split(",")

        if (parts.size < 2) {
            return null
        }

        return parts[1].toDoubleOrNull()
    }
}
```

Then the Bluetooth data source could use it:

```kotlin
class BluetoothDeviceDataSource(
    private val parser: SensorMessageParser = SensorMessageParser()
) : DeviceDataSource {

    override suspend fun readValue(): Double {
        val rawMessage = readRawMessageFromBluetooth()

        return parser.parseValueOrNull(rawMessage)
            ?: throw IllegalArgumentException("Invalid sensor message")
    }

    private fun readRawMessageFromBluetooth(): String {
        // later real Bluetooth read
        return "S001,2.438,OK"
    }
}
```

Again, not for now.
But this shows where things will go.

## 25. Why This Design Helps Research Apps

A research app may change over time.
Today:

- fake random value
- Bluetooth value
- Wi-Fi packet
- multiple channels
- raw waveform instead of one value

If the UI is directly tied to the data source, every change becomes painful.
But with this structure:

```text
DeviceDataSource
 ↓
Repository
 ↓
ViewModel
 ↓
UI
```

we can change the lower layers without rewriting the screen.

## 26. Current Architecture After Lesson 18

After this lesson, the app structure becomes:

```text
Presentation layer
 └── MeasurementScreen

ViewModel layer
 └── ResearchViewModel

Repository layer
 └── MeasurementRepository

Device/data-source layer
 ├── DeviceDataSource interface
 ├── FakeDeviceDataSource
 └── later BluetoothDeviceDataSource / WifiDeviceDataSource

Storage layer
 └── Room database
```

The acquisition path is now:

```text
Start Acquisition
 ↓
ResearchViewModel.startAcquisition()
 ↓
MeasurementRepository.createMeasurementFromDevice()
 ↓
DeviceDataSource.readValue()
 ↓
MeasurementEntity
 ↓
Room database
 ↓
UI state updates
```

This is much closer to a real research app.

## 27. What You Learned in Lesson 18

The key new interface is:

```kotlin
interface DeviceDataSource {
    suspend fun connect()
    suspend fun disconnect()
    suspend fun readValue(): Double
}
```

The fake implementation is:

```kotlin
class FakeDeviceDataSource : DeviceDataSource {
    ...
}
```

The repository now asks:

```kotlin
deviceDataSource.readValue()
```

instead of directly using:

```kotlin
Random.nextDouble(0.0, 5.0)
```

The ViewModel now calls:

```kotlin
measurementRepository.connectDevice()
measurementRepository.disconnectDevice()
measurementRepository.createMeasurementFromDevice(...)
```

The most important mental model is:

> Fake data and real data should enter the app through the same doorway.

That doorway is:

**DeviceDataSource**

This lets us build the app with fake data now, then replace the data source later.

## Lesson 19 Preview

In Lesson 19, we should add a simple signal-processing pipeline.
Right now, the device data source returns one value and we save it directly.
But many research apps need:

```text
raw data
 ↓
filtering or smoothing
 ↓
baseline correction
 ↓
feature extraction
 ↓
classification or result
```

So Lesson 19 should introduce:

- raw data vs processed data
- simple processing functions
- where signal-processing code should live
- how to keep processing separate from UI

This will prepare us for on-device ML inference later.