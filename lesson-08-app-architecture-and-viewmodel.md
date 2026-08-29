# Lesson 8 — App Architecture and ViewModel

So far, we have built everything inside one composable function:

```kotlin
@Composable
fun ResearchScreen() {
    var sampleId by remember { mutableStateOf("") }
    var isConnected by remember { mutableStateOf(false) }

    val measurements = remember {
        mutableStateListOf<Measurement>()
    }

    // UI and logic together
}
```

This is fine for learning, but it becomes messy as the app grows.

In Lesson 8, we separate:

```text
UI code
from
app logic and state
```

using a ViewModel.

Android’s official architecture documentation describes a ViewModel as a business logic or screen-level state holder, and Compose architecture guidance describes the ViewModel as the place that handles UI events and updates state. Android Developers

## 1. Why use a ViewModel?

Right now, ResearchScreen() does too many things:

- stores sample ID

- stores connection state

- stores measurements

- creates fake measurement values

- calculates repetition number

- calculates mean/min/max

- exports CSV

- displays UI

A better structure is:

```text
ResearchScreen
    ↓
shows UI only
```

```text
ResearchViewModel
    ↓
stores state
handles button clicks
creates measurements
calculates statistics
exports CSV text
```

This makes the app easier to read, test, debug, and expand.

## 2. The mental model

Think of the app as two parts:

```text
ViewModel = brain
```

Composable UI = face

The ViewModel stores the state:

```text
sampleId
isConnected
measurements
exportMessage
```

The UI displays that state:

```text
TextField shows sampleId
Text shows device status
Card shows latest value
LazyColumn shows measurement history
```

The UI sends events to the ViewModel:

```text
user types sample ID
user clicks Connect
user clicks Measure
user clicks Clear
user clicks Export
```

The ViewModel updates the state, and Compose redraws the UI.

This is called unidirectional data flow:

- State flows down to the UI
- Events flow up to the ViewModel

Google’s Compose architecture guide explains this pattern as UI events being passed upward to be handled by a state holder such as a ViewModel, which then updates the UI state. Android Developers

## 3. Add the ViewModel dependency

You may already have this if your template included it. In your app-level build.gradle.kts, check dependencies.

You need something like:

```text
implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.10.0")
```

The exact latest version may be different in your project, so it is also fine to use the version Android Studio suggests.

After editing Gradle, click:

Sync Now

You need this dependency so you can get a ViewModel inside Compose using:

```text
viewModel()
```

## 4. Create a UI state data class

Instead of many separate variables, we create one object that represents the whole screen state.

```kotlin
data class ResearchUiState(
    val sampleId: String = "",
    val isConnected: Boolean = false,
    val measurements: List<Measurement> = emptyList(),
    val exportMessage: String = ""
)
```

This is useful because the UI can receive one state object:

```kotlin
ResearchScreenContent(
    uiState = uiState,
    ...
)
```

rather than many separate variables.

## 5. Keep the Measurement data class

We still use:

```kotlin
data class Measurement(
    val sampleId: String,
    val repetition: Int,
    val value: Double,
    val timestamp: Long
)
```

So now we have two data classes:

```kotlin
data class Measurement(...)
data class ResearchUiState(...)
```

Measurement represents one measurement row.

ResearchUiState represents the whole screen state.

## 6. Create ResearchViewModel

Add this class outside your composable functions:

```kotlin
import androidx.lifecycle.ViewModel
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.setValue
import kotlin.random.Random

class ResearchViewModel : ViewModel() {

    var uiState by mutableStateOf(ResearchUiState())
        private set

    fun updateSampleId(newSampleId: String) {
        uiState = uiState.copy(
            sampleId = newSampleId,
            exportMessage = ""
        )
    }

    fun toggleConnection() {
        uiState = uiState.copy(
            isConnected = !uiState.isConnected
        )
    }

    fun addMeasurement() {

        val sampleId = uiState.sampleId

        if (sampleId.isBlank() || !uiState.isConnected) {
            return
        }

        val value = Random.nextDouble(
            from = 0.0,
            until = 5.0
        )

        val repetitionForThisSample =
            uiState.measurements.count {
                it.sampleId == sampleId
            } + 1

        val newMeasurement = Measurement(
            sampleId = sampleId,
            repetition = repetitionForThisSample,
            value = value,
            timestamp = System.currentTimeMillis()
        )

        uiState = uiState.copy(
            measurements = uiState.measurements + newMeasurement,
            exportMessage = ""
        )
    }

    fun clearMeasurements() {
        uiState = uiState.copy(
            measurements = emptyList(),
            exportMessage = ""
        )
    }

    fun setExportMessage(message: String) {
        uiState = uiState.copy(
            exportMessage = message
        )
    }

    fun getCsvText(): String {
        return measurementsToCsv(uiState.measurements)
    }
}
```

This is the “brain” of the screen.

The UI does not need to know how to create a measurement. It only calls:

```kotlin
viewModel.addMeasurement()
```

## 7. Important line: private set

Look at this:

```kotlin
var uiState by mutableStateOf(ResearchUiState())
    private set
```

This means:

The UI can read uiState,

but only the ViewModel can change it.

That is a good pattern.

The UI should not directly do:

```text
viewModel.uiState = ...
```

Instead, it should call clear intention functions:

```kotlin
viewModel.updateSampleId(...)
viewModel.toggleConnection()
viewModel.addMeasurement()
viewModel.clearMeasurements()
```

This keeps the app logic controlled.

## 8. Why use copy()?

When we update the screen state, we write:

```kotlin
uiState = uiState.copy(
    sampleId = newSampleId
)
```

This means:

```text
Create a new ResearchUiState
same as the old one
but with sampleId changed
```

For example, if the old state is:

```text
sampleId = ""
isConnected = false
measurements = []
exportMessage = ""
```

After:

```kotlin
uiState = uiState.copy(sampleId = "S001")
```

the new state becomes:

```text
sampleId = "S001"
isConnected = false
measurements = []
exportMessage = ""
```

Only sampleId changed.

This is why data class is useful.

## 9. Get the ViewModel in Compose

At the top of your file, add:

```kotlin
import androidx.lifecycle.viewmodel.compose.viewModel
```

Then in your MainActivity:

```kotlin
setContent {
    MaterialTheme {
        val viewModel: ResearchViewModel = viewModel()

        ResearchScreen(
            viewModel = viewModel
        )
    }
}
```

Then:

```kotlin
@Composable
fun ResearchScreen(
    viewModel: ResearchViewModel
) {
    val uiState = viewModel.uiState

    ResearchScreenContent(
        uiState = uiState,
        onSampleIdChange = viewModel::updateSampleId,
        onToggleConnection = viewModel::toggleConnection,
        onMeasure = viewModel::addMeasurement,
        onClear = viewModel::clearMeasurements,
        onExportMessage = viewModel::setExportMessage,
        getCsvText = viewModel::getCsvText
    )
}
```

This wrapper connects the UI to the ViewModel.

## 10. Why split ResearchScreen and ResearchScreenContent?

This pattern may look slightly long:

```kotlin
ResearchScreen(...)
ResearchScreenContent(...)
```

But it is useful.

- ResearchScreen
- gets ViewModel

- ResearchScreenContent
- receives plain data and functions

This makes ResearchScreenContent easier to preview and test, because it does not directly depend on the ViewModel.

So:

```kotlin
@Composable
fun ResearchScreen(
    viewModel: ResearchViewModel
) {
    ...
}
```

is the Android/ViewModel-connected layer.

And:

```kotlin
@Composable
fun ResearchScreenContent(
    uiState: ResearchUiState,
    onSampleIdChange: (String) -> Unit,
    onToggleConnection: () -> Unit,
    onMeasure: () -> Unit,
    onClear: () -> Unit,
    onExportMessage: (String) -> Unit,
    getCsvText: () -> String
) {
    ...
}
```

is the mostly pure UI layer.

