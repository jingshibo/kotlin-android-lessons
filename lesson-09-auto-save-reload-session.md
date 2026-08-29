# Lesson 9 — Simple Data Persistence: Auto-Save and Reload Previous Session

In Lesson 7, we exported CSV manually.

In Lesson 8, we moved app state into a ViewModel.

Now the problem is:

ViewModel state is not permanent storage.

A ViewModel can survive configuration changes such as screen rotation, but it is not a replacement for saving data to disk. Android’s documentation describes ViewModel as useful for screen-level state and configuration changes, while persistent app data should be saved using storage mechanisms such as files, databases, or DataStore. Android Developers

For a research app, this matters a lot. You do not want to lose measurements if:

- the app is closed accidentally

- the tablet restarts

- Android kills the app process

- the user forgets to export CSV

So in Lesson 9, we add a simple auto-save system.

## 1. What we will build

We will add this behaviour:

```text
App starts
    ↓
Load previous measurements from internal storage

User records measurement
    ↓
Add measurement to ViewModel state
    ↓
Automatically save measurements to internal CSV file

User clears measurements
    ↓
Clear ViewModel state
    ↓
Clear saved file
```

This is not the final professional architecture, but it is a very practical beginner-friendly persistence layer.

## 2. Which storage method should we use?

For this lesson, we use app-specific internal storage.

That means the file is saved inside the app’s private storage area. Other apps normally do not access it directly, and the user does not need to choose a file location. Android’s official documentation describes app-specific storage as storage for files meant only for your app, and shows openFileOutput() for writing files inside the app’s internal storage. Android Developers

This is different from Lesson 7:

So the two methods serve different purposes:

```text
Auto-save = protect data during app use
Export CSV = share or analyse data outside the app
```

## 3. The simple persistence strategy

We will save the current measurements to a private CSV file called:

```text
autosave_measurements.csv
```

Every time measurements change, the app rewrites this file.

For a small research prototype, this is acceptable.

Later, for larger datasets, we may use:

```text
Room database
DataStore
streaming file append
repository layer
```

But for now, a CSV autosave file is the easiest to understand.

## 4. Add a constant filename

Add this near your data classes/functions:

```kotlin
const val AUTOSAVE_FILENAME = "autosave_measurements.csv"
```

This prevents typing the filename manually in multiple places.

## 5. Save measurements to internal storage

Add this function:

```kotlin
import android.content.Context

fun saveMeasurementsToInternalStorage(
    context: Context,
    measurements: List<Measurement>
) {
    val csvText = measurementsToCsv(measurements)

    context.openFileOutput(
        AUTOSAVE_FILENAME,
        Context.MODE_PRIVATE
    ).use { outputStream ->
        outputStream.write(csvText.toByteArray())
    }
}
```

This line:

```kotlin
Context.MODE_PRIVATE
```

means the file is private to your app. Android’s documentation notes that on Android 7.0/API 24 or higher, Context.MODE_PRIVATE is required with openFileOutput() to avoid a SecurityException. Android Developers

## 6. Load saved measurements

To load the saved CSV, we need to read the file and convert rows back into Measurement objects.

Add this function:

```kotlin
fun loadMeasurementsFromInternalStorage(
    context: Context
): List<Measurement> {

    return try {
        val csvText = context
            .openFileInput(AUTOSAVE_FILENAME)
            .bufferedReader()
            .use { reader ->
                reader.readText()
            }

        measurementsFromCsv(csvText)

    } catch (e: Exception) {
        emptyList()
    }
}
```

The try/catch is important because the file may not exist the first time the app runs.

So if loading fails, we return:

```kotlin
emptyList()
```

That means:

No saved measurements yet.

## 7. Parse CSV back into measurements

Now add:

```kotlin
fun measurementsFromCsv(
    csvText: String
): List<Measurement> {

    val lines = csvText
        .lines()
        .filter { it.isNotBlank() }

    if (lines.size <= 1) {
        return emptyList()
    }

    return lines
        .drop(1)
        .mapNotNull { line ->

            val parts = line.split(",")

            if (parts.size < 4) {
                return@mapNotNull null
            }

            val sampleId = parts[0]
            val repetition = parts[1].toIntOrNull()
            val value = parts[2].toDoubleOrNull()
            val timestamp = parts[3].toLongOrNull()

            if (
                repetition == null ||
                value == null ||
                timestamp == null
            ) {
                return@mapNotNull null
            }

            Measurement(
                sampleId = sampleId,
                repetition = repetition,
                value = value,
                timestamp = timestamp
            )
        }
}
```

This parser is intentionally simple.

It works for simple sample IDs such as:

```text
S001
D1-ETO-W0-U1-S1
```

But it does not fully handle escaped CSV values with commas inside the sample ID.

For learning, that is okay. In a real app, I would either:

