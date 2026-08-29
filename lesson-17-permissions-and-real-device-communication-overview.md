# Lesson 17 — Permissions and Real Device Communication Overview

In Lesson 16, we moved from one large screen to a multi-screen research app structure:

```text
Patient List Screen
 ↓
Patient Detail Screen
 ↓
Measurement Screen
 ↓
Result Screen
```

Now we prepare for the next major stage:

```text
real device communication
```

So far, our app still uses simulated measurements:

```kotlin
val value = Random.nextDouble(0.0, 5.0)
```

That was intentional. The original tutorial also suggested first building the app with fake/random data, and only later replacing it with real device input such as Bluetooth, USB, sensors, network communication, or LiteRT inference. fileciteturn0file0L140-L144 fileciteturn0file0L159-L162

In this lesson, we do **not** fully implement Bluetooth or Wi-Fi yet.

Instead, we learn the structure:

```text
Android permissions
 ↓
permission request UI
 ↓
device connection state
 ↓
fake connection now
 ↓
real connection later
```

This prepares us to replace the simulated data source safely.

---

## 1. Why permissions matter

A research app may need to access:

```text
Bluetooth device
Wi-Fi/network device
USB device
tablet sensors
camera
microphone
storage/export location
notifications
```

Android does not allow apps to access sensitive features freely.

Some permissions are granted automatically at install time. Some require the user to approve them while the app is running. Android’s permission documentation explains that permissions protect restricted data and restricted actions, and apps must be transparent about what data they access and why. citeturn659041search13

For our app, the most relevant permissions are likely:

```text
Bluetooth permissions
Network / internet permission
Possible Wi-Fi permissions
Possible notification permission later
```

---

## 2. Manifest permissions vs runtime permissions

There are two steps to understand.

### Step 1: Declare permission in `AndroidManifest.xml`

For example:

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

This tells Android:

```text
This app may need internet/network access.
```

Android apps must declare permission requests in the manifest with `<uses-permission>`. citeturn659041search29

### Step 2: Request permission at runtime if needed

Some permissions also require a runtime request.

That means the user sees a system dialog, for example:

```text
Allow this app to find, connect to, and determine the relative position of nearby devices?
```

The app must handle both cases:

```text
permission granted
permission denied
```

For Compose apps, Android documentation shows using `rememberLauncherForActivityResult()` with `ActivityResultContracts.RequestPermission()` for runtime permission requests. citeturn659041search3 For multiple permissions, the Activity Result API includes `RequestMultiplePermissions`. citeturn659041search21

---

## 3. Bluetooth permissions

Bluetooth permissions are a little confusing because Android changed them.

For apps targeting Android 12, API level 31, or higher, Android introduced these newer Bluetooth permissions:

```text
BLUETOOTH_SCAN
BLUETOOTH_ADVERTISE
BLUETOOTH_CONNECT
```

Android’s official documentation says these permissions are used for apps targeting Android 12 or higher, especially for apps that interact with Bluetooth devices without requiring device location. citeturn659041search0

For a research app that receives data from a Bluetooth device, the most important ones are usually:

```text
BLUETOOTH_SCAN
 ↓
find nearby Bluetooth devices

BLUETOOTH_CONNECT
 ↓
connect to a paired or discovered Bluetooth device
```

`BLUETOOTH_ADVERTISE` is mainly needed if your tablet/app advertises itself as a Bluetooth device, which may not be needed for your first version.

---

## 4. Bluetooth permissions in the manifest

A simple modern Bluetooth manifest section may look like this:

```xml
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
```

If your app also needs to advertise:

```xml
<uses-permission android:name="android.permission.BLUETOOTH_ADVERTISE" />
```

For older Android versions, you may also see legacy permissions:

```xml
<uses-permission
    android:name="android.permission.BLUETOOTH"
    android:maxSdkVersion="30" />

<uses-permission
    android:name="android.permission.BLUETOOTH_ADMIN"
    android:maxSdkVersion="30" />
```