## 11. Function references: viewModel::addMeasurement

This syntax:

```text
onMeasure = viewModel::addMeasurement
```

means:

Pass the function addMeasurement as a callback.

It is roughly like saying:

```kotlin
onMeasure = {
    viewModel.addMeasurement()
}
```

Both are valid.

So:

viewModel::addMeasurement

is just a shorter way to pass the function.

## 12. Full Lesson 8 app

Here is the full code. Keep your own package name.

        value

```kotlin
package com.example.researchapp

import android.net.Uri
import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.compose.setContent
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.Button
import androidx.compose.material3.Card
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewmodel.compose.viewModel
import kotlin.random.Random

data class Measurement(
    val sampleId: String,
    val repetition: Int,
    val value: Double,
    val timestamp: Long
)

data class ResearchUiState(
    val sampleId: String = "",
    val isConnected: Boolean = false,
    val measurements: List<Measurement> = emptyList(),
    val exportMessage: String = ""
)

fun escapeCsv(value: String): String {
    val needsEscaping =
        value.contains(",") ||
                value.contains("\"") ||
                value.contains("\n")

    return if (needsEscaping) {
        "\"" + value.replace("\"", "\"\"") + "\""
    } else {
    }
}

fun measurementsToCsv(
    measurements: List<Measurement>
): String {

    val header = "sample_id,repetition,value,timestamp"

    val rows = measurements.joinToString(separator = "\n") { measurement ->

        val sampleId = escapeCsv(measurement.sampleId)

        "$sampleId," +
                "${measurement.repetition}," +
                "${measurement.value}," +
                "${measurement.timestamp}"
    }

    return "$header\n$rows"
}

fun safeFilename(text: String): String {
    return text
        .trim()
        .replace(Regex("[^A-Za-z0-9_-]"), "_")
}

class ResearchViewModel : ViewModel() {

    var uiState by mutableStateOf(ResearchUiState())
        private set

    fun updateSampleId(newSampleId: String) {
        uiState = uiState.copy(
            sampleId = newSampleId,
            exportMessage = ""
        )
    }

    fun toggleConnection() {
        uiState = uiState.copy(
            isConnected = !uiState.isConnected
        )
    }

    fun addMeasurement() {

        val sampleId = uiState.sampleId

        if (sampleId.isBlank() || !uiState.isConnected) {
            return
        }

        val value = Random.nextDouble(
            from = 0.0,
            until = 5.0
        )

        val repetitionForThisSample =
            uiState.measurements.count {
                it.sampleId == sampleId
            } + 1

        val newMeasurement = Measurement(
            sampleId = sampleId,
            repetition = repetitionForThisSample,
            value = value,
            timestamp = System.currentTimeMillis()
        )

        uiState = uiState.copy(
            measurements = uiState.measurements + newMeasurement,
            exportMessage = ""
        )
    }

    fun clearMeasurements() {
        uiState = uiState.copy(
            measurements = emptyList(),
            exportMessage = ""
        )
    }

    fun setExportMessage(message: String) {
        uiState = uiState.copy(
            exportMessage = message
        )
    }

    fun getCsvText(): String {
        return measurementsToCsv(uiState.measurements)
    }
}

class MainActivity : ComponentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContent {
            MaterialTheme {
                val viewModel: ResearchViewModel = viewModel()

                ResearchScreen(
                    viewModel = viewModel
                )
            }
        }
    }
}

@Composable
fun ResearchScreen(
    viewModel: ResearchViewModel
) {
    val uiState = viewModel.uiState

    ResearchScreenContent(
        uiState = uiState,
        onSampleIdChange = viewModel::updateSampleId,
        onToggleConnection = viewModel::toggleConnection,
        onMeasure = viewModel::addMeasurement,
        onClear = viewModel::clearMeasurements,
        onExportMessage = viewModel::setExportMessage,
        getCsvText = viewModel::getCsvText
    )
}

@Composable
fun ResearchScreenContent(
    uiState: ResearchUiState,
    onSampleIdChange: (String) -> Unit,
    onToggleConnection: () -> Unit,
    onMeasure: () -> Unit,
    onClear: () -> Unit,
    onExportMessage: (String) -> Unit,
    getCsvText: () -> String
) {
    val context = LocalContext.current

    var pendingCsvText by androidx.compose.runtime.remember {
        mutableStateOf("")
    }

    val createCsvLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.CreateDocument("text/csv"),
        onResult = { uri: Uri? ->

            if (uri != null) {
                context.contentResolver.openOutputStream(uri)?.use { outputStream ->
                    outputStream.write(pendingCsvText.toByteArray())
                }

                onExportMessage("CSV exported successfully.")
            } else {
                onExportMessage("CSV export cancelled.")
            }
        }
    )

    val latestMeasurement = uiState.measurements.lastOrNull()

    val values = uiState.measurements.map {
        it.value
    }

    val meanText = if (values.isNotEmpty()) {
        "%.3f".format(values.average())
    } else {
        "--"
    }

    val minText = values.minOrNull()?.let {
        "%.3f".format(it)
    } ?: "--"

    val maxText = values.maxOrNull()?.let {
        "%.3f".format(it)
    } ?: "--"

    Column(
        modifier = Modifier
            .padding(16.dp)
            .fillMaxWidth(),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        Text(
            text = "Research Measurement App",
            fontSize = 26.sp
        )

        OutlinedTextField(
            value = uiState.sampleId,
            onValueChange = onSampleIdChange,
            label = {
                Text("Sample ID")
            },
            modifier = Modifier.fillMaxWidth()
        )

        if (uiState.sampleId.isBlank()) {
            Text("Please enter a sample ID before measuring.")
        }

        Text(
            text = if (uiState.isConnected) {
                "Device status: Connected"
            } else {
                "Device status: Disconnected"
            }
        )

        Button(
            onClick = onToggleConnection,
            modifier = Modifier.fillMaxWidth()
        ) {
            Text(
                text = if (uiState.isConnected) {
                    "Disconnect"
                } else {
                    "Connect"
                }
            )
        }

        Card(
            modifier = Modifier.fillMaxWidth()
        ) {
            Column(
                modifier = Modifier.padding(16.dp),
                verticalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                Text("Latest value")

                Text(
                    text = latestMeasurement?.value?.let {
                        "%.3f".format(it)
                    } ?: "--",
                    fontSize = 40.sp
                )

                Text("Measurements: ${uiState.measurements.size}")
                Text("Mean: $meanText")
                Text("Min: $minText")
                Text("Max: $maxText")
            }
        }

        Row(
            modifier = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.spacedBy(12.dp)
        ) {
            Button(
                onClick = onMeasure,
                enabled = uiState.sampleId.isNotBlank() && uiState.isConnected,
                modifier = Modifier.weight(1f)
            ) {
                Text("Measure")
            }

            Button(
                onClick = onClear,
                modifier = Modifier.weight(1f)
            ) {
                Text("Clear")
            }
        }

        Button(
            onClick = {

                pendingCsvText = getCsvText()

                val filename = if (uiState.sampleId.isNotBlank()) {
                    "${safeFilename(uiState.sampleId)}_measurements.csv"
                } else {
                    "measurements.csv"
                }

                createCsvLauncher.launch(filename)
            },
            enabled = uiState.measurements.isNotEmpty(),
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("Export CSV")
        }

        if (uiState.exportMessage.isNotBlank()) {
            Text(uiState.exportMessage)
        }

        Text(
            text = "Measurement History",
            fontSize = 20.sp
        )

        LazyColumn(
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            items(uiState.measurements) { measurement ->
                MeasurementRow(measurement)
            }
        }
    }
}

@Composable
fun MeasurementRow(
    measurement: Measurement
) {
    Card(
        modifier = Modifier.fillMaxWidth()
    ) {
        Column(
            modifier = Modifier.padding(12.dp)
        ) {
            Text("Sample: ${measurement.sampleId}")
            Text("Repetition: ${measurement.repetition}")
            Text("Value: ${"%.3f".format(measurement.value)}")
            Text("Timestamp: ${measurement.timestamp}")
        }
    }
}
```

