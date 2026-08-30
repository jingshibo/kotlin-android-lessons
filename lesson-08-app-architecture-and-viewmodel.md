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

Android’s official architecture documentation describes a ViewModel as a business logic or screen-level state holder, and Compose architecture guidance describes the ViewModel as the place that handles UI events and updates state. 

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

Google’s Compose architecture guide explains this pattern as UI events being passed upward to be handled by a state holder such as a ViewModel, which then updates the UI state. 

## 3. Add the ViewModel dependency

You may already have this if your template included it.

Open your app-level Gradle file:

```text
app/build.gradle.kts
```

Be careful: Android projects often have more than one Gradle file.

You want the one inside the `app` module, not the project-level Gradle file.

Inside `app/build.gradle.kts`, find the `dependencies` block:

```kotlin
dependencies {
    ...
}
```

Add the ViewModel Compose dependency inside that block:

```kotlin
dependencies {
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.10.0")
}
```

If the block already has other dependencies, just add this as one more line:

```kotlin
dependencies {
    implementation("androidx.core:core-ktx:...")
    implementation("androidx.activity:activity-compose:...")
    implementation("androidx.compose.material3:material3:...")

    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.10.0")
}
```

The exact version may be different in your project.

That is okay. You can use the version Android Studio suggests.

After editing the Gradle file, click:

```text
Sync Now
```

Android Studio needs to sync Gradle before your Kotlin file can import:

```kotlin
import androidx.lifecycle.viewmodel.compose.viewModel
```

You need this dependency so you can get a ViewModel inside Compose using:

```kotlin
viewModel()
```

If Android Studio shows an error for:

```kotlin
import androidx.lifecycle.viewmodel.compose.viewModel
```

then either:

```text
the dependency is missing
or Gradle has not synced yet
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

Add this class outside your composable functions. It may reads a bit complex. We will explain them in detail in below sections:

```kotlin
import androidx.lifecycle.ViewModel
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.setValue
import kotlin.random.Random

class ResearchViewModel : ViewModel() { // ResearchViewModel inherits from ViewModel.
    
    // Create a ResearchUiState object. uiState is var, so the ViewModel can replace the whole ResearchUiState object.
    // The properties inside ResearchUiState are val, so we use copy() to create a new object with one field changed.
    // The by keyword lets us use uiState like a normal variable instead of writing uiState.value.
    var uiState by mutableStateOf(ResearchUiState()) // There is no "by remember" here because this state lives inside the ViewModel, not inside a composable function. It does not need to remember the state and prevent reset during recomposition.

        // Protection: Everyone see uiState but only the ViewModel can modify uiState
        // UI reads uiState, UI calls ViewModel functions, ViewModel updates uiState
        private set 

