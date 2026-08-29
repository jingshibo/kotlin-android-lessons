# Lesson 12 — Device Connection State and Acquisition Flow

In Lesson 11, we changed the app from:

```text
User clicks once
 ↓
One fake measurement is created
```

to:

```text
User clicks Start Acquisition
 ↓
Fake measurements are generated repeatedly
 ↓
User clicks Stop Acquisition
 ↓
Acquisition stops
```

That was a big step.

But our state model was still too simple.

We only used:

```kotlin
val isAcquiring: Boolean = false
```

This works for a beginner prototype, but a real research app needs clearer states.

For example, the app may be:

```text
device not connected
device connected
ready to record
currently recording
recording stopped
error happened
```

So in this lesson, we improve the app by introducing:

```text
Device connection state
Acquisition state
Start/stop rules
Clear UI status messages
```

---

## 1. Why `isAcquiring` is not enough

In Lesson 11, we had:

```kotlin
val isAcquiring: Boolean = false
```

This only answers one question:

```text
Is the app currently acquiring data?
```

But it does not answer:

```text
Is the device connected?
Is the app ready?
Did an error happen?
Can the user press Start?
Can the user press Stop?
Should the app show “Connect Device” first?
```

For example, imagine the user presses **Start Acquisition** before the device is connected.

With only `isAcquiring`, our app does not clearly represent this situation.

So we need a better state model.

---

## 2. Two types of state

For our research app, it is useful to separate:

```text
Device connection state
```

from:

```text
Acquisition state
```

They are related, but not the same.

For example:

```text
Device connected
```

does not necessarily mean:

```text
Recording
```

The device may be connected but waiting.

So we can think like this:

```text
Device state
 ↓
What is the connection status?

Acquisition state
 ↓
What is the recording status?
```

---

## 3. Device connection state

Let us create an enum:

```kotlin
enum class DeviceConnectionState {
    DISCONNECTED,
    CONNECTING,
    CONNECTED,
    ERROR
}
```

This means the device can be in one of four states.

```text
DISCONNECTED
 ↓
No device is connected.

CONNECTING
 ↓
The app is trying to connect.

CONNECTED
 ↓
The device is ready.

ERROR
 ↓
Something went wrong.
```

This is better than using strings such as:

```kotlin
val deviceState = "CONNECTED"
```

because strings can easily contain mistakes:

```kotlin
val deviceState = "CONECTED" // typo
```

An enum is safer.

This connects back to Lesson 3, where we learned that `enum class` is useful for fixed states.

---

## 4. Acquisition state

Now create another enum:

```kotlin
enum class AcquisitionState {
    IDLE,
    RECORDING,
    STOPPED,
    ERROR
}
```

This represents the measurement process.

```text
IDLE
 ↓
The app has not started recording yet.

RECORDING
 ↓
The app is currently collecting measurements.

STOPPED
 ↓
The app recorded data, but acquisition has now stopped.

ERROR
 ↓
Something went wrong during acquisition.
```

Now the app has a clearer model:

```text
DeviceConnectionState
 ↓
connection condition

AcquisitionState
 ↓
recording condition
```

---

## 5. Update `ResearchUiState`

In Lesson 11, our UI state looked like this:

```kotlin
data class ResearchUiState(
    val sampleId: String = "",
    val isConnected: Boolean = false,
    val measurements: List<Measurement> = emptyList(),
    val isLoading: Boolean = false,
    val message: String = "",
    val isAcquiring: Boolean = false,
    val latestValue: Double? = null
)
```

Now we replace:

```kotlin
val isConnected: Boolean = false
val isAcquiring: Boolean = false
```

with clearer enum states:

```kotlin
val deviceConnectionState: DeviceConnectionState = DeviceConnectionState.DISCONNECTED
val acquisitionState: AcquisitionState = AcquisitionState.IDLE
```

So the improved version becomes:

```kotlin
data class ResearchUiState(
    val sampleId: String = "",
    val deviceConnectionState: DeviceConnectionState = DeviceConnectionState.DISCONNECTED,
    val acquisitionState: AcquisitionState = AcquisitionState.IDLE,
    val measurements: List<Measurement> = emptyList(),
    val latestValue: Double? = null,
    val isLoading: Boolean = false,
    val message: String = ""
)
```

This is more expressive.

Instead of only knowing:

```text
isAcquiring = true or false
```

we now know:

```text
deviceConnectionState = CONNECTED
acquisitionState = RECORDING
```

That is much clearer.

---

## 6. Create readable status messages

Enums are good for app logic, but the user should not see raw enum names like:

```text
DISCONNECTED
RECORDING
```

Instead, we can convert them into readable messages.

