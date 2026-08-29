# Lesson 7 — Saving and Exporting Measurements as a CSV File

In Lesson 6, your app could store measurements in memory and display the history with LazyColumn.

Now we make the app more useful for research:

```text
Measurement objects
    ↓
CSV text
    ↓
exported .csv file
    ↓
open in Excel / Python / MATLAB / R
```

For a research app, this is one of the most important features.

## 1. What we want to export

In Lesson 6, we had:

```kotlin
data class Measurement(
    val sampleId: String,
    val repetition: Int,
    val value: Double,
    val timestamp: Long
)
```

A CSV file should look like this:

```csv
sample_id,repetition,value,timestamp
S001,1,2.438,1755790000000
S001,2,2.517,1755790001000
S001,3,2.394,1755790002000
```

CSV means comma-separated values. It is simple, widely supported, and easy to open later in Python or Excel.

## 2. Two ways to save data in Android

There are two useful approaches:

| Method | Meaning | Good for |
| --- | --- | --- |
| App-specific internal storage | Save inside the app’s private storage | temporary/internal app data |
| Storage Access Framework export | Let the user choose where to save the file | exporting CSV to Downloads, Drive, OneDrive, etc. |

Android’s app-specific storage can be written with openFileOutput(), and the official Android docs show writing text to internal storage using 

```kotlin
context.openFileOutput(filename, Context.MODE_PRIVATE).
```

For a research app, though, I recommend learning export using the Storage Access Framework, because it lets the user choose a location for the CSV file. Android’s official documentation describes ACTION_CREATE_DOCUMENT as the way to let users save a file to a chosen location, and ActivityResultContracts.CreateDocument is the modern activity-result wrapper that prompts the user to create a new document and returns a content: URI. 

So in this lesson, we will use:

```kotlin
ActivityResultContracts.CreateDocument("text/csv")
```

## 3. Convert measurements to CSV text

First, create a function that turns your list of measurements into a CSV string.

```kotlin
fun measurementsToCsv(
    measurements: List<Measurement>
): String {

    val header = "sample_id,repetition,value,timestamp"

    // joinToString(): Convert a list to a single String, with items separated by a separator.
    // The lambda { measurement -> ... } is a transformation lambda: for each item in the list,
    // decide what String text to produce for that item.
    val rows = measurements.joinToString(separator = "\n") { measurement ->
        // For each measurement, create one line of CSV text
        "${measurement.sampleId}," +
                "${measurement.repetition}," +
                "${measurement.value}," +
                "${measurement.timestamp}"
    }

    // Result: header line, then all measurement lines separated by newlines
    return "$header\n$rows"
}
```

Example:

```kotlin
val csvText = measurementsToCsv(measurements)
// Returns:
// "sample_id,repetition,value,timestamp\nS001,1,2.438,1755790000000\nS001,2,2.517,1755790001000\n..."
```

This gives one large String containing the whole CSV file.

## 4. A safer CSV function

The simple version works for sample IDs like:

- S001
- D1-ETO-W0-U1-S1

But CSV can break if a value contains a comma, quote, or newline.

For example:

Sample, 001

So a safer CSV helper is:

```kotlin
fun escapeCsv(value: String): String {
    // CSV files use commas to separate columns. If data contains a comma, quote, or newline,
    // it breaks the file structure. This function detects and escapes such problematic characters.
    val needsEscaping =
        value.contains(",") ||
                value.contains("\"") ||
                value.contains("\n")

    // If escaping is needed, wrap the value in double quotes and escape any existing quotes.
    return if (needsEscaping) {
        // Double-quote the value and replace any internal quotes with two quotes (CSV escape format)
        "\"" + value.replace("\"", "\"\"") + "\""
    } else {
        // No special characters, return as-is
        value
    }
}
```

Then:

```kotlin
fun measurementsToCsv(
    measurements: List<Measurement>
): String {

    val header = "sample_id,repetition,value,timestamp"

    // For each measurement, create one line of CSV text, escaping the sampleId
    val rows = measurements.joinToString(separator = "\n") { measurement ->
        // Apply escapeCsv to ensure the sampleId won't break the CSV structure
        val sampleId = escapeCsv(measurement.sampleId)

        "$sampleId," +
                "${measurement.repetition}," +
                "${measurement.value}," +
                "${measurement.timestamp}"
    }

    return "$header\n$rows"
}
```