## 13. What changed compared with Lesson 7?

In Lesson 7, the screen directly owned the state:

```kotlin
var sampleId by remember { mutableStateOf("") }
val measurements = remember { mutableStateListOf<Measurement>() }
```

In Lesson 8, the ViewModel owns the state:

```kotlin
class ResearchViewModel : ViewModel() {
    var uiState by mutableStateOf(ResearchUiState())
        private set
}
```

The screen now receives state:

```text
uiState = viewModel.uiState
```

and sends events:

```text
onMeasure = viewModel::addMeasurement
onClear = viewModel::clearMeasurements
```

So the UI becomes less responsible for business logic.

## 14. Important benefit: screen rotation

On Android, rotating the device can recreate the Activity. State stored only inside composables with simple remember can be lost in some situations. A ViewModel is designed to hold screen state and survive configuration changes such as screen rotation. Android Developers

So this architecture is more robust for real Android development.

For your research tablet app, that matters because you do not want to accidentally lose the measurement history because of a screen configuration change.

However, ViewModel is not permanent storage. If the app process is killed, data can still be lost unless you save it to file/database. So for important research data, you still need CSV export or local database saving.

## 15. One limitation of this ViewModel

This ViewModel still uses:

```kotlin
Random.nextDouble(0.0, 5.0)
```

