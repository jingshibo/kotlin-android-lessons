# Lesson 11 — Simulated Live Data Stream

In Lesson 10, we learned the basic idea of **coroutines and background work**.

The most important idea was:

```text
Do not run slow or repeated work directly on the UI thread.
```

This matters because the original tutorial already pointed out that real research apps should not block the UI during tasks such as file saving, ML inference, or long-running acquisition. fileciteturn1file0L447-L450

Now we move to the next practical step.

So far, our app works like this:

```text
User clicks Add Measurement
 ↓
One random measurement is generated
 ↓
Measurement is added to the history
 ↓
Measurements are auto-saved
```

That is useful for learning, but a real research app usually works more like this:

```text
User clicks Start Acquisition
 ↓
App continuously receives data
 ↓
Latest value updates on screen
 ↓
Measurement history grows over time
 ↓
User clicks Stop Acquisition
```

In this lesson, we will **simulate** this behaviour first.

We will not connect the real device yet.

Instead, we generate fake sensor values repeatedly.

This is important because it lets us build the Android app logic before dealing with Bluetooth, Wi-Fi, USB, or hardware debugging.

---

## 1. Why simulate live data first?

In a real research app, the data may come from:

```text
Bluetooth sensor
Wi-Fi device
USB device
tablet internal sensor
external acquisition system
ML model output
```

But if we start with real hardware too early, we may face many problems at once:

```text
Android UI problem
Kotlin syntax problem
Bluetooth permission problem
device connection problem
data parsing problem
saving problem
```

That is too much.

So the better learning path is:

```text
Step 1: Build the app with fake data
Step 2: Make the app logic work
Step 3: Replace fake data with real device data later
```

This is the same logic we used earlier when generating one random measurement.

Now we extend it from:

```text
one fake value
```

to:

```text
continuous fake values
```

---

## 2. What we want to build in Lesson 11

By the end of this lesson, the app should behave like this:

```text
User enters sample ID
 ↓
User clicks Start Acquisition
 ↓
App generates one measurement every second
 ↓
Latest value updates
 ↓
Measurement history grows
 ↓
Statistics update
 ↓
User clicks Stop Acquisition
 ↓
Acquisition stops safely
```

The UI should have:

```text
Sample ID input

Current status:
Recording / Stopped

Latest value:
2.438

Number of measurements:
15

Mean value:
2.512

[ Start Acquisition ]

[ Stop Acquisition ]

Measurement history list
```

This is now much closer to a real data-collection app.

---

## 3. New idea: acquisition state

Before we write the streaming logic, we need to store whether the app is currently recording.

Add this to `ResearchUiState`:

```kotlin
val isAcquiring: Boolean = false
```

So the state may look like this:

```kotlin
data class ResearchUiState(
    val sampleId: String = "",
    val isConnected: Boolean = false,
    val measurements: List<Measurement> = emptyList(),
    val isLoading: Boolean = false,
    val message: String = "",
    val isAcquiring: Boolean = false
)
```

The meaning is simple:

```text
isAcquiring = false
 ↓
the app is not recording

isAcquiring = true
 ↓
the app is currently collecting data
```

This will control which buttons are enabled and whether the data stream is running.

---

## 4. Add latest value

It is also useful to store the latest measurement value separately.

Add:

```kotlin
val latestValue: Double? = null
```

So now:

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

Why nullable?

Because before acquisition starts, there is no latest value yet.

So:

```kotlin
latestValue: Double?
```

means:

```text
There may be a value,
or there may be no value yet.
```

This reuses the null-safety idea from Lesson 3.

In the UI, we can display it safely:

```kotlin
val latestValueText = uiState.latestValue?.toString() ?: "No data yet"
```

---

## 5. The coroutine loop idea

In Lesson 10, we saw this pattern:

```kotlin
viewModelScope.launch {
    // background-capable work
}
```

Now we use a coroutine loop.

The idea is:

```kotlin
viewModelScope.launch {
    while (uiState.isAcquiring) { // This works a bit like interrupt
        // generate measurement
        // update state
        // wait
    }
}
```

But we need to be careful.

A loop without waiting would run extremely fast:

```kotlin
while (uiState.isAcquiring) { 
    generateMeasurement()
}
```

That is bad.

It may create thousands of values very quickly and overload the app.

So we add:

```kotlin
delay(1000)
```

This means:

```text
wait for 1000 milliseconds
```

or:

```text
wait for 1 second
```

So the conceptual loop becomes:

```kotlin
while (uiState.isAcquiring) { 
    generate one measurement
    delay(1000)
}
```

This simulates a sensor sending one value every second.

---

## 6. New import: `delay`

To use `delay`, add:

```kotlin
import kotlinx.coroutines.delay
```

So your coroutine imports may now include:

```kotlin
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
```

---

## 7. Start acquisition function