This is a better habit for real research data.

## 5. Exporting a CSV file from Compose

To export a file, we use:

```kotlin
rememberLauncherForActivityResult(...)
```

This allows a Compose screen to launch an Android system action, such as creating a document.

The key idea is:

```text
User clicks Export
    ↓
App prepares CSV text
    ↓
System file picker opens
    ↓
User chooses save location/name
    ↓
App writes CSV text to the selected URI
```

## 6. Important imports

You will need these additional imports:

```kotlin
import android.net.Uri
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.ui.platform.LocalContext
```

Key concepts:
- `Uri`: A pointer/reference to a file location (on device storage, cloud storage, etc.)
- `LocalContext.current`: Provides the current Android system context, available only inside @Composable functions
- `context.contentResolver.openOutputStream(uri)`: Opens a writable output stream to the URI location
- `toByteArray()`: Converts a String to bytes for writing to a file

## 7. Minimal export logic

Inside your composable:

```kotlin
// Provide the current system context for file operations.
// Only use this inside @Composable functions.
val context = LocalContext.current

// Store the CSV text temporarily before the user picks a file location.
var pendingCsvText by remember {
    mutableStateOf("")
}

// Display status messages to the user (success or cancellation).
var exportMessage by remember {
    mutableStateOf("")
}

// Create a launcher that remembers what it is doing (file picking state) after the system dialog closes.
val createCsvLauncher = rememberLauncherForActivityResult(
    // Open the system file picker to create/save a text/csv file
    contract = ActivityResultContracts.CreateDocument("text/csv"),
    // When the user finishes in the system picker and returns to your app, this lambda runs automatically.
    onResult = { uri: Uri? ->  // uri is a pointer (Uri type) to the file location the user chose.

        if (uri != null) {  // If uri is NOT null: We have permission to write to that location.
            // Open an output stream to that Uri, and use it to write bytes.
            // The ?. operator ensures we handle the case where openOutputStream returns null.
            // The .use { } block ensures the stream is closed automatically after writing.
            context.contentResolver.openOutputStream(uri)?.use { outputStream ->
                // Convert the CSV text String to bytes and write to the file.
                // This approach works for small files (<50MB).
                outputStream.write(pendingCsvText.toByteArray())
            }

            exportMessage = "CSV exported successfully."
        } else {  // If uri is null: The user pressed back or cancelled.
            exportMessage = "CSV export cancelled."
        }
    }
)
```

Then the button:

```kotlin
Button(
    onClick = {
        pendingCsvText = measurementsToCsv(measurements)

        createCsvLauncher.launch("measurements.csv")
    },
    enabled = measurements.isNotEmpty()
) {
    Text("Export CSV")
}
```

This means:

If there are measurements:

    convert them to CSV

```text
    open save-file dialog
```

    write the CSV file

## 8. Filename suggestion

Instead of always using:

"measurements.csv"

we can create a filename from the sample ID:

```kotlin
val filename = if (sampleId.isNotBlank()) {
    "${sampleId}_measurements.csv"
} else {
    "measurements.csv"
}
```

But sample IDs may contain characters that are not nice for filenames.

So use a helper:

```kotlin
/**
 * Cleans a string to make it safe for use as a file name.
 * 1. Trims whitespace from both ends.
 * 2. Replaces any character that is NOT a letter, number, underscore, or hyphen with an underscore.
 */
fun safeFilename(text: String): String {
    return text
        .trim()  // Remove leading/trailing whitespace
        .replace(Regex("[^A-Za-z0-9_-]"), "_")
        // [^...] means "NOT one of these characters"
        // A-Za-z0-9_- means "letters, numbers, underscores, hyphens are allowed"
        // Anything else gets replaced with an underscore
}
```

Then:

```kotlin
val filename = if (sampleId.isNotBlank()) {
    "${safeFilename(sampleId)}_measurements.csv"
} else {
    "measurements.csv"
}
```

