# Lesson 32 — Adding Session CSV Export

In Lesson 31, we connected the screens, navigation, ViewModel, repository, fake device, processing, fake ML, and Room database.

The connected app flow is now:

```text
PatientListScreen
 ↓
PatientDetailScreen
 ↓
MeasurementScreen
 ↓
ResultScreen
```

And the action flow is:

```text
Screen
 ↓
ViewModel
 ↓
Repository
 ↓
Room / FakeDevice / SignalProcessor / FakeModel
```

Now we add the next important research-app function:

```text
export session data
```

This corresponds to **Step A9** in Direction A:

```text
Add export
```

The goal of Lesson 32 is to let the app export one complete session as a CSV file.

---

## 1. Why export matters

For a research app, saving data inside the app is not enough.

Eventually, you may want to analyse the data in:

```text
Python
Excel
MATLAB
R
statistical software
machine-learning scripts
```

So the app should allow exported data.

For our current app, one useful export unit is:

```text
one session
```

A session contains:

```text
patient/sample information
session information
measurements
latest inference result
```

So the CSV should not only contain raw numbers.

It should contain context.

---

## 2. What we want to export

For one session, the export should include:

```text
Patient code
Patient ID
Session ID
Session name
Started time
Ended time
Latest result label
Latest result confidence

Measurement rows:
repetition
timestamp
rawValue
processedValue
status
```

So the exported file may look like this:

```text
Patient Code,P001
Patient ID,1
Session ID,10
Session Name,Baseline
Started At,1755700000
Ended At,1755700600
Result Label,Positive
Result Confidence,0.92

repetition,timestamp,rawValue,processedValue,status
1,1755700001,2.438,2.238,OK
2,1755700002,2.562,2.362,OK
3,1755700003,0.100,-0.100,INVALID
```

This is simple but already useful.

---

## 3. Where export code belongs

Do not build CSV text inside the UI screen.

Avoid this:

```text
ResultScreen
 └── manually builds CSV text
 └── manually loads measurements
 └── manually formats patient/session/result
```

A better structure is:

```text
ResultScreen
 ↓
ViewModel
 ↓
Repository
 ↓
ExportFormatter
```

The responsibilities are:

```text
ResultScreen
 ↓
user clicks Export

ViewModel
 ↓
prepares export request

Repository
 ↓
loads patient/session/measurement/result data

ExportFormatter
 ↓
turns data into CSV text
```

This keeps export logic separate from UI.

---

## 4. Create the `export` package

From Lesson 23, we planned this folder:

```text
export
 └── ExportFormatter.kt
```

Create this folder:

```text
app/src/main/java/com/example/researchapp/export
```

Inside it, create:

```text
ExportFormatter.kt
```

---

## 5. Create `ExportFormatter.kt`

Use this code:

```kotlin
package com.example.researchapp.export

import com.example.researchapp.data.entity.MeasurementEntity
import com.example.researchapp.data.entity.PatientEntity
import com.example.researchapp.data.entity.ResultEntity
import com.example.researchapp.data.entity.SessionEntity

class ExportFormatter {

    fun buildSessionCsv(
        patient: PatientEntity?,
        session: SessionEntity,
        measurements: List<MeasurementEntity>,
        latestResult: ResultEntity?
    ): String {
        val builder = StringBuilder()

        builder.appendLine(
            csvRow("Patient Code", patient?.patientCode ?: "Unknown")
        )
        builder.appendLine(
            csvRow("Patient ID", patient?.id?.toString() ?: "")
        )
        builder.appendLine(
            csvRow("Session ID", session.id.toString())
        )
        builder.appendLine(
            csvRow("Session Name", session.sessionName)
        )
        builder.appendLine(
            csvRow("Started At", session.startedAt.toString())
        )
        builder.appendLine(
            csvRow("Ended At", session.endedAt?.toString() ?: "")
        )
        builder.appendLine(
            csvRow("Result Label", latestResult?.label ?: "")
        )
        builder.appendLine(
            csvRow(
                "Result Confidence",
                latestResult?.confidence?.toString() ?: ""
            )
        )

        builder.appendLine()

        builder.appendLine(
            csvRow(
                "repetition",
                "timestamp",
                "rawValue",
                "processedValue",
                "status"
            )
        )

        measurements.forEach { measurement ->
            builder.appendLine(
                csvRow(
                    measurement.repetition.toString(),
                    measurement.timestamp.toString(),
                    measurement.rawValue.toString(),
                    measurement.processedValue.toString(),
                    measurement.status
                )
            )
        }

        return builder.toString()
    }

    fun buildSessionFileName(
        patientCode: String,
        sessionName: String,
        sessionId: Long
    ): String {
        val safePatientCode = makeSafeFilePart(patientCode)
        val safeSessionName = makeSafeFilePart(sessionName)

        return "${safePatientCode}_${safeSessionName}_session_${sessionId}.csv"
    }

    private fun csvRow(
        vararg values: String
    ): String {
        return values.joinToString(",") { value ->
            escapeCsvValue(value)
        }
    }

    private fun escapeCsvValue(
        value: String
    ): String {
        val escaped = value.replace("\"", "\"\"")

        return if (
            escaped.contains(",") ||
            escaped.contains("\n") ||
            escaped.contains("\"")
        ) {
            "\"$escaped\""
        } else {
            escaped
        }
    }

    private fun makeSafeFilePart(
        text: String
    ): String {
        return text
            .trim()
            .replace(" ", "_")
            .replace("/", "_")
            .replace("\\", "_")
            .replace(":", "_")
            .ifBlank {
                "unknown"
            }
    }
}
```

This class does two main things:

```text
buildSessionCsv()
 ↓
creates CSV text

buildSessionFileName()
 ↓
creates a safe filename
```

---

## 6. Why use `escapeCsvValue()`

CSV looks simple, but values may contain commas or quotes.

For example:

```text
Session name = Baseline, morning
```

If we write this directly:

```text
Session Name,Baseline, morning
```

CSV software may think there are three columns instead of two.

So we need escaping.

This function:

```kotlin
private fun escapeCsvValue(
    value: String
): String {
    val escaped = value.replace("\"", "\"\"")

    return if (
        escaped.contains(",") ||
        escaped.contains("\n") ||
        escaped.contains("\"")
    ) {
        "\"$escaped\""
    } else {
        escaped
    }
}
```

means:

```text
If the value contains comma, quote, or new line,
wrap it safely in quotes.
```

This is a good habit for export code.

---

## 7. Update `MeasurementRepository`

Now connect `ExportFormatter` to the repository.

In `MeasurementRepository.kt`, add this import:

```kotlin
import com.example.researchapp.export.ExportFormatter
```

Then update the constructor:

```kotlin
class MeasurementRepository(
    context: Context,
    private val deviceDataSource: DeviceDataSource = FakeDeviceDataSource(),
    private val signalProcessor: SignalProcessor = SignalProcessor(),
    private val modelRunner: ModelRunner = FakeModelRunner(),
    private val exportFormatter: ExportFormatter = ExportFormatter()
) {
    ...
}
```

Now the repository can use export formatting.

---

## 8. Add export function to repository

Inside `MeasurementRepository`, add:

```kotlin
suspend fun buildSessionCsvExport(
    sessionId: Long
): String? {
    val session = getSessionById(
        sessionId = sessionId
    ) ?: return null

    val patient = getPatientById(
        patientId = session.patientId
    )

    val measurements = getMeasurementsForSession(
        sessionId = sessionId
    )

    val latestResult = getLatestResultForSession(
        sessionId = sessionId
    )

    return exportFormatter.buildSessionCsv(
        patient = patient,
        session = session,
        measurements = measurements,
        latestResult = latestResult
    )
}
```

This function does:

```text
load session
 ↓
load patient
 ↓
load measurements
 ↓
load latest result
 ↓
build CSV text
```

The return type is:

```kotlin
String?
```

because the session may not exist.

---

## 9. Add export filename function to repository

Also add:

```kotlin
suspend fun buildSessionCsvFileName(
    sessionId: Long
): String {
    val session = getSessionById(
        sessionId = sessionId
    )

    val patient = session?.let {
        getPatientById(
            patientId = it.patientId
        )
    }

    return exportFormatter.buildSessionFileName(
        patientCode = patient?.patientCode ?: "unknown_patient",
        sessionName = session?.sessionName ?: "unknown_session",
        sessionId = sessionId
    )
}
```

This creates a filename such as:

```text
P001_Baseline_session_10.csv
```

or:

```text
D1-ETO-W0-U1-S1_face_A_session_10.csv
```

This is better than a generic name like:

```text
export.csv
```

because research data files should be identifiable.

---

## 10. Update `ResearchUiState`

In `ResearchUiState.kt`, add:

```kotlin
val pendingExportText: String = "",
val pendingExportFileName: String = ""
```

So the end of your state may look like:

```kotlin
val latestPrediction: PredictionResult? = null,

val pendingExportText: String = "",
val pendingExportFileName: String = "",

val isLoading: Boolean = false,
val message: String = ""
```

These fields temporarily store the CSV text and filename before the UI writes the file.

---

## 11. Why store pending export text in UI state?

On Android, saving a file through the system file picker usually happens from the UI layer.

The ViewModel can prepare:

```text
CSV text
filename
```

But the UI launches:

```text
CreateDocument
```

and writes to the selected file location.

So the flow is:

```text
ResultScreen Export button
 ↓
ViewModel prepares CSV text
 ↓
UI launches file picker
 ↓
User chooses save location
 ↓
UI writes CSV text to selected URI
```

That is why we store:

```text
pendingExportText
pendingExportFileName
```

in `ResearchUiState`.

---

## 12. Add export preparation function to ViewModel

In `ResearchViewModel.kt`, add:

```kotlin
fun prepareSessionCsvExport(
    onExportReady: (String) -> Unit
) {
    val sessionId = uiState.currentSessionId

    if (sessionId == null) {
        uiState = uiState.copy(
            message = "No session selected for export"
        )
        return
    }

    viewModelScope.launch {
        uiState = uiState.copy(
            isLoading = true,
            message = "Preparing export..."
        )

        try {
            val csvText =
                measurementRepository.buildSessionCsvExport(
                    sessionId = sessionId
                )

            if (csvText == null) {
                uiState = uiState.copy(
                    isLoading = false,
                    message = "Could not find session for export"
                )
                return@launch
            }

            val fileName =
                measurementRepository.buildSessionCsvFileName(
                    sessionId = sessionId
                )

            uiState = uiState.copy(
                pendingExportText = csvText,
                pendingExportFileName = fileName,
                isLoading = false,
                message = "Export ready"
            )

            onExportReady(fileName)
        } catch (e: Exception) {
            uiState = uiState.copy(
                isLoading = false,
                message = "Could not prepare export"
            )
        }
    }
}
```

This function prepares the CSV but does not directly write the file.

That keeps the ViewModel separate from Android file-picker UI details.

---

## 13. Add function after export is complete

Also add:

```kotlin
fun onExportFinished() {
    uiState = uiState.copy(
        pendingExportText = "",
        pendingExportFileName = "",
        message = "Export complete"
    )
}

fun onExportFailed() {
    uiState = uiState.copy(
        message = "Export failed"
    )
}
```

These functions let the UI tell the ViewModel what happened after writing the file.

---

## 14. Update `ResearchApp.kt` for file export

In `ResearchApp.kt`, add these imports:

```kotlin
import android.content.Context
import android.net.Uri
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.ui.platform.LocalContext
```

Then add this helper function near the bottom of the file:

```kotlin
private fun writeTextToUri(
    context: Context,
    uri: Uri,
    text: String
) {
    context.contentResolver.openOutputStream(uri)?.use { outputStream ->
        outputStream.write(
            text.toByteArray()
        )
    }
}
```

This writes the CSV text to the file selected by the user.

---

## 15. Create the file launcher in `ResearchApp`

Inside `ResearchApp`, before `NavHost`, add:

```kotlin
val context = LocalContext.current

val exportLauncher =
    rememberLauncherForActivityResult(
        contract = ActivityResultContracts.CreateDocument("text/csv")
    ) { uri ->
        if (uri != null) {
            try {
                writeTextToUri(
                    context = context,
                    uri = uri,
                    text = uiState.pendingExportText
                )

                viewModel.onExportFinished()
            } catch (e: Exception) {
                viewModel.onExportFailed()
            }
        }
    }
```

This launcher opens Android’s file creation UI.

The important part is:

```kotlin
ActivityResultContracts.CreateDocument("text/csv")
```

This tells Android:

```text
Let the user create a CSV file.
```

---

## 16. Connect the ResultScreen export button

In Lesson 31, the Result route had this placeholder:

```kotlin
onExportClick = {
    // Export will be connected in a later lesson.
}
```

Now replace it with:

```kotlin
onExportClick = {
    viewModel.prepareSessionCsvExport { fileName ->
        exportLauncher.launch(fileName)
    }
}
```

So the `ResultScreen` call becomes:

```kotlin
ResultScreen(
    sessionId = sessionId,
    predictionLabel = uiState.latestPrediction?.label,
    predictionConfidence = uiState.latestPrediction?.confidence,
    measurementCount = uiState.measurements.size,
    message = uiState.message,
    onExportClick = {
        viewModel.prepareSessionCsvExport { fileName ->
            exportLauncher.launch(fileName)
        }
    },
    onBackClick = {
        navController.popBackStack()
    }
)
```

Now the export button has a real action.

---

## 17. Full export flow

After Lesson 32, the export flow is:

```text
User clicks Export Session CSV
 ↓
ResultScreen calls onExportClick
 ↓
ResearchApp calls viewModel.prepareSessionCsvExport()
 ↓
ViewModel asks repository to build CSV
 ↓
Repository loads session, patient, measurements, latest result
 ↓
ExportFormatter builds CSV text
 ↓
ViewModel stores pendingExportText and pendingExportFileName
 ↓
ResearchApp launches CreateDocument
 ↓
User chooses file location
 ↓
ResearchApp writes CSV text to URI
 ↓
ViewModel marks export complete
```

This is a clean flow.

---

## 18. Why the repository does not write the file directly

You may wonder:

```text
Why does the repository build CSV text but not save the file?
```

Because Android file saving often needs UI interaction.

The user chooses where to save the file.

That is handled through:

```text
ActivityResultContracts.CreateDocument
```

which belongs in the Compose/UI layer.

So the split is:

```text
Repository
 ↓
builds export content

UI layer
 ↓
asks user where to save
writes file
```

This is a practical compromise.

---

## 19. Why export one session first?

A more complete app may export:

```text
one session
one patient with all sessions
all patients
daily batch export
JSON backup
raw binary data
```

But for Direction A, one session export is the correct first step.

Why?

Because a session is the natural research unit:

```text
one data-collection run
one set of measurements
one result
one export file
```

Later, patient-level export can combine several sessions.

---

## 20. Should export include invalid measurements?

In this lesson, the CSV includes all measurements:

```kotlin
val measurements = getMeasurementsForSession(
    sessionId = sessionId
)
```

That includes:

```text
OK
INVALID
ERROR
```

This is intentional.

For research traceability, export should usually include all collected records.

The status column tells you which values were valid or invalid.

So the export keeps the raw history.

Later, if you want, you can add a second export option:

```text
Export valid measurements only
```

But the default research export should preserve data.

---

## 21. Should export include raw and processed values?

