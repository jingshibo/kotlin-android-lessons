# Lesson 19 — Permissions and Real Device Communication Overview

In Lesson 18, we moved from one large screen to a multi-screen research app structure:

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

### Permission belongs to the app, not to one Bluetooth device

The Bluetooth permission is granted to your installed app on the Android phone or tablet.

It is **not** granted separately for every sensor or Bluetooth device.

For example:

```text
User grants this app permission to use nearby Bluetooth devices
 ↓
App may scan for or connect to devices, depending on the granted permissions
 ↓
App does not request the same permission again for every connection
```

The app should still check the current permission before every protected Bluetooth operation. It should open the permission request only when the required permission is missing.

This matters because permission can later be removed when:

- the user revokes it in Android Settings
- Android resets permissions for an app that has not been used for a long time
- the app is reinstalled
- the app's data is cleared

So the practical rule is:

```text
Check every time
Request only when permission is missing
```

Bluetooth permission and Bluetooth pairing are different:

- **App permission** allows the app to use Bluetooth and is normally requested once.
- **Pairing or bonding** identifies and trusts a particular remote device. A PIN or confirmation dialog may appear the first time each new device is paired.
- A previously paired device normally reconnects without another pairing dialog.
- Some Bluetooth Low Energy devices do not require pairing at all.

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

In Lesson 14, we already had:

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
Check whether permission is already granted
 ↓
If missing, request permission
 ↓
User grants permission
 ↓
Connect Device button becomes available
 ↓
User selects and connects a device
 ↓
If that device requires pairing, Android may show a separate pairing dialog
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

Your code must choose the runtime permissions for the Android version currently running. Android does not add this version check to your app automatically.

For an app that scans for and connects to devices, a beginner-compatible version is:

```kotlin
val bluetoothPermissions =
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        arrayOf(
            Manifest.permission.BLUETOOTH_SCAN,
            Manifest.permission.BLUETOOTH_CONNECT
        )
    } else {
        arrayOf(
            Manifest.permission.ACCESS_FINE_LOCATION
        )
    }
```

`Build.VERSION_CODES.S` means Android 12, API level 31. Android evaluates the condition at runtime and uses the appropriate branch.

On Android 11 and lower, the legacy `BLUETOOTH` and `BLUETOOTH_ADMIN` permissions are declared in the manifest. They are normal install-time permissions, so they are not added to this runtime permission launcher. Location is included here because scanning on those Android versions can require runtime location permission.

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

Before launching the permission dialog, check whether every required permission is already granted:

```kotlin
val context = LocalContext.current

fun areBluetoothPermissionsGranted(): Boolean {
    return bluetoothPermissions.all { permission ->
        ContextCompat.checkSelfPermission(
            context,
            permission
        ) == PackageManager.PERMISSION_GRANTED
    }
}

Button(
    onClick = {
        if (areBluetoothPermissionsGranted()) {
            // Permission was granted earlier, so no system dialog is needed.
            viewModel.onBluetoothPermissionGranted()
        } else {
            bluetoothPermissionLauncher.launch(bluetoothPermissions)
        }
    }
) {
    Text("Check / Request Bluetooth Permission")
}
```

The permission remains attached to this installed app, not to one selected Bluetooth device. Connecting to Sensor A and then Sensor B does not normally require the app permission dialog twice.

However, pairing is separate. Android may show a pairing confirmation or PIN dialog the first time the app connects to each device that requires pairing.

For learning, a visible check/request button is clearer. A later version could perform the same check automatically when the screen opens or resumes.

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

This permission check does not approve one particular sensor. It confirms that the app currently has permission to perform the Bluetooth operation.

When real Bluetooth code is added, keep the two flows separate:

```text
App permission missing
 ↓
show Android permission request

App permission granted, but selected device is not paired
 ↓
Android may show device pairing confirmation or PIN

App permission granted and device already paired
 ↓
connect normally without showing either dialog again
```

The app should check the real system permission before connecting, even if the ViewModel previously stored `PermissionState.GRANTED`, because the user can revoke permission while the app is not active.

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

This is the same state-driven UI idea from Lesson 14.

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
    val context = LocalContext.current

    val bluetoothPermissions =
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            arrayOf(
                Manifest.permission.BLUETOOTH_SCAN,
                Manifest.permission.BLUETOOTH_CONNECT
            )
        } else {
            arrayOf(
                Manifest.permission.ACCESS_FINE_LOCATION
            )
        }

    fun areBluetoothPermissionsGranted(): Boolean {
        return bluetoothPermissions.all { permission ->
            ContextCompat.checkSelfPermission(
                context,
                permission
            ) == PackageManager.PERMISSION_GRANTED
        }
    }

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
                if (areBluetoothPermissionsGranted()) {
                    viewModel.onBluetoothPermissionGranted()
                } else {
                    bluetoothPermissionLauncher.launch(bluetoothPermissions)
                }
            }
        ) {
            Text("Check / Request Bluetooth Permission")
        }

        Spacer(modifier = Modifier.height(16.dp))

        Button(
            onClick = {
                if (areBluetoothPermissionsGranted()) {
                    viewModel.connectDevice()
                } else {
                    // Permission may have been revoked after it was first granted.
                    viewModel.onBluetoothPermissionDenied()
                }
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
import androidx.core.content.ContextCompat
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Button
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import android.Manifest
import android.content.pm.PackageManager
import android.os.Build
```

---

## 16. Important limitation of the simple version

The Android 12+ branch requests:

```kotlin
BLUETOOTH_SCAN
BLUETOOTH_CONNECT
```

This is appropriate for the Android 12+ permission model. The version check in the full example also selects `ACCESS_FINE_LOCATION` when the app runs on Android 11 or lower.

This version-specific logic matters only when the app supports those older Android versions:

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

If the research app will run only on a known modern Android tablet, you can keep development focused on that device first. Keep the older-device branch when the app must support both generations.

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

This is why Lesson 15’s Repository layer was important.

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

## 24. What You Learned in Lesson 19

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

## Lesson 20 Preview

In Lesson 20, we should move from permission preparation to the actual data-source abstraction.
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
Lesson 20 will still use a fake data source first, but it will be structured so that a real Bluetooth or Wi-Fi data source can be inserted later without rewriting the UI.