Inside `ResearchViewModel`, we can add:

```kotlin
fun startAcquisition(context: Context) {
    if (uiState.isAcquiring) {
        return
    }

    uiState = uiState.copy(
        isAcquiring = true,
        message = "Acquisition started"
    )

    viewModelScope.launch {
        while (uiState.isAcquiring) {
            addSimulatedMeasurement(context)
            delay(1000)
        }
    }
}
```

Let us break this down.

First:

```kotlin
if (uiState.isAcquiring) {
    return
}
```

This prevents starting two acquisition loops at the same time.

Without this check, the user could press Start several times and accidentally create multiple loops.

That would cause duplicated data.

Then:

```kotlin
uiState = uiState.copy(
    isAcquiring = true,
    message = "Acquisition started"
)
```

This updates the app state.

Then:

```kotlin
viewModelScope.launch {
    while (uiState.isAcquiring) {
        addSimulatedMeasurement(context)
        delay(1000)
    }
}
```

This starts the repeated measurement loop.

---

## 8. Stop acquisition function

Stopping is simpler:

```kotlin
fun stopAcquisition() {
    uiState = uiState.copy(
        isAcquiring = false,
        message = "Acquisition stopped"
    )
}
```

Why does this stop the loop?

Because the loop condition is:

```kotlin
while (uiState.isAcquiring)
```

When `isAcquiring` becomes `false`, the loop finishes.

The mental model is:

```text
Start button
 ↓
isAcquiring = true
 ↓
loop runs

Stop button
 ↓
isAcquiring = false
 ↓
loop stops
```

---

## 9. Add one simulated measurement

Previously, we had `addMeasurement()`.

Now we can create a more specific function:

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

This function does five things:

```text
generate a fake value
create a Measurement object
add it to the history
update latestValue
auto-save the updated measurement list
```

This is very close to what real acquisition will do later.

Later, instead of this:

```kotlin
val value = Random.nextDouble(0.0, 5.0)
```

we may use something like:

```kotlin
val value = deviceDataSource.readValue()
```

But not yet.

For now, fake data is enough.

---

## 10. Important issue: repeated auto-saving

In this lesson, every generated measurement triggers auto-save:

```kotlin
saveMeasurementsAsync(...)
```

For one value per second, this is acceptable for learning.

But in a real research app, your device may send data much faster:

```text
10 Hz
50 Hz
100 Hz
1000 Hz
```

If you save the whole CSV file after every single sample, that may become inefficient.

Later, we may improve this by:

```text
saving in batches
saving at the end of each session
saving every few seconds
using Room database
using streaming file writing
```

But for Lesson 11, we keep the simple approach.

The goal now is to understand live acquisition logic.

---

## 11. Display acquisition status in the UI

In `ResearchScreen`, we can show:

```kotlin
val acquisitionStatus = if (uiState.isAcquiring) {
    "Recording"
} else {
    "Stopped"
}
```

Then display:

```kotlin
Text("Status: $acquisitionStatus")
```

This gives the user a clear indication of what the app is doing.

For a research app, this is important.

The user should always know whether data is currently being collected.

---

## 12. Display latest value

We can display the latest value safely:

```kotlin
val latestValueText = uiState.latestValue?.let {
    "%.3f".format(it)
} ?: "No data yet"
```

Then:

```kotlin
Text("Latest value: $latestValueText")
```

This means:

```text
If latestValue exists, show it with 3 decimal places.
Otherwise, show "No data yet".
```

Example:

```text
Latest value: 2.438
```

or:

```text
Latest value: No data yet
```

---

## 13. Start and Stop buttons

Now the screen should have two buttons:

```kotlin
Button(
    onClick = {
        viewModel.startAcquisition(context)
    },
    enabled = !uiState.isAcquiring
) {
    Text("Start Acquisition")
}
```

and:

```kotlin
Button(
    onClick = {
        viewModel.stopAcquisition()
    },
    enabled = uiState.isAcquiring
) {
    Text("Stop Acquisition")
}
```

Notice the `enabled` logic.

Start button:

```kotlin
enabled = !uiState.isAcquiring
```

means:

```text
Only allow Start when the app is not already recording.
```

Stop button:

```kotlin
enabled = uiState.isAcquiring
```

means:

```text
Only allow Stop when the app is currently recording.
```

This prevents invalid actions.

---

## 14. Full ViewModel pattern for Lesson 11

Here is the main ViewModel logic.

This builds on Lesson 10.

