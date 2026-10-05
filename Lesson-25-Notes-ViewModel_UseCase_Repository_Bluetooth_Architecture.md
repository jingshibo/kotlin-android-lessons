# Lesson 25 Notes - ViewModel, Use Case, Repository, and Bluetooth Architecture

Lesson 25 turns the app's conceptual architecture into a clean Android project structure.

This companion note goes deeper into how the presentation, domain/application, data, device, and Android platform parts should depend on one another.

This companion note answers the larger architecture questions that appear when the fake data source becomes a real Bluetooth implementation:

```text
Where does the ViewModel belong?
Where does a use case belong?
Can a ViewModel call a repository directly?
When should a use case coordinate multiple repositories?
Is BluetoothManager a repository?
Where does Android's BluetoothManager belong?
What is the difference between saved device data and a live connection?
```

The short mental model is:

```text
UI asks for an action
 ↓
ViewModel manages screen state
 ↓
Use case coordinates a complex business operation, when one is needed
 ↓
Repositories expose application data
 ↓
Data sources and Android APIs perform the low-level work
```

A use case is optional. A ViewModel may call a repository directly when the operation is simple.

---

## 1. Use consistent layer names

Architecture terminology varies between books and frameworks.

For this tutorial, use these meanings:

```text
Presentation / UI layer
    Compose screens
    ViewModels
    UI state

Domain / Application layer
    use cases
    reusable business rules
    domain models

Data layer
    repository implementations
    database data sources
    Bluetooth data sources
    mapping and synchronization

Platform / Infrastructure
    Room
    Android Bluetooth APIs
    files
    network libraries
```

Some Clean Architecture descriptions call the use-case layer the **application layer**. Android's architecture guidance usually calls it the optional **domain layer**.

In both descriptions, the important dependency direction is the same:

```text
Presentation
 ↓
Domain/Application
 ↓
Data
 ↓
Platform APIs
```

Do not decide a class's layer from its name alone. Decide from what the class is responsible for and what it depends on.

---

## 2. ViewModel and use case are not the same layer

A ViewModel belongs to the presentation/UI layer.

A use case belongs to the domain/application layer.

```text
MeasurementScreen
        ↓ sends user event
MeasurementViewModel
        ↓ calls
StartRecordingUseCase
        ↓ calls
repositories and device gateways
```

Results travel back in the opposite direction:

```text
repositories and device gateways
        ↓ return data or Flow
StartRecordingUseCase
        ↓ returns result
MeasurementViewModel
        ↓ updates UiState
MeasurementScreen
        ↓ recomposes
```

The ViewModel is therefore **above** the use case in the dependency graph because it is closer to the UI.

The ViewModel does not need to cross into the domain layer. It remains a presentation component that calls the domain layer through public functions.

### ViewModel responsibilities

A ViewModel normally handles:

- screen-level UI state
- user events from the screen
- `viewModelScope`
- loading, success, and error presentation
- deciding which use case or repository operation to call
- converting returned data into `UiState`

Examples of ViewModel state:

```kotlin
data class MeasurementUiState(
    val isLoading: Boolean = false,
    val connectionText: String = "Disconnected",
    val canStartMeasurement: Boolean = false,
    val errorMessage: String? = null
)
```

These fields describe what the screen should display.

### Use-case responsibilities

A use case normally handles one business operation or rule:

- verify that a session exists before recording
- require a connected sensor before starting acquisition
- coordinate several repositories
- apply the same rule for multiple ViewModels
- keep a complex workflow out of a ViewModel

A use case should not normally decide:

```text
whether a Compose dialog is visible
which tab is selected
what Snackbar text is displayed
which color a status label uses
```

Those are presentation decisions.

A use case should also normally remain lightweight and avoid owning mutable UI or Bluetooth connection state. The ViewModel owns screen state, while the repository or device gateway owns the live data state it exposes. A streaming use case can return or transform a `Flow` without storing Android `BluetoothGatt` objects itself.

---

## 3. A ViewModel can call a repository directly

A use case is not required between every ViewModel and repository.

For a simple data operation, this is a good structure:

```text
PatientScreen
 ↓
PatientViewModel
 ↓
PatientRepository
```

Example:

```kotlin
class PatientViewModel(
    private val patientRepository: PatientRepository
) : ViewModel() {

    val patients = patientRepository.observePatients()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = emptyList()
        )

    fun addPatient(patient: PatientRecord) {
        viewModelScope.launch {
            patientRepository.addPatient(patient)
        }
    }
}
```