For scanning on Android 11 and lower, location permission may be needed because Bluetooth scanning could be used to infer location. Android’s Bluetooth permission documentation states that `ACCESS_FINE_LOCATION` is necessary on Android 11 and lower for this reason. citeturn659041search5

So a more complete beginner-compatible block may look like:

```xml
<!-- Android 12+ Bluetooth permissions -->
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />

<!-- Legacy Bluetooth permissions for Android 11 and lower -->
<uses-permission
    android:name="android.permission.BLUETOOTH"
    android:maxSdkVersion="30" />

<uses-permission
    android:name="android.permission.BLUETOOTH_ADMIN"
    android:maxSdkVersion="30" />

<!-- Needed for Bluetooth scanning on Android 11 and lower -->
<uses-permission
    android:name="android.permission.ACCESS_FINE_LOCATION"
    android:maxSdkVersion="30" />
```

Do not worry if this looks a lot.

The simple idea is:

```text
New Android:
use BLUETOOTH_SCAN and BLUETOOTH_CONNECT

Older Android:
legacy Bluetooth permissions and sometimes location
```

---

## 5. Network / Wi-Fi permissions

If your device sends data over Wi-Fi or a local network, the app may need internet/network permissions.

The basic manifest permissions are:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

Android’s networking documentation says `INTERNET` and `ACCESS_NETWORK_STATE` are normal permissions, meaning they are granted at install time and do not need runtime permission dialogs. citeturn659041search1

So for normal TCP/HTTP-style communication, the app usually needs:

```text
manifest declaration
```

but not:

```text
runtime permission dialog
```

That is different from Bluetooth permissions.

---

## 6. Add permission state to UI state

In Lesson 12, we already had:

```kotlin
enum class DeviceConnectionState {
    DISCONNECTED,
    CONNECTING,
    CONNECTED,
    ERROR
}
```

Now we can add permission-related state.

For example:

```kotlin
enum class PermissionState {
    UNKNOWN,
    GRANTED,
    DENIED
}
```

Then update `ResearchUiState`:

```kotlin
data class ResearchUiState(
    val patientCode: String = "",
    val sessionName: String = "",
    val currentPatientId: Long? = null,
    val currentSessionId: Long? = null,

    val deviceConnectionState: DeviceConnectionState = DeviceConnectionState.DISCONNECTED,
    val acquisitionState: AcquisitionState = AcquisitionState.IDLE,

    val bluetoothPermissionState: PermissionState = PermissionState.UNKNOWN,

    val measurements: List<MeasurementEntity> = emptyList(),
    val latestValue: Double? = null,

    val isLoading: Boolean = false,
    val message: String = ""
)
```

Now the UI can display:

```text
Bluetooth permission: unknown
Bluetooth permission: granted
Bluetooth permission: denied
```

This is useful because the app can explain why connection is not possible.

---

## 7. What should the UI flow be?

For a real device app, the user flow should be something like:

```text
Open Measurement Screen
 ↓
Check permission
 ↓
If needed, request permission
 ↓
User grants permission
 ↓
Connect Device button becomes available
 ↓
User connects device
 ↓
Start Acquisition becomes available
```

This is better than immediately showing a connection error.

The app should guide the user.

A good research-app flow is:

```text
Permission first
Device connection second
Acquisition third
```

---

## 8. Requesting Bluetooth permissions in Compose

Inside a composable screen, we can create a permission launcher.

For multiple Bluetooth permissions:

```kotlin
val bluetoothPermissionLauncher =
    rememberLauncherForActivityResult(
        contract = ActivityResultContracts.RequestMultiplePermissions()
    ) { permissions ->
        val allGranted = permissions.values.all { it }

        if (allGranted) {
            viewModel.onBluetoothPermissionGranted()
        } else {
            viewModel.onBluetoothPermissionDenied()
        }
    }
```

Required imports:

```kotlin
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
```

The important idea is:

```text
launch permission request
 ↓
receive result
 ↓
update ViewModel state
```

The UI should not store the final app logic itself.

It should report the result to the ViewModel.

---

## 9. Decide which Bluetooth permissions to request

For Android 12+:

```kotlin
val bluetoothPermissions = arrayOf(
    android.Manifest.permission.BLUETOOTH_SCAN,
    android.Manifest.permission.BLUETOOTH_CONNECT
)
```

If you also advertise:

```kotlin
android.Manifest.permission.BLUETOOTH_ADVERTISE
```

For a first research app that only connects to a device and reads data, start with:

```text
SCAN
CONNECT
```

Conceptually:

```text
SCAN
 ↓
find devices

CONNECT
 ↓
connect to selected device
```

---

## 10. Permission request button

In the UI, add a button:

```kotlin
Button(
    onClick = {
        bluetoothPermissionLauncher.launch(bluetoothPermissions)
    }
) {
    Text("Request Bluetooth Permission")
}
```

This is the simplest beginner version.

Later, we can make the app automatically check whether permission is already granted.

But for learning, a visible button is clearer.

---

## 11. ViewModel functions for permission result

In the ViewModel:

```kotlin
fun onBluetoothPermissionGranted() {
    uiState = uiState.copy(
        bluetoothPermissionState = PermissionState.GRANTED,
        message = "Bluetooth permission granted"
    )
}

fun onBluetoothPermissionDenied() {
    uiState = uiState.copy(
        bluetoothPermissionState = PermissionState.DENIED,
        message = "Bluetooth permission denied"
    )
}
```

This keeps the logic consistent:

```text
UI asks for permission
 ↓
system returns result
 ↓
ViewModel updates UI state
```

---

## 12. Only allow connection if permission is granted

Update `connectDevice()`.

Before:

```kotlin
fun connectDevice() {
    uiState = uiState.copy(
        deviceConnectionState = DeviceConnectionState.CONNECTING,
        message = "Connecting to device..."
    )

    viewModelScope.launch {
        delay(1000)

        uiState = uiState.copy(
            deviceConnectionState = DeviceConnectionState.CONNECTED,
            message = "Device connected"
        )
    }
}
```

Now:

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
        delay(1000)

        uiState = uiState.copy(
            deviceConnectionState = DeviceConnectionState.CONNECTED,
            message = "Device connected"
        )
    }
}
```

This is still a simulated connection, but the flow is now realistic.

The app says:

```text
No permission
 ↓
cannot connect

Permission granted
 ↓
can attempt connection
```

---

## 13. Update button enable logic

The Connect button should be enabled only when permission is granted and the device is disconnected:

```kotlin
Button(
    onClick = {
        viewModel.connectDevice()
    },
    enabled =
        uiState.bluetoothPermissionState == PermissionState.GRANTED &&
        uiState.deviceConnectionState == DeviceConnectionState.DISCONNECTED
) {
    Text("Connect Device")
}
```

This is the same state-driven UI idea from Lesson 12.

The UI does not randomly enable all buttons.

It asks:

```text
What state is the app currently in?
Which action is allowed now?
```

---

## 14. Display permission status

In the Measurement Screen, show:

```kotlin
Text(
    text = "Bluetooth permission: ${uiState.bluetoothPermissionState}"
)
```

This may show:

```text
Bluetooth permission: UNKNOWN
Bluetooth permission: GRANTED
Bluetooth permission: DENIED
```

Later, you can convert enum values to nicer display text, like:

```kotlin
fun getPermissionStatusText(
    state: PermissionState
): String {
    return when (state) {
        PermissionState.UNKNOWN -> "Bluetooth permission not requested"
        PermissionState.GRANTED -> "Bluetooth permission granted"
        PermissionState.DENIED -> "Bluetooth permission denied"
    }
}
```

Then:

```kotlin
Text("Bluetooth permission: ${getPermissionStatusText(uiState.bluetoothPermissionState)}")
```

This is more user-friendly.

---

## 15. Full beginner UI permission pattern

Inside `MeasurementScreen`, the permission part could look like this:

```kotlin
@Composable
fun MeasurementScreen(
    uiState: ResearchUiState,
    viewModel: ResearchViewModel
) {
    val bluetoothPermissions = arrayOf(
        android.Manifest.permission.BLUETOOTH_SCAN,
        android.Manifest.permission.BLUETOOTH_CONNECT
    )

    val bluetoothPermissionLauncher =
        rememberLauncherForActivityResult(
            contract = ActivityResultContracts.RequestMultiplePermissions()
        ) { permissions ->
            val allGranted = permissions.values.all { it }

            if (allGranted) {
                viewModel.onBluetoothPermissionGranted()
            } else {
                viewModel.onBluetoothPermissionDenied()
            }
        }

    Column(
        modifier = Modifier.padding(16.dp)
    ) {
        Text("Measurement Screen")

        Spacer(modifier = Modifier.height(16.dp))

        Text(
            text = "Bluetooth permission: ${getPermissionStatusText(uiState.bluetoothPermissionState)}"
        )

        Button(
            onClick = {
                bluetoothPermissionLauncher.launch(bluetoothPermissions)
            }
        ) {
            Text("Request Bluetooth Permission")
        }

        Spacer(modifier = Modifier.height(16.dp))

        Button(
            onClick = {
                viewModel.connectDevice()
            },
            enabled =
                uiState.bluetoothPermissionState == PermissionState.GRANTED &&
                uiState.deviceConnectionState == DeviceConnectionState.DISCONNECTED
        ) {
            Text("Connect Device")
        }
    }
}
```

Important imports:

```kotlin
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Button
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
```

---

## 16. Important limitation of the simple version

The simple version above requests:

```kotlin
BLUETOOTH_SCAN
BLUETOOTH_CONNECT
```

This is appropriate for the Android 12+ permission model.

But if you need to support older Android versions, you may need version-specific permission logic.

For example:

```text
Android 12+
 ↓