- restrict sample IDs to safe characters

- use a proper CSV parser library

- store JSON instead of CSV internally

- use Room database

For your current research app prototype, restricting sample IDs is probably easiest.

## 8. Add load/save functions to the ViewModel

In Lesson 8, our ResearchViewModel did not know about storage.

Now we add two functions:

```kotlin
fun loadSavedMeasurements(context: Context) {
    val loadedMeasurements =
        loadMeasurementsFromInternalStorage(context)

    uiState = uiState.copy(
        measurements = loadedMeasurements
    )
}
```

and:

```kotlin
private fun autoSave(context: Context) {
    saveMeasurementsToInternalStorage(
        context = context,
        measurements = uiState.measurements
    )
}
```

Then modify addMeasurement() so it receives context:

```kotlin
fun addMeasurement(context: Context) {
    ...
    uiState = uiState.copy(
        measurements = uiState.measurements + newMeasurement,
        exportMessage = ""
    )

    autoSave(context)
}
```

Also modify clearMeasurements():

```kotlin
fun clearMeasurements(context: Context) {
    uiState = uiState.copy(
        measurements = emptyList(),
        exportMessage = ""
    )

    autoSave(context)
}
```

This is not the cleanest architecture because the ViewModel is now receiving Context. For a teaching app, it is acceptable. Later, we would move storage into a repository.

## 9. Load once when the screen opens

Inside the composable, we use:

```kotlin
LaunchedEffect(Unit) {
    onLoadSavedMeasurements()
}
```

LaunchedEffect(Unit) means:

Run this once when this composable first enters the screen.

So the app can load saved measurements when the screen opens.

You need:

```kotlin
import androidx.compose.runtime.LaunchedEffect
```

## 10. Full Lesson 9 version

Below is the full version based on Lesson 8, with internal autosave added.

Keep your own package name.

```kotlin
package com.example.researchapp

import android.content.Context
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
import androidx.compose.runtime.LaunchedEffect
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

const val AUTOSAVE_FILENAME = "autosave_measurements.csv"

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

fun measurementsFromCsv(
    csvText: String
): List<Measurement> {

    val lines = csvText
        .lines()
        .filter { it.isNotBlank() }

    if (lines.size <= 1) {
        return emptyList()
    }

    return lines
        .drop(1)
        .mapNotNull { line ->

            val parts = line.split(",")

            if (parts.size < 4) {
                return@mapNotNull null
            }

            val sampleId = parts[0]
            val repetition = parts[1].toIntOrNull()
            val value = parts[2].toDoubleOrNull()
            val timestamp = parts[3].toLongOrNull()

            if (
                repetition == null ||
                value == null ||
                timestamp == null
            ) {
                return@mapNotNull null
            }

            Measurement(
                sampleId = sampleId,
                repetition = repetition,
                value = value,
                timestamp = timestamp
            )
        }
}

fun safeFilename(text: String): String {
    return text
        .trim()
        .replace(Regex("[^A-Za-z0-9_-]"), "_")
}

fun saveMeasurementsToInternalStorage(
    context: Context,
    measurements: List<Measurement>
) {
    val csvText = measurementsToCsv(measurements)

    context.openFileOutput(
        AUTOSAVE_FILENAME,
        Context.MODE_PRIVATE
    ).use { outputStream ->
        outputStream.write(csvText.toByteArray())
    }
}

fun loadMeasurementsFromInternalStorage(
    context: Context
): List<Measurement> {

    return try {
        val csvText = context
            .openFileInput(AUTOSAVE_FILENAME)
            .bufferedReader()
            .use { reader ->
                reader.readText()
            }

        measurementsFromCsv(csvText)

    } catch (e: Exception) {
        emptyList()
    }
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

    fun loadSavedMeasurements(context: Context) {

        val loadedMeasurements =
            loadMeasurementsFromInternalStorage(context)

        uiState = uiState.copy(
            measurements = loadedMeasurements
        )
    }

    fun addMeasurement(context: Context) {

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

        autoSave(context)
    }

    fun clearMeasurements(context: Context) {
        uiState = uiState.copy(
            measurements = emptyList(),
            exportMessage = ""
        )

        autoSave(context)
    }

    fun setExportMessage(message: String) {
        uiState = uiState.copy(
            exportMessage = message
        )
    }

    fun getCsvText(): String {
        return measurementsToCsv(uiState.measurements)
    }

    private fun autoSave(context: Context) {
        saveMeasurementsToInternalStorage(
            context = context,
            measurements = uiState.measurements
        )
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
    val context = LocalContext.current
    val uiState = viewModel.uiState

    ResearchScreenContent(
        uiState = uiState,
        onSampleIdChange = viewModel::updateSampleId,
        onToggleConnection = viewModel::toggleConnection,
        onMeasure = {
            viewModel.addMeasurement(context)
        },
        onClear = {
            viewModel.clearMeasurements(context)
        },
        onExportMessage = viewModel::setExportMessage,
        onLoadSavedMeasurements = {
            viewModel.loadSavedMeasurements(context)
        },
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
    onLoadSavedMeasurements: () -> Unit,
    getCsvText: () -> String
) {
    val context = LocalContext.current

    var pendingCsvText by androidx.compose.runtime.remember {
        mutableStateOf("")
    }

    LaunchedEffect(Unit) {
        onLoadSavedMeasurements()
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

## 11. What happens now?

When the app starts:

```kotlin
LaunchedEffect(Unit) {
    onLoadSavedMeasurements()
}
```

loads the saved file.

When you click Measure:

```kotlin
viewModel.addMeasurement(context)
```

adds one measurement and then calls:

```kotlin
autoSave(context)
```

When you click Clear:

```kotlin
viewModel.clearMeasurements(context)
```

clears the measurement list and saves an empty CSV file.

So the current dataset should survive closing and reopening the app.

## 12. Important limitation: this simple CSV parser

Our measurementsToCsv() function can escape complex sample IDs, but our simple measurementsFromCsv() parser uses:

```kotlin
line.split(",")
```

That does not correctly parse quoted CSV fields containing commas.

Therefore, for now, keep sample IDs simple:

Good:

S001

D1-ETO-W0-U1-S1

U01_S03

```text
Avoid for now:
Sample, 001
Sample "A"
Sample
001
For a real app, use one of these approaches:
- enforce a safe sample ID format
```

- use a real CSV parser

- store internal autosave as JSON

- use Room database

For your research app, enforcing safe IDs may actually be useful because it also improves file naming and downstream data processing.

## 13. Important limitation: file writing on the UI thread

In this lesson, we write the autosave file directly from the click event path.

For a small CSV file, this is usually okay for a learning prototype. But for larger files or real-time data acquisition, you should not do heavy file writing on the main UI thread.

Later we should move saving into:

viewModelScope.launch

```text
    ↓