Adding this use case would usually provide little value:

```kotlin
class AddPatientUseCase(
    private val patientRepository: PatientRepository
) {
    suspend operator fun invoke(patient: PatientRecord) {
        patientRepository.addPatient(patient)
    }
}
```

It only forwards the same arguments to the same function and contains no additional rule.

The practical rule is:

```text
Simple data operation
 → ViewModel can call repository directly

Complex or reusable business workflow
 → ViewModel calls a use case
```

---

## 4. When a use case is useful

Create a use case when an operation:

- coordinates multiple repositories or gateways
- contains an important business rule
- is reused by multiple ViewModels
- has several ordered steps
- needs centralized validation or error handling
- would otherwise make the ViewModel difficult to read or test

For example, starting a recording may require all of these conditions:

```text
patient exists
 ↓
session exists
 ↓
sensor is connected
 ↓
device is prepared
 ↓
recording starts
 ↓
session status is updated
```

That is a reasonable use case:

```kotlin
class StartRecordingUseCase(
    private val patientRepository: PatientRepository,
    private val sessionRepository: SessionRepository,
    private val sensorGateway: SensorDeviceGateway
) {
    suspend operator fun invoke(
        patientCode: String,
        sessionId: Long
    ) {
        val patient = patientRepository.findPatient(patientCode)
            ?: throw IllegalArgumentException("Patient not found")

        val session = sessionRepository.findSession(sessionId)
            ?: throw IllegalArgumentException("Session not found")

        check(sensorGateway.connectionState.value == DeviceConnectionState.CONNECTED) {
            "Sensor is not connected"
        }

        sensorGateway.startMeasurement()
        sessionRepository.markRecordingStarted(session.id)
    }
}
```

The ViewModel handles the screen state around that operation:

```kotlin
class MeasurementViewModel(
    private val startRecording: StartRecordingUseCase
) : ViewModel() {

    fun onStartClicked(
        patientCode: String,
        sessionId: Long
    ) {
        viewModelScope.launch {
            try {
                updateLoadingState(true)
                startRecording(patientCode, sessionId)
                updateRecordingState()
            } catch (exception: Exception) {
                updateErrorState(exception.message)
            }
        }
    }
}
```

The ViewModel knows how to represent progress and errors on the screen. The use case knows the rules for starting recording.

---

## 5. Multiple repositories in one use case

Injecting multiple repositories into a use case is normal when one business operation needs several kinds of data.

```kotlin
class CompleteDatasetTransferUseCase(
    private val patientRepository: PatientRepository,
    private val sessionRepository: SessionRepository,
    private val resultRepository: ResultRepository,
    private val sensorGateway: SensorDeviceGateway
)
```

Each dependency should still have a coherent responsibility:

```text
PatientRepository
    patient data

SessionRepository
    session data

ResultRepository
    result data

SensorDeviceGateway
    live device communication

CompleteDatasetTransferUseCase
    coordinates the complete workflow
```

This keeps the ViewModel from becoming the place where every business step is manually connected.

### Repositories are not automatically one per table

A repository represents a coherent area of application data. It is not required to match one Room table exactly.

A repository may use:

- one DAO
- several DAOs
- a local database and a network source
- a Bluetooth source and a cache

Splitting repositories by patient, session, and result may be sensible for this app, but do not turn `one table = one repository` into an absolute rule.

### Think about atomic database work

If several database changes must either all succeed or all fail, simply calling several repositories in sequence may not provide one database transaction.

For example:

```text
save session
save 100 measurements
save result
mark session complete
```

If those writes must be atomic, expose a data-layer operation that performs them inside a Room transaction. The domain use case can request the operation, but the Room transaction boundary belongs in the data layer.

---

## 6. Evaluate `DataTransmissionManager` by responsibility

The word `Manager` does not determine a layer.

`DataTransmissionManager` belongs to the domain/application layer only if it coordinates a business workflow such as:

```text
find patient
validate session
start sensor
receive dataset
save measurements
save result
complete session
```

In that case, use an action-focused name:

```kotlin
class TransferDatasetUseCase(...)
```

or:

```kotlin
class RecordMeasurementSessionUseCase(...)
```

These names explain what operation the class performs.

`DataTransmissionManager` does **not** belong to the domain layer if it mainly performs low-level work such as:

- calling Android Bluetooth APIs
- managing `BluetoothGatt`
- opening and closing sockets
- decoding characteristic notifications
- maintaining scan callbacks
- handling platform permissions