    fun updateSampleId(newSampleId: String) {
        uiState = uiState.copy(  // ResearchUiState properties are val, so we cannot mutate one property directly. However, since uiState itself is var, we can reassign it with a new uiState value. // Therefore, we reassign it by creating a new state object via copying, and replacing only the sampleId field.
            sampleId = newSampleId, // Because uiState is created with mutableStateOf, assigning a new value tells Compose that state changed.
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
            measurements = uiState.measurements + newMeasurement, // returns a brand-new list (original + one element appended) rather than mutating the existing list in place.
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

    // This belongs in the ViewModel because it reads the current uiState.
    // The actual CSV conversion is a pure helper function outside the ViewModel.
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
// Protection: Everyone see uiState but only the ViewModel can modify uiState
// UI reads uiState, UI calls ViewModel functions, ViewModel updates uiState
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

`ResearchUiState` represents the current screen state.

You can think of it as one snapshot of the screen:

```kotlin
data class ResearchUiState(
    val sampleId: String = "",
    val isConnected: Boolean = false,
    val measurements: List<Measurement> = emptyList(),
    val exportMessage: String = ""
)
```

Because these properties use `val`, an existing `ResearchUiState` object cannot be edited directly.

So this is not allowed:

```kotlin
uiState.sampleId = "S001"
```

That is intentional.

Instead of editing the old state object, we create a new state object.

For example, when the user types a new sample ID, we want the next state to be:

```text
same as the old state,
but with a different sampleId
```

One way to do that is to manually create a new `ResearchUiState`:

```kotlin
uiState = ResearchUiState(
    sampleId = newSampleId,
    isConnected = uiState.isConnected,
    measurements = uiState.measurements,
    exportMessage = uiState.exportMessage
)
```

This works, but it is long and easy to get wrong.

Because `ResearchUiState` is a data class, Kotlin gives it a `copy()` function.

`copy()` lets us create a new object from the old object:

```kotlin
uiState = uiState.copy(
    sampleId = newSampleId
)
```

This means:

```text
Start from the current uiState.
Copy all its existing values.
Change only sampleId.
Return a new ResearchUiState object.
Assign that new object back to uiState.
```

So `copy()` is useful because it lets us update one field without rewriting every field.

For example, if the old state is:

```text
sampleId = ""
isConnected = false
measurements = []
exportMessage = ""
```

After this:

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

Only `sampleId` changed.

The other values were copied from the old state.

### Why is uiState var but its properties are val?

This line can feel confusing. uiState itself is var type but its properties are val type. What does this mean:

```kotlin
var uiState by mutableStateOf(ResearchUiState())
```

The important detail is that there are two levels:

```text
uiState itself is var
    -> the ViewModel can replace the whole ResearchUiState object

properties inside ResearchUiState are val
    -> the old ResearchUiState object cannot be edited directly
```

So this is allowed:

```kotlin
uiState = uiState.copy(sampleId = "S001")
```

because we are replacing the whole state object.

This is not allowed:

```kotlin
uiState.sampleId = "S001"
```

because we are trying to edit a `val` property inside the existing state object.

### Why not use var properties?

You might ask:

```text
Why not declare the properties inside ResearchUiState as var?
```

For example:

```kotlin
data class ResearchUiState(
    var sampleId: String = "",
    var isConnected: Boolean = false
)
```

Then this would look possible:

```kotlin
uiState.sampleId = "S001"
```

But this is not the pattern we want for Compose UI state.

`mutableStateOf` makes Compose observe `uiState`. 

**Because uiState is created with mutableStateOf, assigning a new value tells Compose that state changed.**

Compose can clearly notice this:

```kotlin
uiState = uiState.copy(sampleId = "S001")
```

because `uiState` is assigned a new object.

But if you only changed a property inside the same object:

```kotlin
uiState.sampleId = "S001"
```

then the `uiState` variable itself was not replaced.

That makes the state change less clear, and it may not trigger the UI update you expect.

So the preferred pattern is:

```text
Do not edit the old state snapshot.
Create the next state snapshot.
Assign the next snapshot back to uiState.
```

In code:

```kotlin
uiState = uiState.copy(sampleId = "S001")
```

This gives Compose a clear state change:

```text
old uiState -> new uiState
```

The short version:

```text
uiState is var
    -> replace the whole state object

ResearchUiState properties are val
    -> each state object is a stable snapshot

copy()
    -> create the next snapshot while changing only the fields you name
```

## 9. Get the ViewModel in Compose

At the top of your file, add:

```kotlin
import androidx.lifecycle.viewmodel.compose.viewModel
```

Then in your MainActivity:

```kotlin
setContent {
    MaterialTheme {
        // Ask Compose for the ResearchViewModel that belongs to this screen.
        // The left viewModel is a variable name.
        // The right viewModel() is a function call from androidx.lifecycle.viewmodel.compose.viewModel.
        val viewModel: ResearchViewModel = viewModel()

        ResearchScreen(
            viewModel = viewModel
        )
    }
}
```

Pause here, because this line looks strange when you first meet it:

```kotlin
val viewModel: ResearchViewModel = viewModel()
```

Read it like this:

```text
Create a variable named viewModel.
The variable type is ResearchViewModel.
Get the object by calling the Compose viewModel() function.
```

There are three names that look similar, but they are different:

| Code | What it is |
|---|---|
| `ViewModel` | Android parent class |
| `ResearchViewModel` | Your own class; it is a kind of `ViewModel` |
| `viewModel()` | Compose helper function that gets the correct ViewModel object |

Important: this line is not mainly a parent/child assignment example. It is mainly a function-call assignment example.

```text
Inheritance idea:
ResearchViewModel is a kind of ViewModel.

This line's main idea:
viewModel() returns a ResearchViewModel object.
```

This is why we do not usually write this inside a composable:

```kotlin
val viewModel: ResearchViewModel = ResearchViewModel()
```

That would call the constructor directly and make a new object yourself.

In Compose, the screen can re-run many times. The `viewModel()` function asks Android's ViewModel system for the existing `ResearchViewModel` for this screen, or creates one if needed, and keeps it tied to the screen lifecycle.

So mentally translate:

```kotlin
val viewModel: ResearchViewModel = viewModel()
```

to:

```kotlin
val viewModel: ResearchViewModel = getTheResearchViewModelForThisScreen()
```

You may also see the generic form:

```kotlin
val viewModel = viewModel<ResearchViewModel>()
```

Both versions mean the same basic thing. The lesson uses the explicit variable type on the left side, so Kotlin can infer that `viewModel()` should return a `ResearchViewModel`.

If the repeated name feels too confusing, this version is clearer:

```kotlin
val researchViewModel: ResearchViewModel = viewModel()
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
) { // This function is just for connecting viewModel variable to UI drawing function.
    // It separates the UI drawing process from viewModel inputs, so the code for drawing is independent
    val uiState = viewModel.uiState // Getting viewModel state value
    
    ResearchScreenContent( // Plot UI using viewModel values and functions
        uiState = uiState, // You can also direct use uiState = viewModel.uiState
        onSampleIdChange = viewModel::updateSampleId,
        onToggleConnection = viewModel::toggleConnection,
        onMeasure = viewModel::addMeasurement,
        onClear = viewModel::clearMeasurements,
        onExportMessage = viewModel::setExportMessage,
        getCsvText = viewModel::getCsvText
    )
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

```kotlin
onMeasure = viewModel::addMeasurement
```

means:

```text
Pass the addMeasurement function itself.
Do not call it yet.
Let ResearchScreenContent call it later when the button is clicked.
```

That is different from this:

```kotlin
onMeasure = viewModel.addMeasurement()
```

That would mean:

```text
Call addMeasurement immediately while building the UI.
Then pass its result to onMeasure.
```

That is not what we want, because the measurement should happen later, when the user presses the button.

Remember that `ResearchScreenContent` receives:

```kotlin
onMeasure: () -> Unit
```

This means:

```text
onMeasure is a function.
It takes no input.
It returns nothing important.
```

Then the button can use it:

```kotlin
Button(
    onClick = onMeasure
) {
    Text("Measure")
}
```

The button needs a function it can call later.

It is roughly like saying:

```kotlin
onMeasure = {
    viewModel.addMeasurement()
}
```

Both are valid, because both pass a function.

So:

```kotlin
viewModel::addMeasurement
```

is just a shorter way to write:

```kotlin
{
    viewModel.addMeasurement()
}
```

Quick comparison:

| Code | Meaning |
|---|---|
| `viewModel.addMeasurement()` | call the function now |
| `viewModel::addMeasurement` | pass the function itself |
| `{ viewModel.addMeasurement() }` | pass a small function that calls it later |

The `::` syntax is called a function reference.

## 12. Full Lesson 8 app

Here is the full code. Keep your own package name.

        

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

// These CSV/name functions are pure helper functions.
// They do not read uiState, update state, or draw UI.
// The ViewModel can call them, but it does not need to own them.
fun escapeCsv(value: String): String {
    val needsEscaping =
        value.contains(",") ||
                value.contains("\"") ||
                value.contains("\n")

    return if (needsEscaping) {
        "\"" + value.replace("\"", "\"\"") + "\""
    } else {
         value
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

    // There is no remember here because this state lives inside the ViewModel.
    // The by keyword lets us use uiState like a normal variable instead of writing uiState.value.
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
            measurements = uiState.measurements + newMeasurement, // old list + new item = new list. It does not change the old measurements list but reassigned it. 
            exportMessage = ""
        )
    }

    fun clearMeasurements() {
        uiState = uiState.copy(
            measurements = emptyList(), // replace measurements with a new empty list
            exportMessage = ""
        )
    }

    fun setExportMessage(message: String) {
        uiState = uiState.copy(
            exportMessage = message
        )
    }

    // This belongs in the ViewModel because it reads the current uiState.
    // The actual CSV conversion is a pure helper function outside the ViewModel.
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

## 13. Alternative: let ViewModel prepare the CSV export

The full app above keeps the whole export button flow together in the composable:

```kotlin
pendingCsvText = getCsvText()

val filename = if (uiState.sampleId.isNotBlank()) {
    "${safeFilename(uiState.sampleId)}_measurements.csv"
} else {
    "measurements.csv"
}

createCsvLauncher.launch(filename)
```

That is okay for learning, because you can see the whole export flow in one place.

But this block has two different kinds of work:

| Code | Kind of work | Good place |
|---|---|---|
| create CSV text | app/data preparation | ViewModel |
| decide export filename | app/export decision | ViewModel or helper |
| launch Android file picker | UI/system action | Composable |

So a cleaner version is:

```text
ViewModel prepares what to export.
Composable launches the file picker.
```

First, create a small data class for the export request:

```kotlin
data class CsvExportRequest(
    val filename: String,
    val csvText: String
)
```

Then the ViewModel can prepare both the filename and the text:

```kotlin
fun prepareCsvExport(): CsvExportRequest {
    val filename = if (uiState.sampleId.isNotBlank()) {
        "${safeFilename(uiState.sampleId)}_measurements.csv"
    } else {
        "measurements.csv"
    }

    return CsvExportRequest(
        filename = filename,
        csvText = measurementsToCsv(uiState.measurements)
    )
}
```

Now `ResearchScreen` passes that function to the UI:

```kotlin
ResearchScreenContent(
    uiState = uiState,
    onSampleIdChange = viewModel::updateSampleId,
    onToggleConnection = viewModel::toggleConnection,
    onMeasure = viewModel::addMeasurement,
    onClear = viewModel::clearMeasurements,
    onExportMessage = viewModel::setExportMessage,
    prepareCsvExport = viewModel::prepareCsvExport
)
```

So `ResearchScreenContent` receives:

```kotlin
prepareCsvExport: () -> CsvExportRequest
```

Then the export button becomes:

```kotlin
Button(
    onClick = {
        val exportRequest = prepareCsvExport()

        pendingCsvText = exportRequest.csvText

        createCsvLauncher.launch(exportRequest.filename)
    },
    enabled = uiState.measurements.isNotEmpty(),
    modifier = Modifier.fillMaxWidth()
) {
    Text("Export CSV")
}
```

Notice what stayed in the composable:

```kotlin
createCsvLauncher.launch(exportRequest.filename)
```

That line opens the Android file picker, so it belongs with the UI code.

Notice what moved into the ViewModel:

```kotlin
val filename = ...
val csvText = ...
```

Those lines prepare the export data, so they fit well in the ViewModel.

This alternative is not required for the first version of Lesson 8. It is just a cleaner architecture direction:

```text
ViewModel: decide what data should be exported
Composable: ask Android where to save it
```

## 14. What changed compared with Lesson 7?

In Lesson 7, the screen directly owned the state:

```kotlin
var sampleId by remember { mutableStateOf("") }
val measurements = remember { mutableStateListOf<Measurement>() }
```

In Lesson 8, the ViewModel owns the state:

```kotlin
class ResearchViewModel : ViewModel() {
    // There is no remember here because this state lives inside the ViewModel.
    // The by keyword lets us use uiState like a normal variable instead of writing uiState.value.
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

## 15. Important benefit: screen rotation

On Android, rotating the device can recreate the Activity. State stored only inside composables with simple remember can be lost in some situations. A ViewModel is designed to hold screen state and survive configuration changes such as screen rotation. Android Developers

So this architecture is more robust for real Android development.

For your research tablet app, that matters because you do not want to accidentally lose the measurement history because of a screen configuration change.

However, ViewModel is not permanent storage. If the app process is killed, data can still be lost unless you save it to file/database. So for important research data, you still need CSV export or local database saving.

## 16. One limitation of the current ViewModel

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

## 17. Simple architecture comparison

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

## 18. What you should remember from Lesson 8

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