BLUETOOTH_SCAN / BLUETOOTH_CONNECT

Android 11 and lower
 ↓
legacy Bluetooth permissions
possibly ACCESS_FINE_LOCATION for scanning
```

For this tutorial path, I would keep development focused on a modern Android tablet first, then add older-device compatibility only if needed.

That keeps the learning manageable.

---

## 17. Where should real device code go?

This is very important.

Do **not** put real Bluetooth code directly inside the Composable UI.

Avoid this:

```text
MeasurementScreen
 └── Bluetooth connection code
```

A better structure is:

```text
MeasurementScreen
 ↓
ResearchViewModel
 ↓
MeasurementRepository
 ↓
BluetoothDataSource
```

or:

```text
MeasurementScreen
 ↓
ResearchViewModel
 ↓
DeviceRepository
 ↓
BluetoothDataSource
```

The screen should only show:

```text
permission status
device status
buttons
latest value
measurement list
```

The real device communication should live deeper in the app.

---

## 18. Introduce a `DeviceDataSource`

Eventually, we can create something like:

```kotlin
interface DeviceDataSource {
    suspend fun connect()
    suspend fun disconnect()
    suspend fun readValue(): Double
}
```

Then a fake version:

```kotlin
class FakeDeviceDataSource : DeviceDataSource {

    override suspend fun connect() {
        delay(1000)
    }

    override suspend fun disconnect() {
        // nothing to do in fake version
    }

    override suspend fun readValue(): Double {
        delay(1000)
        return Random.nextDouble(0.0, 5.0)
    }
}
```

Later, a real Bluetooth version:

```kotlin
class BluetoothDeviceDataSource : DeviceDataSource {

    override suspend fun connect() {
        // real Bluetooth connection later
    }

    override suspend fun disconnect() {
        // real Bluetooth disconnection later
    }

    override suspend fun readValue(): Double {
        // read real incoming value later
        return 0.0
    }
}
```

This is the key design idea.

The ViewModel should not care whether the data source is fake or real.

It only calls:

```kotlin
deviceDataSource.readValue()
```

---

## 19. Replace fake data gradually

Right now, our repository does something like:

```kotlin
val value = Random.nextDouble(0.0, 5.0)
```

A better next structure is:

```text
MeasurementRepository
 ↓
asks DeviceDataSource for value
 ↓
creates MeasurementEntity
 ↓