For device state:

```kotlin
fun getDeviceStatusText(
    state: DeviceConnectionState
): String {
    return when (state) {
        DeviceConnectionState.DISCONNECTED -> "Device disconnected"
        DeviceConnectionState.CONNECTING -> "Connecting to device..."
        DeviceConnectionState.CONNECTED -> "Device connected"
        DeviceConnectionState.ERROR -> "Device error"
    }
}
```

For acquisition state:

```kotlin
fun getAcquisitionStatusText(
    state: AcquisitionState
): String {
    return when (state) {
        AcquisitionState.IDLE -> "Ready to start"
        AcquisitionState.RECORDING -> "Recording"
        AcquisitionState.STOPPED -> "Recording stopped"
        AcquisitionState.ERROR -> "Acquisition error"
    }
}
```

The important pattern is:

```kotlin
when (state) {
    ...
}
```

Because we are using an enum, Kotlin knows all possible states.

---

## 7. Simulated device connection

We still do not connect a real device yet.

Instead, we simulate connection.

This follows the same idea as Lesson 11:

```text
First build the app logic with fake behaviour.
Later replace fake behaviour with real Bluetooth/Wi-Fi/USB code.
```

Add this function to the ViewModel:

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

This simulates:

```text
User clicks Connect Device
 ↓
App shows Connecting...
 ↓
After 1 second, app shows Device connected
```

Later, this fake `delay(1000)` can be replaced by real connection logic.

---

## 8. Simulated device disconnection

We also need a disconnect function:

```kotlin
fun disconnectDevice() {
    if (uiState.acquisitionState == AcquisitionState.RECORDING) {
        stopAcquisition()
    }

    uiState = uiState.copy(
        deviceConnectionState = DeviceConnectionState.DISCONNECTED,
        acquisitionState = AcquisitionState.IDLE,
        message = "Device disconnected"
    )
}
```

Notice this part:

```kotlin
if (uiState.acquisitionState == AcquisitionState.RECORDING) {
    stopAcquisition()
}
```

This means:

```text
If the app is recording,
stop recording before disconnecting.
```

That is important.

A real research app should not continue acquisition after the device is disconnected.

---

## 9. Start acquisition rule

Now we should not allow acquisition unless the device is connected.

So `startAcquisition()` should check:

```kotlin
if (uiState.deviceConnectionState != DeviceConnectionState.CONNECTED) {
    uiState = uiState.copy(
        message = "Connect device before starting acquisition"
    )
    return
}
```

Full version:

```kotlin
fun startAcquisition(context: Context) {
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
            addSimulatedMeasurement(context)
            delay(1000)
        }
    }
}
```

This now has two safety checks:

```text
Check 1:
Is the device connected?

Check 2:
Is acquisition already running?
```

This prevents invalid app behaviour.

---

## 10. Stop acquisition rule

The stop function becomes:

```kotlin
fun stopAcquisition() {
    if (uiState.acquisitionState != AcquisitionState.RECORDING) {
        return
    }

    uiState = uiState.copy(
        acquisitionState = AcquisitionState.STOPPED,
        message = "Acquisition stopped"
    )
}
```

This means:

```text
Only stop if currently recording.
```

If the app is already idle or stopped, pressing Stop should do nothing.

---

## 11. Add simulated measurement

This function is similar to Lesson 11:

```kotlin
private fun addSimulatedMeasurement(context: Context) {
    val value = Random.nextDouble(0.0, 5.0)

    val newMeasurement = Measurement(
        sampleId = uiState.sampleId,
        repetition = uiState.measurements.size + 1,
        value = value,
        timestamp = System.currentTimeMillis(),
        status = "OK"
    )

    val updatedMeasurements = uiState.measurements + newMeasurement

    uiState = uiState.copy(
        measurements = updatedMeasurements,
        latestValue = value
    )

    saveMeasurementsAsync(
        context = context,
        measurements = updatedMeasurements
    )
}
```

The logic is still:

```text
Generate fake value
 ↓
Create Measurement object
 ↓
Add it to history
 ↓
Update latest value
 ↓
Auto-save
```

Later, only this part needs to change:

```kotlin
val value = Random.nextDouble(0.0, 5.0)
```

For a real device, it may become something like:

```kotlin
val value = deviceDataSource.readValue()
```

But for now, simulation is good.

---

## 12. Button logic in the UI

Now the UI can use the state to decide which buttons should be enabled.

### Connect button

```kotlin
Button(
    onClick = {
        viewModel.connectDevice()
    },
    enabled = uiState.deviceConnectionState == DeviceConnectionState.DISCONNECTED
) {
    Text("Connect Device")
}
```