Yes.

The CSV includes:

```text
rawValue
processedValue
```

This is important because:

```text
rawValue
 ↓
lets you reprocess later

processedValue
 ↓
shows what the app used at the time
```

If your processing method changes later, raw values are still available.

This is a key research-data habit.

---

## 22. Should export include model version?

Eventually, yes.

Right now, `ResultEntity` contains:

```text
sessionId
label
confidence
createdAt
```

So the export can only include those fields.

Later, when using a real model, I would add:

```text
modelVersion
processingVersion
featureWindowSize
normalisationVersion
```

Then the export should include them too.

For now, we keep the beginner skeleton simple.

---

## 23. Current architecture after Lesson 32

After this lesson, the architecture becomes:

```text
ResearchApp
 ↓
ResultScreen
 ↓
ResearchViewModel
 ↓
MeasurementRepository
 ├── ResearchDatabase
 ├── DeviceDataSource
 ├── SignalProcessor
 ├── ModelRunner
 └── ExportFormatter
```

The export path is:

```text
Room database
 ↓
patient + session + measurements + latest result
 ↓
MeasurementRepository
 ↓
ExportFormatter
 ↓
CSV text
 ↓
ResearchApp file picker
 ↓
saved CSV file
```

This completes the main fake research workflow.

---

## 24. Current files after Lesson 32

After this lesson, the project should include:

```text
export
 └── ExportFormatter.kt
```

And these files should be updated:

```text
data/MeasurementRepository.kt
viewmodel/ResearchUiState.kt
viewmodel/ResearchViewModel.kt
ui/ResearchApp.kt
ui/ResultScreen.kt
```

The project structure is now:

```text
com.example.researchapp
 ├── data
 │    ├── ResearchDatabase.kt
 │    ├── MeasurementRepository.kt
 │    ├── entity
 │    └── dao
 │
 ├── device
 │    ├── DeviceDataSource.kt
 │    └── FakeDeviceDataSource.kt
 │
 ├── processing
 │    ├── SignalProcessor.kt
 │    └── SignalFeatures.kt
 │
 ├── ml
 │    ├── PredictionResult.kt
 │    ├── ModelRunner.kt
 │    └── FakeModelRunner.kt
 │
 ├── export
 │    └── ExportFormatter.kt
 │
 ├── ui
 │    ├── ResearchApp.kt
 │    ├── PatientListScreen.kt
 │    ├── PatientDetailScreen.kt
 │    ├── MeasurementScreen.kt
 │    └── ResultScreen.kt
 │
 └── viewmodel
      ├── ResearchUiState.kt
      ├── ResearchViewModel.kt
      └── AppStateEnums.kt
```

---

## 25. What you learned in Lesson 32

You created:

```text
ExportFormatter
```

You added repository functions to:

```text
buildSessionCsvExport()
buildSessionCsvFileName()
```

You updated the ViewModel to:

```text
prepare CSV export
store pending export text
store pending filename
handle export success or failure
```

You connected the Result Screen export button to Android’s file creation flow.

The most important mental model is:

```text
Export is part of the research pipeline,
not just a UI button.
```

## 26. Final flow after Lesson 32

The clean export flow is:

```text
ResultScreen
 ↓
ViewModel
 ↓
Repository
 ↓
ExportFormatter
 ↓
CSV text
 ↓
Android file picker
 ↓
saved file
```

Now the app can support the complete fake-data research workflow:

```text
create patient
 ↓
create session
 ↓
connect fake device
 ↓
collect processed measurements
 ↓
run fake inference
 ↓
save result
 ↓
export session CSV
```

---

## 27. Lesson 33 preview

In Lesson 33, we will test the whole fake-data workflow.

We will review the expected user path:

```text
open app
create patient
create session
connect fake device
start acquisition
stop acquisition
run inference
view result
export CSV
```

Then we will discuss common errors, missing dependencies, and how to check whether each layer is working correctly.