That work belongs to a Bluetooth data-source or gateway implementation.

If one class currently performs both business coordination and Bluetooth operations, split it:

```text
RecordMeasurementSessionUseCase
    business workflow

BluetoothSensorDeviceGateway
    Android Bluetooth communication
```

---

## 7. Saved device information and a live connection are different

The word `device` can mean two different things in this app.

### Persistent device information

`DeviceRepository` may store information that must remain after the app closes:

```text
device code
display name
Bluetooth address or stable identifier
calibration information
last-used time
research configuration
whether the device is active or retired
```

Example:

```kotlin
interface DeviceRepository {
    fun observeSavedDevices(): Flow<List<DeviceRecord>>

    suspend fun findDevice(
        deviceId: String
    ): DeviceRecord?

    suspend fun saveDevice(
        device: DeviceRecord
    )

    suspend fun updateCalibration(
        deviceId: String,
        calibration: CalibrationData
    )
}
```

This repository may use Room.

### Live device communication

A live-device interface handles temporary runtime communication:

```text
scan
connect
disconnect
receive sensor packets
start and stop measurement
send preparation or maintenance commands
```

Your existing custom `BluetoothManager` interface already performs this role.

Therefore, do not add a separate `DeviceConnectionRepository` that duplicates all the same functions.

You can either keep the existing interface or rename it to clarify its role.

---

## 8. A clearer name for the custom Bluetooth interface

Android already provides a class named:

```kotlin
android.bluetooth.BluetoothManager
```

Naming the app's own interface `BluetoothManager` creates two different types with the same name.

A clearer Bluetooth-specific name is:

```kotlin
interface BluetoothSensorController
```

If the app may later support USB or Wi-Fi, use a transport-independent name:

```kotlin
interface SensorDeviceGateway
```

The word **gateway** means that this interface is the app's doorway to an external device.

An adapted version of the current interface could be:

```kotlin
interface SensorDeviceGateway {
    val connectionState: StateFlow<DeviceConnectionState>
    val activeDevice: StateFlow<SensorDevice?>
    val discoveredDevices: StateFlow<List<SensorDevice>>
    val sensorDataStream: Flow<SensorDataPacket>

    val isMeasuring: StateFlow<Boolean>
    val isScanning: StateFlow<Boolean>

    fun startScan()
    fun stopScan()

    fun connect(deviceId: String)
    fun disconnect()

    fun startMeasurement()
    fun stopMeasurement()

    fun prepareDevice(deviceId: String)
    fun clearDeviceData(deviceId: String)
    fun rebootDevice()
}
```

If the interface remains Bluetooth-specific, names such as `BluetoothSensorDevice` and `deviceAddress` are appropriate.

If the interface is intended to support several transports, prefer general names such as `SensorDevice` and `deviceId`. The Bluetooth implementation can internally translate that identifier into a Bluetooth address.

---

## 9. Relationship to Android's `BluetoothManager`

The custom gateway and Android's platform manager have different jobs:

```text
SensorDeviceGateway
    app-facing contract
 ↓ implemented by
BluetoothSensorDeviceGateway
    app's Bluetooth implementation
 ↓ uses
android.bluetooth.BluetoothManager
    Android system service
 ↓ provides
BluetoothAdapter
 ↓ finds
BluetoothDevice
 ↓ connects through
BluetoothGatt for BLE
or BluetoothSocket for Bluetooth Classic
```

Android's `BluetoothManager` is obtained from `Context`:

```kotlin
val platformBluetoothManager =
    context.getSystemService(
        android.bluetooth.BluetoothManager::class.java
    )

val bluetoothAdapter =
    platformBluetoothManager.adapter
```

It provides access to the tablet's Bluetooth adapter and overall Bluetooth state. It normally does not perform the final client connection itself.

For Bluetooth Low Energy, the connection is normally created from a `BluetoothDevice`:

```kotlin
bluetoothGatt = device.connectGatt(
    context,
    false,
    bluetoothGattCallback
)
```

For Bluetooth Classic, the connection normally uses a `BluetoothSocket`:

```kotlin
val socket =
    device.createRfcommSocketToServiceRecord(serviceUuid)

socket.connect()
```

The Android platform types should stay inside the concrete Bluetooth implementation rather than appearing in the ViewModel or use case.

---

## 10. Example Bluetooth implementation boundary

The concrete implementation can use Android APIs while exposing only app models through the interface:

```kotlin
class BluetoothSensorDeviceGateway(
    private val context: Context
) : SensorDeviceGateway {

    private val platformBluetoothManager =
        context.getSystemService(
            android.bluetooth.BluetoothManager::class.java
        )

    private val bluetoothAdapter =
        platformBluetoothManager.adapter

    private var bluetoothGatt: BluetoothGatt? = null

    private val _connectionState =
        MutableStateFlow(DeviceConnectionState.DISCONNECTED)

    override val connectionState:
        StateFlow<DeviceConnectionState> =
        _connectionState.asStateFlow()

    private val gattCallback =
        object : BluetoothGattCallback() {

            override fun onConnectionStateChange(
                gatt: BluetoothGatt,
                status: Int,
                newState: Int
            ) {
                _connectionState.value =
                    when (newState) {
                        BluetoothProfile.STATE_CONNECTED ->
                            DeviceConnectionState.CONNECTED

                        else ->
                            DeviceConnectionState.DISCONNECTED
                    }
            }
        }

    override fun connect(deviceId: String) {
        // Check BLUETOOTH_CONNECT before using protected APIs.
        val device = bluetoothAdapter.getRemoteDevice(deviceId)

        bluetoothGatt = device.connectGatt(
            context,
            false,
            gattCallback
        )
    }

    // Implement the remaining SensorDeviceGateway members.
}
```

This is a structural example, not the complete production BLE implementation. A real version also needs permission checks, scan handling, GATT service discovery, characteristic subscriptions, error handling, cleanup, and thread-safe state updates.

The important boundary is:

```text
ViewModel and use case know SensorDeviceGateway

Only BluetoothSensorDeviceGateway knows
android.bluetooth.BluetoothManager and BluetoothGatt
```

---

## 11. Where Bluetooth permissions belong

Android does not request Bluetooth permissions automatically when the platform `BluetoothManager` is obtained.

The responsibilities are divided like this:

```text
Compose screen
    launches the Android permission dialog

ViewModel
    represents permission status in UI state

Bluetooth gateway implementation
    checks permission before protected Bluetooth operations
```

On Android 12 and above:

```text
scan
 → BLUETOOTH_SCAN

connect or communicate
 → BLUETOOTH_CONNECT
```

The UI may remember that permission was previously granted, but the real Bluetooth implementation must still protect operations because permission can later be revoked.

---

## 12. Should the live-device interface be split?

The current interface covers:

```text
discovery
connection
measurement streaming
device commands
```

These responsibilities are related closely enough that one interface can be acceptable at the current project size.

Do not split it merely to create more architecture layers.

Split it later if the responsibilities grow independently:

```kotlin
interface DeviceDiscovery {
    val discoveredDevices: StateFlow<List<SensorDevice>>
    val isScanning: StateFlow<Boolean>

    fun startScan()
    fun stopScan()
}

interface DeviceConnection {
    val connectionState: StateFlow<DeviceConnectionState>
    val activeDevice: StateFlow<SensorDevice?>

    fun connect(deviceId: String)
    fun disconnect()
}

interface SensorMeasurementSource {
    val sensorDataStream: Flow<SensorDataPacket>
    val isMeasuring: StateFlow<Boolean>

    fun startMeasurement()
    fun stopMeasurement()
}
```

One Bluetooth class may implement all three:

```kotlin
class BluetoothSensorController(
    private val context: Context
) : DeviceDiscovery,
    DeviceConnection,
    SensorMeasurementSource {
    ...
}
```

Use this separation only when it improves clarity, testing, or reuse.

---

## 13. Recommended dependency structure

For the research app, a clear structure is:

```text
MeasurementScreen
        ↓ user actions
MeasurementViewModel
        ├── directly calls simple repository operations
        │
        └── calls RecordMeasurementSessionUseCase
                    ├── PatientRepository
                    ├── SessionRepository
                    ├── ResultRepository
                    └── SensorDeviceGateway
                                ↓ implemented by
                         BluetoothSensorDeviceGateway
                                ↓ uses
                    Android BluetoothManager / GATT
```

Persistent device metadata is separate:

```text
DeviceRepository
 ↓
DeviceDao
 ↓
Room
```

The use case may depend on both when the workflow needs both saved metadata and a live connection:

```kotlin
class RecordMeasurementSessionUseCase(
    private val deviceRepository: DeviceRepository,
    private val sessionRepository: SessionRepository,
    private val resultRepository: ResultRepository,
    private val sensorGateway: SensorDeviceGateway
)
```