as fake device input.

Later, the ViewModel should call another class such as:

```text
MeasurementRepository
DeviceManager
SensorReader
InferenceEngine
```

For example:

```text
ResearchViewModel
    ↓
MeasurementRepository
    ↓
Bluetooth / USB / sensor / model
```

That is a better long-term structure.

But for now, keeping fake measurement generation inside the ViewModel is acceptable for learning.

## 16. Simple architecture comparison

For a very small app:

Composable only

is acceptable.

For your research app:

Composable + ViewModel

is better.

For a larger app with real hardware and ML:

```text
Composable
    ↓
ViewModel
    ↓
Repository / Manager classes
    ↓
Hardware / file / database / ML model
```

is better still.

## 17. What you should remember from Lesson 8

The most important pattern is this:

```kotlin
data class ResearchUiState(
    val sampleId: String = "",
    val isConnected: Boolean = false,
    val measurements: List<Measurement> = emptyList()
)
class ResearchViewModel : ViewModel() {

    var uiState by mutableStateOf(ResearchUiState())
        private set

    fun updateSampleId(newSampleId: String) {
        uiState = uiState.copy(sampleId = newSampleId)
    }

    fun addMeasurement() {
        // app logic here
    }
}
val viewModel: ResearchViewModel = viewModel()
val uiState = viewModel.uiState
ResearchScreenContent(
    uiState = uiState,
    onSampleIdChange = viewModel::updateSampleId,
    onMeasure = viewModel::addMeasurement
)
```

The key idea:

UI displays state.

UI sends events.

ViewModel handles events.

ViewModel updates state.

Compose redraws UI.

That is the foundation of a clean Compose Android app.

## Small exercise

Add a new function to the ViewModel:

```kotlin
fun removeLastMeasurement() {
    uiState = uiState.copy(
        measurements = uiState.measurements.dropLast(1),
        exportMessage = ""
    )
}
```

Then add a button in the UI:

Undo Last

It should only be enabled when:

```kotlin
uiState.measurements.isNotEmpty()
```

This is a good exercise because it reinforces the event-flow pattern:

```text
Button click
    ↓
onUndoLast()
    ↓
ViewModel removes measurement
    ↓
uiState changes
    ↓
UI updates
```

## Lesson 9 preview

Next, I suggest we cover file/data persistence more properly:

```text
ViewModel is not permanent storage
    ↓
save automatically after each measurement
    ↓
load previous session
    ↓
avoid accidental data loss
```

We can start with simple app-specific storage, then later discuss Room database if needed.