Dispatchers.IO
    ↓
repository.saveMeasurements(...)
That keeps file operations off the UI thread.
```

## 14. Internal autosave vs export CSV

Now your app has two layers of protection:

1. Internal autosave

- private to the app

- automatic

- protects against accidental app close

2. Export CSV

- user chooses save location

- useful for Excel/Python/MATLAB

- useful for backup and analysis

This is a good pattern for research apps.

## 15. When should you use DataStore or Room?

For your current prototype:

internal CSV autosave + manual CSV export

```text
is enough.
Use DataStore for small structured settings, such as:
- last used sample ID
```

- device name

- acquisition frequency

- default export filename

- user preferences

Android’s DataStore documentation describes it as a Jetpack storage solution, commonly used for key-value or typed data persistence and as a replacement for SharedPreferences. Android Developers

Use Room database when you need:

- thousands or millions of rows

- querying by sample ID/date/status

- editing/deleting individual measurements

- multiple tables

- reliable structured local database storage

We do not need Room yet, but it is likely useful later if your app becomes a serious data-collection tool.

## 16. What you learned in Lesson 9

The key concepts are:

```text
ViewModel is not permanent storage.
Use internal storage for automatic app-private saving.
Use export CSV for sharing and analysis.
Load saved data when the screen opens.
Save after each measurement.
```

The essential Kotlin/Android patterns are:

```kotlin
context.openFileOutput(
    AUTOSAVE_FILENAME,
    Context.MODE_PRIVATE
)
```

```kotlin
context.openFileInput(AUTOSAVE_FILENAME)
```

```kotlin
LaunchedEffect(Unit) {
    onLoadSavedMeasurements()
}
```

```kotlin
saveMeasurementsToInternalStorage(
    context = context,
    measurements = uiState.measurements
)
```

Conceptually:

```text
Measurement list in memory
    ↓
CSV text
    ↓
private autosave file
    ↓
reload when app restarts
```

## Small exercise

Add a text message showing whether autosaved data was loaded.

For example, add to ResearchUiState:

```kotlin
val loadMessage: String = ""
```

Then in loadSavedMeasurements():

```kotlin
uiState = uiState.copy(
    measurements = loadedMeasurements,
    loadMessage = "Loaded ${loadedMeasurements.size} saved measurements."
)
```

Display it under the title:

```kotlin
if (uiState.loadMessage.isNotBlank()) {
    Text(uiState.loadMessage)
}
```

This helps the user know that previous data has been restored.

## Lesson 10 preview
Next, I suggest we cover coroutines and background work, because real research apps should not block the UI when doing:
- file saving
- sensor reading
- Bluetooth/USB communication
- ML inference
- long-running acquisition
That lesson will prepare us for real device communication and model inference.