This means:

```text
Only allow Connect when currently disconnected.
```

### Disconnect button

```kotlin
Button(
    onClick = {
        viewModel.disconnectDevice()
    },
    enabled = uiState.deviceConnectionState == DeviceConnectionState.CONNECTED
) {
    Text("Disconnect Device")
}
```

This means:

```text
Only allow Disconnect when connected.
```

### Start button

```kotlin
Button(
    onClick = {
        viewModel.startAcquisition(context)
    },
    enabled =
        uiState.deviceConnectionState == DeviceConnectionState.CONNECTED &&
        uiState.acquisitionState != AcquisitionState.RECORDING
) {
    Text("Start Acquisition")
}
```

This means:

```text
Only allow Start when:
device is connected
and
app is not already recording
```

### Stop button

```kotlin
Button(
    onClick = {
        viewModel.stopAcquisition()
    },
    enabled = uiState.acquisitionState == AcquisitionState.RECORDING
) {
    Text("Stop Acquisition")
}
```

This means:

```text
Only allow Stop when recording.
```

This is much better than allowing every button all the time.

---

## 13. Display state in the UI

Inside `ResearchScreen`, add:

```kotlin
val deviceStatusText = getDeviceStatusText(
    uiState.deviceConnectionState
)

val acquisitionStatusText = getAcquisitionStatusText(
    uiState.acquisitionState
)

val latestValueText = uiState.latestValue?.let {
    "%.3f".format(it)
} ?: "No data yet"
```

Then display:

```kotlin
Text("Device status: $deviceStatusText")
Text("Acquisition status: $acquisitionStatusText")
Text("Latest value: $latestValueText")
Text("Measurements: ${uiState.measurements.size}")
```

This gives the user a clear view:

```text
Device status: Device connected
Acquisition status: Recording
Latest value: 2.438
Measurements: 12
```

For a research app, this is important because the user must always know:

```text
Is the device connected?
Is recording active?
How many measurements have been collected?
What is the latest value?
```

---

## 14. Full state definitions

Here are the key data structures for this lesson:

```kotlin
enum class DeviceConnectionState {
    DISCONNECTED,
    CONNECTING,
    CONNECTED,
    ERROR
}

enum class AcquisitionState {
    IDLE,
    RECORDING,
    STOPPED,
    ERROR
}

data class ResearchUiState(
    val sampleId: String = "",
    val deviceConnectionState: DeviceConnectionState = DeviceConnectionState.DISCONNECTED,
    val acquisitionState: AcquisitionState = AcquisitionState.IDLE,
    val measurements: List<Measurement> = emptyList(),
    val latestValue: Double? = null,
    val isLoading: Boolean = false,
    val message: String = ""
)
```

You may also still have:

```kotlin
data class Measurement(
    val sampleId: String,
    val repetition: Int,
    val value: Double,
    val timestamp: Long,
    val status: String
)
```

---

## 15. Full ViewModel pattern for Lesson 12

This is the main ViewModel logic.

```kotlin
class ResearchViewModel : ViewModel() {

    var uiState by mutableStateOf(ResearchUiState())
        private set

    fun updateSampleId(newSampleId: String) {
        uiState = uiState.copy(
            sampleId = newSampleId
        )
    }

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

    fun disconnectDevice() {
        if (uiState.acquisitionState == AcquisitionState.RECORDING) {
            stopAcquisition()
        }

        uiState = uiState.copy(
            deviceConnectionState = DeviceConnectionState.DISCONNECTED,
            acquisitionState = AcquisitionState.IDLE,
            message = "Device disconnected"
        )
    }

    fun startAcquisition(context: Context) {
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
                addSimulatedMeasurement(context)
                delay(1000)
            }
        }
    }

    fun stopAcquisition() {
        if (uiState.acquisitionState != AcquisitionState.RECORDING) {
            return
        }

        uiState = uiState.copy(
            acquisitionState = AcquisitionState.STOPPED,
            message = "Acquisition stopped"
        )
    }

    private fun addSimulatedMeasurement(context: Context) {
        val value = Random.nextDouble(0.0, 5.0)

        val newMeasurement = Measurement(
            sampleId = uiState.sampleId,
            repetition = uiState.measurements.size + 1,
            value = value,
            timestamp = System.currentTimeMillis(),
            status = "OK"
        )

        val updatedMeasurements = uiState.measurements + newMeasurement

        uiState = uiState.copy(
            measurements = updatedMeasurements,
            latestValue = value
        )

        saveMeasurementsAsync(
            context = context,
            measurements = updatedMeasurements
        )
    }

    private fun saveMeasurementsAsync(
        context: Context,
        measurements: List<Measurement>
    ) {
        viewModelScope.launch {
            try {
                withContext(Dispatchers.IO) {
                    saveMeasurementsToInternalStorage(
                        context = context,
                        measurements = measurements
                    )
                }

                uiState = uiState.copy(
                    message = "Measurements auto-saved"
                )
            } catch (e: Exception) {
                uiState = uiState.copy(
                    message = "Auto-save failed"
                )
            }
        }
    }
}
```