saves to Room
```

For example:

```kotlin
class MeasurementRepository(
    context: Context,
    private val deviceDataSource: DeviceDataSource = FakeDeviceDataSource()
) {
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
}
```

Now the source of the value is hidden behind:

```kotlin
DeviceDataSource
```

That prepares us for real Bluetooth or Wi-Fi later.

---

## 20. Bluetooth vs Wi-Fi in App Architecture

From an app architecture point of view, Bluetooth and Wi-Fi can be treated similarly.

```text
BluetoothDataSource
 ↓
connect
read bytes
parse value
disconnect

WifiDataSource
 ↓
connect/open socket or HTTP
read message
parse value
disconnect
```

The ViewModel should not know the low-level details.
It should only know:

- device connected or disconnected
- latest value
- measurement saved
- error message

So we can design a common interface:

```kotlin
interface DeviceDataSource {
    suspend fun connect()
    suspend fun disconnect()
    suspend fun readValue(): Double
}
```

Then later choose an implementation:

- `FakeDeviceDataSource`
- `BluetoothDeviceDataSource`
- `WifiDeviceDataSource`
- `UsbDeviceDataSource`

This is why Lesson 13’s Repository layer was important.

## 21. Real Data Is Usually Bytes or Strings

A real device usually does not send a clean Double.
It may send:

```text
"2.438\n"
or:
"S001,2.438,OK\n"
or bytes:
0x02 0x10 0xA4 ...
```

So later we will need a parser.
For example:

```kotlin
fun parseSensorMessage(
    message: String
): Double {
    return message.trim().toDouble()
}
```

Then:

```text
raw device message
 ↓
parser
 ↓
Double value
 ↓
MeasurementEntity
 ↓
Room database
```

This is why we should not mix device code directly with UI code.
## 22. Error Handling for Device Communication

Real devices can fail.
For example:

- permission denied
- device not found
- connection timeout
- device disconnected
- invalid data format
- battery low
- sensor sends corrupted message

So connection code should always update state safely.
For example:

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
            // later: deviceRepository.connect()

            delay(1000)

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

The important structure is:

```text
try to connect
 ↓
if successful, state = CONNECTED
 ↓
if failed, state = ERROR
```

## 23. Updated Research-App Flow

After this lesson, the real future flow becomes:

```text
Patient List
 ↓
Patient Detail
 ↓
Create / select Session
 ↓
Measurement Screen
 ↓
Request Bluetooth permission
 ↓
Connect Device
 ↓
Start Acquisition
 ↓
Read real device data
 ↓
Save MeasurementEntity to Room
 ↓
Stop Acquisition
 ↓
View Result
```

This is the main path toward your research app.

## 24. What You Learned in Lesson 17

The key concepts are:

**Manifest permission** declares what the app may need.

**Runtime permission** asks the user for approval while the app is running.

```kotlin
rememberLauncherForActivityResult(
    contract = ActivityResultContracts.RequestMultiplePermissions()
)
```

is a Compose-friendly way to request multiple permissions.

```kotlin
enum class PermissionState {
    UNKNOWN,
    GRANTED,
    DENIED
}
```

lets the app remember permission status.

```kotlin
interface DeviceDataSource {
    suspend fun connect()
    suspend fun disconnect()
    suspend fun readValue(): Double
}
```

prepares the app for fake, Bluetooth, Wi-Fi, or USB data sources.

The most important mental model is:

> UI should not directly talk to hardware.

```text
UI
 ↓
ViewModel
 ↓
Repository / DeviceDataSource
 ↓
Bluetooth / Wi-Fi / USB / fake data
```

For a research app, this is important because real device communication can fail, permissions can be denied, and raw data may need parsing before it becomes a valid measurement.

## Lesson 18 Preview

In Lesson 18, we should move from permission preparation to the actual data-source abstraction.
We will build the app around:

**DeviceDataSource**

and show how to replace:

```kotlin
Random.nextDouble(0.0, 5.0)
```

with:

```kotlin
deviceDataSource.readValue()
```
Lesson 18 will still use a fake data source first, but it will be structured so that a real Bluetooth or Wi-Fi data source can be inserted later without rewriting the UI.