That is not duplication because the two device dependencies have different jobs:

```text
DeviceRepository
    remembers information about devices

SensorDeviceGateway
    communicates with the currently active device
```

---

## 14. Suggested package structure

One possible project structure is:

```text
presentation/
    measurement/
        MeasurementScreen.kt
        MeasurementViewModel.kt
        MeasurementUiState.kt

domain/
    model/
        SensorDevice.kt
        SensorDataPacket.kt
        DeviceConnectionState.kt

    gateway/
        SensorDeviceGateway.kt

    usecase/
        StartRecordingUseCase.kt
        StopRecordingUseCase.kt
        RecordMeasurementSessionUseCase.kt

data/
    repository/
        PatientRepository.kt
        SessionRepository.kt
        DeviceRepository.kt
        ResultRepository.kt

    bluetooth/
        BluetoothSensorDeviceGateway.kt
        SensorGattCallback.kt
        SensorPacketDecoder.kt

    local/
        ResearchDatabase.kt
        PatientDao.kt
        SessionDao.kt
        DeviceDao.kt
        ResultDao.kt
```

This is an example, not a rule. A small app can use fewer packages while preserving the same dependency direction.

---

## 15. Common mistakes

### Mistake 1: use case for every repository function

```text
ViewModel
 ↓
GetPatientUseCase
 ↓
PatientRepository.getPatient()
```

If the use case only forwards the call and provides no rule or reuse, the ViewModel can call the repository directly.

### Mistake 2: business workflow inside the ViewModel

```text
ViewModel manually coordinates four repositories,
Bluetooth callbacks, validation, and database transactions
```

Move the reusable business workflow into a use case and keep UI-state handling in the ViewModel.

### Mistake 3: Android Bluetooth types in the domain layer

Avoid domain/use-case APIs that expose:

```text
BluetoothGatt
BluetoothAdapter
BluetoothDevice
ScanResult
Context
```

Map them to app models inside the Bluetooth implementation.

### Mistake 4: duplicate connection abstractions

Do not create both:

```text
BluetoothManager
DeviceConnectionRepository
```

when they expose the same state and operations. Keep one interface with one clear name.

### Mistake 5: one large `Manager` doing everything

If a class owns UI state, business rules, Room access, Bluetooth callbacks, parsing, and export, it has too many responsibilities.

Separate by reason to change:

```text
ViewModel
    UI state changes

Use case
    workflow rules change

Repository
    data policy changes

Bluetooth gateway
    device protocol changes

Parser
    message format changes
```

---

## 16. Decision guide

Ask these questions when placing a class:

### Does it manage screen state?

```text
Yes → ViewModel / presentation layer
```

### Does it perform one complex or reusable business operation?

```text
Yes → use case / domain-application layer
```

### Does it expose and coordinate application data?

```text
Yes → repository / data layer
```

### Does it scan, connect, read bytes, or call `BluetoothGatt`?

```text
Yes → Bluetooth gateway or data source implementation
```

### Is the repository call simple?

```text
Yes → ViewModel may call it directly
```

### Does one operation need several repositories and rules?

```text
Yes → introduce a use case
```

---

## 17. Final mental model

```text
ViewModel
    owns screen state and responds to UI events

Use case
    coordinates a meaningful business operation when needed

Repository
    exposes a coherent area of application data

SensorDeviceGateway
    exposes live sensor communication to the app

BluetoothSensorDeviceGateway
    implements that gateway using Android Bluetooth APIs

android.bluetooth.BluetoothManager
    gives the implementation access to the device's BluetoothAdapter
```

The most important rules are:

```text
A ViewModel may call a repository directly for simple work.

Use cases are useful for complex or reusable workflows.

A use case may depend on multiple repositories.

Saved device records and a live device connection are different responsibilities.

Your custom Bluetooth interface already acts as the live-device gateway.

Do not add another interface that duplicates it.

Keep Android Bluetooth classes inside the concrete Bluetooth implementation.
```

## Further reading

- [Android domain layer](https://developer.android.com/topic/architecture/domain-layer)
- [Android data layer](https://developer.android.com/topic/architecture/data-layer)
- [Android architecture recommendations](https://developer.android.com/topic/architecture/recommendations)
- [Set up Bluetooth](https://developer.android.com/develop/connectivity/bluetooth/setup)
- [Connect to a BLE GATT server](https://developer.android.com/develop/connectivity/bluetooth/ble/connect-gatt-server)