The key idea is not every line of code.

The key idea is this structure:

```text
UI button
 ↓
ViewModel function
 ↓
check current state
 ↓
allow or reject the action
 ↓
update UI state
 ↓
perform background work if needed
```

---

## 16. Full UI pattern for the control buttons

Inside `ResearchScreen`, your button area can look like this:

```kotlin
Row(
    horizontalArrangement = Arrangement.spacedBy(12.dp)
) {
    Button(
        onClick = {
            viewModel.connectDevice()
        },
        enabled = uiState.deviceConnectionState == DeviceConnectionState.DISCONNECTED
    ) {
        Text("Connect")
    }

    Button(
        onClick = {
            viewModel.disconnectDevice()
        },
        enabled = uiState.deviceConnectionState == DeviceConnectionState.CONNECTED
    ) {
        Text("Disconnect")
    }
}

Spacer(modifier = Modifier.height(12.dp))

Row(
    horizontalArrangement = Arrangement.spacedBy(12.dp)
) {
    Button(
        onClick = {
            viewModel.startAcquisition(context)
        },
        enabled =
            uiState.deviceConnectionState == DeviceConnectionState.CONNECTED &&
            uiState.acquisitionState != AcquisitionState.RECORDING
    ) {
        Text("Start Acquisition")
    }

    Button(
        onClick = {
            viewModel.stopAcquisition()
        },
        enabled = uiState.acquisitionState == AcquisitionState.RECORDING
    ) {
        Text("Stop Acquisition")
    }
}
```

This gives the app a more professional control flow.

The user cannot start acquisition before connecting the device.

The user cannot press Stop unless acquisition is active.

---

## 17. What this teaches you

This lesson is not mainly about making more buttons.

It teaches a very important Android app design idea:

```text
App behaviour should be controlled by state.
```

Instead of thinking:

```text
What should this button do?
```

you should think:

```text
What state is the app currently in?
What actions are allowed in this state?
What state should come next?
```

For example:

```text
DISCONNECTED
 ↓
Allowed action:
Connect

CONNECTED
 ↓
Allowed actions:
Start Acquisition
Disconnect

RECORDING
 ↓
Allowed action:
Stop Acquisition

ERROR
 ↓
Allowed action:
Reset or reconnect
```

This is much closer to how a real research data-collection app should behave.

---

## 18. Why this matters for research apps

In research data collection, unclear app state can cause real problems.

For example:

```text
The user thinks the device is recording, but it is not.
The user starts recording before entering the correct sample ID.
The device disconnects but the app keeps showing old values.
The app saves data under the wrong session.
The user presses Start multiple times and duplicates measurements.
```

A clear state model reduces these risks.

This is especially important when collecting experimental data, because the app should support reliable and repeatable procedures.

---

## 19. What you learned in Lesson 12

The key new patterns are:

```kotlin
enum class DeviceConnectionState {
    DISCONNECTED,
    CONNECTING,
    CONNECTED,
    ERROR
}
```

for device connection status.

```kotlin
enum class AcquisitionState {
    IDLE,
    RECORDING,
    STOPPED,
    ERROR
}
```

for recording status.

```kotlin
if (uiState.deviceConnectionState != DeviceConnectionState.CONNECTED) {
    return
}
```

for preventing invalid actions.

```kotlin
enabled =
    uiState.deviceConnectionState == DeviceConnectionState.CONNECTED &&
    uiState.acquisitionState != AcquisitionState.RECORDING
```

for controlling UI buttons from state.

The most important mental model is:

```text
Clear state
 ↓
clear allowed actions
 ↓
safer research data collection
```

## Lesson 13 Preview

In Lesson 13, we should improve the structure again.

Right now, the ViewModel still does many things:

- controls UI state
- simulates device connection
- generates fake measurements
- calls save functions
- handles acquisition flow

This is okay for learning, but as the app grows, the ViewModel will become too large.

So Lesson 13 should introduce the:

**Repository layer**

The goal will be to separate:

```text
UI state and app flow
```

from:

```text
data saving/loading
fake device data
real device data later
```

This prepares us for:

- Room database
- real Bluetooth/Wi-Fi data source
- signal processing
- ML inference

The next lesson will make the app architecture cleaner.