Example:

Input: "D1-ETO-W0-U1-S1"  
Output: "D1-ETO-W0-U1-S1_measurements.csv" (no change needed, all characters are safe)

Input: "Sample (001)"  
Output: "Sample__001__measurements.csv" (parentheses replaced with underscores)

## 9. Full Lesson 7 app

This version builds on Lesson 6 and adds:

- CSV conversion

- Export CSV button

- save-file picker

- export status message

Replace your MainActivity.kt with this, but keep your own package name.

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
import androidx.compose.runtime.mutableStateListOf
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import kotlin.random.Random

data class Measurement(
    val sampleId: String,
    val repetition: Int,
    val value: Double,
    val timestamp: Long
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

class MainActivity : ComponentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContent {
            MaterialTheme {
                ResearchScreen()
            }
        }
    }
}

@Composable
fun ResearchScreen() {

    val context = LocalContext.current

    var sampleId by remember {
        mutableStateOf("")
    }

    var isConnected by remember {
        mutableStateOf(false)
    }

    var pendingCsvText by remember {
        mutableStateOf("")
    }

    var exportMessage by remember {
        mutableStateOf("")
    }

    val measurements = remember {
        mutableStateListOf<Measurement>()
    }

    val createCsvLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.CreateDocument("text/csv"),
        onResult = { uri: Uri? ->

            if (uri != null) {
                context.contentResolver.openOutputStream(uri)?.use { outputStream ->
                    outputStream.write(pendingCsvText.toByteArray())
                }

                exportMessage = "CSV exported successfully."
            } else {
                exportMessage = "CSV export cancelled."
            }
        }
    )

    val latestMeasurement = measurements.lastOrNull()

    val values = measurements.map {
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
            value = sampleId,
            onValueChange = {
                sampleId = it
            },
            label = {
                Text("Sample ID")
            },
            modifier = Modifier.fillMaxWidth()
        )

        if (sampleId.isBlank()) {
            Text("Please enter a sample ID before measuring.")
        }

        Text(
            text = if (isConnected) {
                "Device status: Connected"
            } else {
                "Device status: Disconnected"
            }
        )

        Button(
            onClick = {
                isConnected = !isConnected
            },
            modifier = Modifier.fillMaxWidth()
        ) {
            Text(
                text = if (isConnected) {
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

                Text("Measurements: ${measurements.size}")
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
                onClick = {

                    val value = Random.nextDouble(
                        from = 0.0,
                        until = 5.0
                    )

                    val repetitionForThisSample = measurements.count {
                        it.sampleId == sampleId
                    } + 1

                    val newMeasurement = Measurement(
                        sampleId = sampleId,
                        repetition = repetitionForThisSample,
                        value = value,
                        timestamp = System.currentTimeMillis()
                    )

                    measurements.add(newMeasurement)
                    exportMessage = ""
                },
                enabled = sampleId.isNotBlank() && isConnected,
                modifier = Modifier.weight(1f)
            ) {
                Text("Measure")
            }

            Button(
                onClick = {
                    measurements.clear()
                    exportMessage = ""
                },
                modifier = Modifier.weight(1f)
            ) {
                Text("Clear")
            }
        }

        Button(
            onClick = {

                pendingCsvText = measurementsToCsv(measurements)

                val filename = if (sampleId.isNotBlank()) {
                    "${safeFilename(sampleId)}_measurements.csv"
                } else {
                    "measurements.csv"
                }

                createCsvLauncher.launch(filename)
            },
            enabled = measurements.isNotEmpty(),
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("Export CSV")
        }

        if (exportMessage.isNotBlank()) {
            Text(exportMessage)
        }

        Text(
            text = "Measurement History",
            fontSize = 20.sp
        )

        LazyColumn(
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            items(measurements) { measurement ->
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

## 10. What happens when you click Export CSV?

When the user clicks:

Export CSV

this line creates the CSV text:

```text
pendingCsvText = measurementsToCsv(measurements)
```

This line creates a suggested filename:

```kotlin
val filename = "${safeFilename(sampleId)}_measurements.csv"
```

This line opens the Android save-file dialog:

```text
createCsvLauncher.launch(filename)
```

Then, after the user chooses where to save the file, this part writes the CSV content:

```kotlin
context.contentResolver.openOutputStream(uri)?.use { outputStream ->
    outputStream.write(pendingCsvText.toByteArray())
}
```

So the app does not need to know whether the file is going to Downloads, Google Drive, OneDrive, or another document provider. The Android file picker handles that.

## 11. Check the CSV output

If you record three measurements for sample S001, the exported file may contain:

```csv
sample_id,repetition,value,timestamp
S001,1,3.1929484818231,1755800000000
S001,2,2.4412848293442,1755800004000
S001,3,4.0394829481348,1755800008000
```

This can be imported into Python:

```kotlin
import pandas as pd

df = pd.read_csv("S001_measurements.csv")
```

print(df.head())

Or opened directly in Excel.

## 12. Improve value formatting in CSV

Right now, the CSV uses the raw Double value:

${measurement.value}

For research, sometimes raw precision is fine. But if you want three decimals, change it to:

"%.3f".format(measurement.value)

So the row becomes:

"$sampleId," +

```text
        "${measurement.repetition}," +
        "${"%.3f".format(measurement.value)}," +
        "${measurement.timestamp}"
```

Then the CSV will look like:

```csv
sample_id,repetition,value,timestamp
S001,1,3.193,1755800000000
S001,2,2.441,1755800004000
S001,3,4.039,1755800008000
```

For actual research data, I usually prefer saving enough precision in the CSV, then formatting only for the screen display.

## 13. Add readable timestamp later

Currently, timestamp is saved as:

```text
System.currentTimeMillis()
```

This gives milliseconds since Unix epoch.

That is good for computation, but not very readable.

Later we can add a readable time column such as:

```csv
sample_id,repetition,value,timestamp_ms,datetime
S001,1,2.438,1755800000000,2026-08-21 21:06:00
```

For now, keeping timestamp as milliseconds is simple and reliable.

## 14. Why we did not use direct external storage

You may see older tutorials that write directly to paths like:

/sdcard/Download/measurements.csv

For modern Android, I would avoid starting there. Android storage rules have changed over the years, and the Storage Access Framework is a cleaner beginner-friendly export route because the user explicitly chooses the save location. Android’s documentation describes SAF as the framework for interacting with document providers and shared storage, including creating documents through the system picker. Android Developers

For a research tablet app, this is also practical: after data collection, the user can save/export the CSV to a chosen folder or cloud location.

## 15. What you learned in Lesson 7

The essential patterns are:

```kotlin
fun measurementsToCsv(
    measurements: List<Measurement>
): String
val createCsvLauncher = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.CreateDocument("text/csv"),
    onResult = { uri: Uri? ->
        // write file
    }
)
context.contentResolver.openOutputStream(uri)?.use { outputStream ->
    outputStream.write(csvText.toByteArray())
}
createCsvLauncher.launch("measurements.csv")
```

Conceptually:

```text
data class objects
    ↓
CSV string
    ↓
system save-file picker
    ↓
.csv file
```

This is a major step toward a real research app.

## Small exercise

Add a status column to the CSV.

First add:

```kotlin
enum class MeasurementStatus {
    LOW,
    NORMAL,
    HIGH
}
```

Then update:

```kotlin
data class Measurement(
    val sampleId: String,
    val repetition: Int,
    val value: Double,
    val timestamp: Long,
    val status: MeasurementStatus
)
```

Then your CSV header should become:

```kotlin
val header = "sample_id,repetition,value,timestamp,status"
```

And each row should include:

measurement.status

## Lesson 8 preview

Next, I suggest we cover app architecture and ViewModel.

Right now, all state lives directly inside ResearchScreen():

```text
sampleId
isConnected
measurements
exportMessage
```

That is okay for learning, but as the app grows, we should move the logic into a ViewModel:

```text
UI screen
    ↓
ViewModel
    ↓
measurement state and logic
```

That will make the app cleaner, easier to test, and closer to real Android development.