```kotlin
class ResearchViewModel : ViewModel() {

    var uiState by mutableStateOf(ResearchUiState())
        private set

    fun updateSampleId(newSampleId: String) {
        uiState = uiState.copy(
            sampleId = newSampleId
        )
    }

    fun startAcquisition(context: Context) {
        if (uiState.isAcquiring) {
            return
        }

        uiState = uiState.copy(
            isAcquiring = true,
            message = "Acquisition started"
        )

        viewModelScope.launch {
            while (uiState.isAcquiring) {
                addSimulatedMeasurement(context)
                delay(1000)
            }
        }
    }

    fun stopAcquisition() {
        uiState = uiState.copy(
            isAcquiring = false,
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

You also need these imports:

```kotlin
import android.content.Context
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.setValue
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
import kotlin.random.Random
```

---

## 15. Updated `ResearchUiState`

Your UI state should now include acquisition-related values:

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

The most important additions are:

```kotlin
val isAcquiring: Boolean = false
val latestValue: Double? = null
```

These two values let the UI know:

```text
whether acquisition is running
what the newest value is
```

---

## 16. Updated UI section

Inside `ResearchScreen`, you can show the acquisition state like this:

```kotlin
val context = LocalContext.current

val acquisitionStatus = if (uiState.isAcquiring) {
    "Recording"
} else {
    "Stopped"
}

val latestValueText = uiState.latestValue?.let {
    "%.3f".format(it)
} ?: "No data yet"
```

Then in the UI:

```kotlin
Text("Status: $acquisitionStatus")
Text("Latest value: $latestValueText")
Text("Number of measurements: ${uiState.measurements.size}")
```

For buttons:

```kotlin
Row(
    horizontalArrangement = Arrangement.spacedBy(12.dp)
) {
    Button(
        onClick = {
            viewModel.startAcquisition(context)
        },
        enabled = !uiState.isAcquiring
    ) {
        Text("Start Acquisition")
    }

    Button(
        onClick = {
            viewModel.stopAcquisition()
        },
        enabled = uiState.isAcquiring
    ) {
        Text("Stop Acquisition")
    }
}
```

This gives a basic live acquisition control panel.

---

## 17. Add simple statistics

Because measurements are now generated repeatedly, statistics become more useful.

You can calculate:

```kotlin
val values = uiState.measurements.map {
    it.value
}
```

Then:

```kotlin
val meanText = if (values.isNotEmpty()) {
    "%.3f".format(values.average())
} else {
    "--"
}
```

Display:

```kotlin
Text("Mean value: $meanText")
```

Later, you can also add:

```text
minimum
maximum
standard deviation
signal quality
classification result
```

But for now, mean value is enough.

---

## 18. Important mental model

This lesson introduces a very important structure:

```text
Start Acquisition
 ↓
set isAcquiring = true
 ↓
start coroutine loop
 ↓
generate value
 ↓
append measurement
 ↓
update UI
 ↓
delay
 ↓
repeat until stopped
```

The stop logic is:

```text
Stop Acquisition
 ↓
set isAcquiring = false
 ↓
loop condition becomes false
 ↓
coroutine finishes
```

This is the first time our app behaves like a real acquisition system.

---

## 19. Why this prepares us for real devices

Right now, we use:

```kotlin
val value = Random.nextDouble(0.0, 5.0)
```

Later, we can replace it with real data input.

For example, conceptually:

```kotlin
val value = bluetoothDataSource.readValue()
```

or:

```kotlin
val value = wifiDataSource.readValue()
```

or:

```kotlin
val value = sensorParser.parse(rawMessage)
```

The rest of the app does not need to change much.

That is the key point.

The UI does not care whether the value came from:

```text
random generator
Bluetooth
Wi-Fi
USB
internal sensor
ML model
```

The UI only cares that the ViewModel provides:

```text
latestValue
measurements
isAcquiring
message
```

This is a very important app-design idea.

---

## 20. What you learned in Lesson 11

The key new patterns are:

```kotlin
val isAcquiring: Boolean = false
```

to represent whether the app is recording.

```kotlin
val latestValue: Double? = null
```

to represent the newest available measurement.

```kotlin
viewModelScope.launch {
    while (uiState.isAcquiring) {
        addSimulatedMeasurement(context)
        delay(1000)
    }
}
```

to simulate a live data stream.

```kotlin
Button(
    enabled = !uiState.isAcquiring
)
```

to prevent invalid button actions.

The most important mental model is:

```text
Do not connect real hardware first.
First build the app with simulated live data.
Then replace the fake data source later.
```

For a research app, this is very useful because it lets you develop:

```text
UI
state management
measurement history
auto-save
statistics
start/stop acquisition flow
```

before dealing with real device communication.

---

# Lesson 12 preview

In Lesson 12, we should improve the app state model.

Right now, we only have:

```kotlin
isAcquiring: Boolean
```

But a real app has more states:

```text
Disconnected
Connected
Ready
Recording
Stopped
Error
```

So Lesson 12 should cover:

```text
Device connection state
Acquisition state
Start/stop logic
Preventing invalid actions
Showing clear status messages
```

That will make the app behave more like a real research measurement system.
