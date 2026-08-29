# Lesson 21 — Exporting Complete Research Data

In Lesson 20, the app became more like an **edge-AI research app**:

```text
Patient
 ↓
Session
 ↓
Measurements
 ↓
Signal processing
 ↓
ML inference
 ↓
Result
```

Now we need to export this richer data.

Earlier in the original tutorial, Lesson 7 covered exporting simple measurement data as a CSV file and added an **Export CSV** button with a save-file picker. fileciteturn1file0L270-L306 But now our app has more than only measurements.

The app now has:

```text
Patient
Session
Measurement
Result
```

So Lesson 21 is about exporting **complete research data**, not only a flat measurement list.

---

## 1. Why export matters

Room is useful for storing structured data inside the app.

But for research, you usually also need data outside the app:

```text
Python analysis
Excel checking
statistical analysis
model validation
backup
sharing with collaborators
publication figures
```

So the app needs an export function.

A good research export should answer:

```text
Who/what was measured?
Which session was this?
When did the session start and end?
What raw values were collected?
What processed values were produced?
What model result was generated?
```

So the export should include both:

```text
metadata
```

and:

```text
measurement data
```

---

## 2. Room storage vs export files

Do not confuse these two.

```text
Room database
 ↓
internal structured app storage

CSV / JSON export
 ↓
external file for analysis, backup, or sharing
```

Room is mainly for the app to manage data.

CSV or JSON is mainly for the researcher to use the data elsewhere.

So the structure is:

```text
Room
 ↓
repository reads selected data
 ↓
export text is generated
 ↓
user saves CSV/JSON file
 ↓
file is opened in Python/Excel/lab computer
```

---

## 3. CSV vs JSON

For our app, both CSV and JSON are useful.

### CSV

CSV is good for table-like data:

```text
timestamp,repetition,rawValue,processedValue,status
1755701000,1,2.438,2.238,OK
1755701001,2,2.421,2.221,OK
```

CSV is easy to open in:

```text
Excel
Python pandas
R
MATLAB
```

So CSV is good for measurement rows.

### JSON

JSON is better for nested structured data:

```json
{
  "patient": {
    "id": 1,
    "patientCode": "P001"
  },
  "session": {
    "id": 10,
    "sessionName": "Baseline"
  },
  "measurements": [
    {
      "repetition": 1,
      "rawValue": 2.438,
      "processedValue": 2.238
    }
  ],
  "result": {
    "label": "Positive",
    "confidence": 0.92
  }
}
```

JSON is better when you want to preserve the full structure:

```text
Patient
 └── Session
      ├── Measurements
      └── Result
```

Kotlin’s official serialization documentation describes `kotlinx.serialization` as the Kotlin library used to serialize objects to JSON, which is useful if you later want cleaner JSON export code. citeturn902638search4

For Lesson 21, we will start with CSV because it is simpler and very useful for research analysis.

---

## 4. What should one session export contain?

For one measurement session, I suggest exporting:

```text
patient metadata
session metadata
measurement rows
latest/final result
```

Example CSV structure:

```text
Patient Code,P001
Session Name,Baseline
Session ID,10
Started At,1755700000
Ended At,1755700600
Result Label,Positive
Result Confidence,0.92

repetition,timestamp,rawValue,processedValue,status
1,1755700001,2.438,2.238,OK
2,1755700002,2.421,2.221,OK
3,1755700003,2.470,2.270,OK
```

This is not a pure rectangular table because the top part is metadata and the bottom part is measurement rows.

That is okay for a practical research export.

Later, if you want easier Python analysis, you may export separate files:

```text
session_metadata.csv
measurements.csv
results.csv
```

But for a beginner app, one file per session is easier.

---

## 5. Add DAO functions for export

To export a session, the repository needs to load:

```text
Patient
Session
Measurements
Results
```

So we need DAO functions.

### PatientDao

```kotlin
@Dao
interface PatientDao {

    @Insert
    suspend fun insertPatient(
        patient: PatientEntity
    ): Long

    @Query("SELECT * FROM patients ORDER BY createdAt DESC")
    suspend fun getAllPatients(): List<PatientEntity>

    @Query("SELECT * FROM patients WHERE id = :patientId LIMIT 1")
    suspend fun getPatientById(
        patientId: Long
    ): PatientEntity?
}
```

The new function is:

```kotlin
getPatientById(patientId)
```

It returns:

```kotlin
PatientEntity?
```

because the patient might not exist.

---

### SessionDao

```kotlin
@Dao
interface SessionDao {

    @Insert
    suspend fun insertSession(
        session: SessionEntity
    ): Long

    @Query("SELECT * FROM sessions WHERE patientId = :patientId ORDER BY startedAt DESC")
    suspend fun getSessionsForPatient(
        patientId: Long
    ): List<SessionEntity>

    @Query("SELECT * FROM sessions WHERE id = :sessionId LIMIT 1")
    suspend fun getSessionById(
        sessionId: Long
    ): SessionEntity?

    @Query("UPDATE sessions SET endedAt = :endedAt WHERE id = :sessionId")
    suspend fun endSession(
        sessionId: Long,
        endedAt: Long
    )
}
```

The new function is:

```kotlin
getSessionById(sessionId)
```

---

### MeasurementDao

We already had something like:

```kotlin
@Query("SELECT * FROM measurements WHERE sessionId = :sessionId ORDER BY timestamp ASC")
suspend fun getMeasurementsForSession(
    sessionId: Long
): List<MeasurementEntity>
```

This is exactly what we need for export.

---

### ResultDao

```kotlin
@Dao
interface ResultDao {

    @Insert
    suspend fun insertResult(
        result: ResultEntity
    ): Long

    @Query("SELECT * FROM results WHERE sessionId = :sessionId ORDER BY createdAt DESC")
    suspend fun getResultsForSession(
        sessionId: Long
    ): List<ResultEntity>

    @Query("SELECT * FROM results WHERE sessionId = :sessionId ORDER BY createdAt DESC LIMIT 1")
    suspend fun getLatestResultForSession(
        sessionId: Long
    ): ResultEntity?
}
```

For export, we often want the latest result:

```kotlin
getLatestResultForSession(sessionId)
```

---

## 6. Create a CSV builder function

Now we create a function that turns session data into CSV text.

Inside `MeasurementRepository`, add:

```kotlin
suspend fun buildSessionCsvExport(
    sessionId: Long
): String {
    val session = sessionDao.getSessionById(sessionId)
        ?: throw IllegalArgumentException("Session not found")

    val patient = patientDao.getPatientById(session.patientId)
        ?: throw IllegalArgumentException("Patient not found")

    val measurements = measurementDao.getMeasurementsForSession(
        sessionId = sessionId
    )

    val latestResult = resultDao.getLatestResultForSession(
        sessionId = sessionId
    )

    val builder = StringBuilder()

    builder.appendLine("Patient Code,${patient.patientCode}")
    builder.appendLine("Patient ID,${patient.id}")
    builder.appendLine("Session Name,${session.sessionName}")
    builder.appendLine("Session ID,${session.id}")
    builder.appendLine("Started At,${session.startedAt}")
    builder.appendLine("Ended At,${session.endedAt ?: ""}")
    builder.appendLine("Result Label,${latestResult?.label ?: ""}")
    builder.appendLine("Result Confidence,${latestResult?.confidence ?: ""}")
    builder.appendLine()

    builder.appendLine(
        "repetition,timestamp,rawValue,processedValue,status"
    )

    for (measurement in measurements) {
        builder.appendLine(
            "${measurement.repetition}," +
                "${measurement.timestamp}," +
                "${measurement.rawValue}," +
                "${measurement.processedValue}," +
                measurement.status
        )
    }

    return builder.toString()
}
```

The flow is:

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

---

## 7. Why use `StringBuilder`?

You could write:

```kotlin
var csv = ""
csv += "Patient Code,..."
csv += "Session Name,..."
```

But for many rows, that is inefficient.

`StringBuilder` is better because it gradually builds the text.

The pattern is:

```kotlin
val builder = StringBuilder()

builder.appendLine("line 1")
builder.appendLine("line 2")
builder.appendLine("line 3")

return builder.toString()
```

This is useful for CSV export.

---

## 8. Escape CSV values

There is one important detail.

If text contains commas, the CSV can break.

For example:

```text
notes = "Baseline, morning session"
```

This contains a comma.

A safer helper function is:

```kotlin
fun escapeCsvValue(
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

Then use:

```kotlin
escapeCsvValue(patient.patientCode)
```

For numeric values, this is less important.

For text fields like patient code, session name, notes, and status, it is safer.

---

## 9. Improved CSV builder with escaping

A more careful version:

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

Then:

```kotlin
builder.appendLine(
    "Patient Code,${escapeCsvValue(patient.patientCode)}"
)

builder.appendLine(
    "Session Name,${escapeCsvValue(session.sessionName)}"
)
```

For measurement rows:

```kotlin
builder.appendLine(
    listOf(
        measurement.repetition.toString(),
        measurement.timestamp.toString(),
        measurement.rawValue.toString(),
        measurement.processedValue.toString(),
        escapeCsvValue(measurement.status)
    ).joinToString(",")
)
```

This is more robust.

---

## 10. File naming rule

A good export file name should contain enough information.

For example:

```text
P001_Baseline_session10_2026-08-26.csv
```

or:

```text
P001_Baseline_measurements.csv
```

A simple function:

```kotlin
fun buildExportFileName(
    patientCode: String,
    sessionName: String,
    sessionId: Long
): String {
    val safePatientCode = patientCode.replace(" ", "_")
    val safeSessionName = sessionName.replace(" ", "_")

    return "${safePatientCode}_${safeSessionName}_session${sessionId}.csv"
}
```

For real apps, you may also want to remove symbols such as:

```text
/
\
:
*
?
"
<
>
|
```

because they can be problematic in file names.

A safer simple version:

```kotlin
fun makeSafeFilePart(
    text: String
): String {
    return text
        .replace(" ", "_")
        .replace("/", "_")
        .replace("\\", "_")
        .replace(":", "_")
}
```

Then:

```kotlin
fun buildExportFileName(
    patientCode: String,
    sessionName: String,
    sessionId: Long
): String {
    return "${makeSafeFilePart(patientCode)}_" +
        "${makeSafeFilePart(sessionName)}_" +
        "session${sessionId}.csv"
}
```

---

## 11. Add export state to `ResearchUiState`

The ViewModel needs to hold the export text temporarily.

Add:

```kotlin
val pendingExportText: String = ""
val pendingExportFileName: String = ""
```

So part of `ResearchUiState` becomes:

```kotlin
data class ResearchUiState(
    val patientCode: String = "",
    val sessionName: String = "",
    val currentPatientId: Long? = null,
    val currentSessionId: Long? = null,

    val measurements: List<MeasurementEntity> = emptyList(),
    val latestPrediction: PredictionResult? = null,

    val pendingExportText: String = "",
    val pendingExportFileName: String = "",

    val isLoading: Boolean = false,
    val message: String = ""
)
```

This allows this flow:

```text
user clicks Export
 ↓
ViewModel builds CSV text
 ↓
UI opens save-file picker
 ↓
user chooses location
 ↓
ViewModel writes CSV text to selected Uri
```

---

## 12. Prepare export in the ViewModel

Inside `ResearchViewModel`:

```kotlin
fun prepareCurrentSessionCsvExport(
    onExportReady: (String) -> Unit
) {
    val sessionId = uiState.currentSessionId

    if (sessionId == null) {
        uiState = uiState.copy(
            message = "No active session to export"
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

            val fileName =
                "session_${sessionId}_export.csv"

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

The `onExportReady(fileName)` callback tells the UI:

```text
The CSV text is ready.
Now open the save-file picker.
```

Why do we need the UI involved?

Because the file picker is an Android UI action.

---

## 13. Use `CreateDocument` for saving

For user-selected export, we can use:

```kotlin
ActivityResultContracts.CreateDocument("text/csv")
```

The Android API reference describes `CreateDocument` as an activity result contract that prompts the user to select a path for a new document and returns a `content:` URI for the created item. citeturn902638search0

In the screen:

```kotlin
val exportLauncher =
    rememberLauncherForActivityResult(
        contract = ActivityResultContracts.CreateDocument("text/csv")
    ) { uri ->
        if (uri != null) {
            viewModel.writePendingExportToUri(
                context = context,
                uri = uri
            )
        }
    }
```

Required imports:

```kotlin
import android.net.Uri
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
```

The user flow is:

```text
button clicked
 ↓
CSV text prepared
 ↓
system save dialog opens
 ↓
user chooses location/name
 ↓
Uri returned
 ↓
app writes export text to Uri
```

---

## 14. Write export text to Uri

Inside the ViewModel:

```kotlin
fun writePendingExportToUri(
    context: Context,
    uri: Uri
) {
    val exportText = uiState.pendingExportText

    if (exportText.isBlank()) {
        uiState = uiState.copy(
            message = "No export data available"
        )
        return
    }

    viewModelScope.launch {
        try {
            withContext(Dispatchers.IO) {
                context.contentResolver
                    .openOutputStream(uri)
                    ?.use { outputStream ->
                        outputStream.write(
                            exportText.toByteArray()
                        )
                    }
            }

            uiState = uiState.copy(
                message = "Export saved"
            )
        } catch (e: Exception) {
            uiState = uiState.copy(
                message = "Export failed"
            )
        }
    }
}
```

This uses:

```kotlin
context.contentResolver.openOutputStream(uri)
```

to write to the user-selected document.

The Android shared storage documentation explains that `ACTION_CREATE_DOCUMENT` lets the user choose a location where the app can write a file through the Storage Access Framework. citeturn902638search11

---

## 15. Export button in Compose

Inside `MeasurementScreen` or `ResultScreen`:

```kotlin
val context = LocalContext.current

val exportLauncher =
    rememberLauncherForActivityResult(
        contract = ActivityResultContracts.CreateDocument("text/csv")
    ) { uri ->
        if (uri != null) {
            viewModel.writePendingExportToUri(
                context = context,
                uri = uri
            )
        }
    }

Button(
    onClick = {
        viewModel.prepareCurrentSessionCsvExport(
            onExportReady = { fileName ->
                exportLauncher.launch(fileName)
            }
        )
    },
    enabled =
        uiState.currentSessionId != null &&
        uiState.acquisitionState != AcquisitionState.RECORDING
) {
    Text("Export Session CSV")
}
```

Notice:

```kotlin
enabled = uiState.acquisitionState != AcquisitionState.RECORDING
```

For the beginner version, we disable export during recording.

Why?

Because exporting while data is still being collected can create confusion:

```text
Did the export include all measurements?
Was the session finished?
Was the result already generated?
```

For a research app, a safer first rule is:

```text
Stop acquisition first.
Then export.
```

---

## 16. Where should export happen?

Export is not just UI.

The responsibilities should be separated:

```text
UI
 ↓
opens save-file picker

ViewModel
 ↓
coordinates export action and updates message

Repository
 ↓
loads data from Room and builds export text

Room DAOs
 ↓
provide patient/session/measurement/result data
```

So the architecture is:

```text
ResultScreen / MeasurementScreen
 ↓
ResearchViewModel
 ↓
MeasurementRepository
 ↓
Room database
```

The UI should not manually query Room or build CSV rows.

---

## 17. Should we export one session or all sessions?

For now, export one session.

That is easier and safer:

```text
current session
 ↓
one CSV file
```

Later, you may add:

```text
export selected patient
export all sessions for a patient
export all data
export date range
export only unexported sessions
```

But starting with one-session export is better.

It matches the research workflow:

```text
collect one session
run inference
export that session
check in Python/Excel
```

---

## 18. JSON export idea

CSV is good, but JSON may be better for complete structured export.

A JSON export might look like:

```json
{
  "patientCode": "P001",
  "sessionName": "Baseline",
  "sessionId": 10,
  "measurements": [
    {
      "repetition": 1,
      "timestamp": 1755700001,
      "rawValue": 2.438,
      "processedValue": 2.238,
      "status": "OK"
    }
  ],
  "result": {
    "label": "Positive",
    "confidence": 0.92
  }
}
```

This preserves hierarchy better than CSV.

For Android/Kotlin, `kotlinx.serialization` can serialize Kotlin objects into JSON; the Kotlin documentation shows using `@Serializable` data classes and `Json.encodeToString(...)` for JSON output. citeturn902638search4turn902638search14

We do not need to fully implement JSON in this lesson, but the idea is important.

---

## 19. Export DTOs for JSON

A DTO means:

```text
Data Transfer Object
```

It is a data class designed for sending/exporting data.

For example:

```kotlin
@Serializable
data class SessionExportDto(
    val patientCode: String,
    val sessionName: String,
    val sessionId: Long,
    val startedAt: Long,
    val endedAt: Long?,
    val measurements: List<MeasurementExportDto>,
    val result: ResultExportDto?
)

@Serializable
data class MeasurementExportDto(
    val repetition: Int,
    val timestamp: Long,
    val rawValue: Double,
    val processedValue: Double,
    val status: String
)

@Serializable
data class ResultExportDto(
    val label: String,
    val confidence: Double,
    val createdAt: Long
)
```

Then you can eventually do:

```kotlin
val jsonText = Json {
    prettyPrint = true
}.encodeToString(exportDto)
```

This is useful later, especially if you want to preserve metadata cleanly.

---

## 20. What Should Be Exported for ML Traceability?

For a research app with ML inference, the export should eventually include:

- model version
- processing version
- feature window size
- normalisation parameters version
- prediction threshold
- raw values
- processed values
- final result

In Lesson 20, we mentioned that saving the model version is important.
A future ResultEntity may become:

```kotlin
@Entity(tableName = "results")
data class ResultEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,

    val sessionId: Long,
    val label: String,
    val confidence: Double,
    val modelVersion: String,
    val processingVersion: String,
    val createdAt: Long = System.currentTimeMillis()
)
```

For now, we keep it simple.
But the research habit is:

> Export enough information to understand how the result was produced.

## 21. Export Flow After Lesson 21

The complete export flow is now:

```text
User stops acquisition
 ↓
User runs inference
 ↓
User clicks Export Session CSV
 ↓
ViewModel asks Repository to build CSV
 ↓
Repository loads Patient, Session, Measurements, Result from Room
 ↓
Repository creates CSV text
 ↓
UI opens Android save-file picker
 ↓
User chooses location
 ↓
ViewModel writes CSV to selected Uri
 ↓
App shows "Export saved"
```

This is a practical and realistic app workflow.

## 22. Current Architecture After Lesson 21

The architecture is now:

```text
Presentation layer
 ├── PatientListScreen
 ├── PatientDetailScreen
 ├── MeasurementScreen
 └── ResultScreen

ViewModel layer
 └── ResearchViewModel

Repository layer
 └── MeasurementRepository
      ├── buildSessionCsvExport()
      ├── runAndSaveInferenceForSession()
      ├── createMeasurementFromDevice()
      └── Room access

Data source layer
 └── DeviceDataSource

Processing layer
 └── SignalProcessor

ML layer
 └── ModelRunner

Storage layer
 └── Room database

Export layer
 └── CSV / JSON text written through Android document picker
```

This is now a complete research-app path:

- collect
- process
- infer
- store
- export
- analyse outside app

## 23. What You Learned in Lesson 21

The key idea is:

> Export complete research context, not only values.

A good session export should include:

- patient metadata
- session metadata
- measurement rows
- model result

The key repository function is:

```kotlin
suspend fun buildSessionCsvExport(
    sessionId: Long
): String
```

The key Android export tool is:

```kotlin
ActivityResultContracts.CreateDocument("text/csv")
```

which lets the user choose where to save a new CSV document. Android Developers
The key writing function is:

```kotlin
context.contentResolver.openOutputStream(uri)
```

The most important mental model is:

```text
Room keeps structured app data.
Export turns selected data into researcher-friendly files.
```

For a research app, export is not an afterthought. It is part of the data pipeline:

```text
Patient
 ↓
Session
 ↓
Measurements
 ↓
Processing
 ↓
Inference
 ↓
Result
 ↓
Export
 ↓
Python / Excel / statistics / reporting
```

## Lesson 22 Preview

In Lesson 22, we should step back and review the complete research app architecture.
We will connect all previous lessons into one clear mental model:

- Presentation layer
- Application/ViewModel layer
- Domain/data model
- Repository layer
- Device communication layer
- Processing layer
- ML inference layer
- Storage/export layer

This will help you see how the whole app fits together before moving into more advanced implementation details.
