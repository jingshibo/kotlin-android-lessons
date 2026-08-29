# Lesson 22 — Final Research App Architecture Review

In Lesson 21, we completed the main research-app workflow:

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
 ↓
Export
```

Now Lesson 22 is a review lesson.

The goal is to connect everything we have built so far into one clear mental model.

This is important because the tutorial started with a focused idea: do not learn all Kotlin/Android theory first; instead, learn the small subset needed to build a practical research app. fileciteturn0file0L6-L8

Now we can see the full path.

---

## 1. What kind of app are we building?

We are not building a generic calculator or to-do app.

We are building a research tablet app that can eventually:

```text
manage patients/samples
create measurement sessions
connect to a device
collect live data
process sensor values
run ML inference
save structured data locally
export research files
show results to the user
```

So the app is both:

```text
a data-collection app
```

and:

```text
an edge-AI research app
```

The full app flow is:

```text
Open app
 ↓
Select or create patient/sample
 ↓
Create session
 ↓
Request permissions
 ↓
Connect device
 ↓
Start acquisition
 ↓
Collect raw values
 ↓
Process values
 ↓
Save measurements
 ↓
Stop acquisition
 ↓
Run ML inference
 ↓
Save result
 ↓
Export CSV/JSON
```

This is the main path we have been building.

---

## 2. The complete architecture

At the highest level, the app can be divided like this:

```text
Presentation Layer
 ↓
Application / ViewModel Layer
 ↓
Domain / Data Model Layer
 ↓
Repository Layer
 ↓
Device / Processing / ML Layer
 ↓
Storage and Export Layer
```

More concretely:

```text
PatientListScreen
PatientDetailScreen
MeasurementScreen
ResultScreen
        ↓
ResearchViewModel
        ↓
MeasurementRepository
        ↓
DeviceDataSource
SignalProcessor
ModelRunner
Room DAOs
        ↓
Room database
CSV / JSON export
```

This is the whole architecture in one view.

---

## 3. Presentation Layer

The Presentation Layer is the UI.

In our app, this includes:

```text
PatientListScreen
PatientDetailScreen
MeasurementScreen
ResultScreen
```

Its job is to:

```text
display state
show buttons and input fields
send user actions to the ViewModel
show messages, values, and results
```

It should not directly:

```text
read from Bluetooth
write to Room
run ML inference
parse raw device messages
perform heavy signal processing
```

A good UI screen should mostly say:

```text
Here is the current state.
Here are the actions the user can take.
```

For example:

```kotlin
Button(
    onClick = {
        viewModel.startAcquisition()
    },
    enabled =
        uiState.deviceConnectionState == DeviceConnectionState.CONNECTED &&
        uiState.acquisitionState != AcquisitionState.RECORDING
) {
    Text("Start Acquisition")
}
```

The button does not know how acquisition works internally.

It only calls:

```kotlin
viewModel.startAcquisition()
```

That is the correct idea.

---

## 4. Application / ViewModel Layer

The ViewModel controls the screen state and user flow.

The original tutorial already introduced ViewModel because putting all state directly inside one composable becomes messy as the app grows. fileciteturn1file0L323-L339

In our app, the ViewModel manages:

```text
current patient/session IDs
device connection state
acquisition state
permission state
latest raw/processed values
latest prediction
loading state
user messages
```

Example state:

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

    val latestRawValue: Double? = null,
    val latestProcessedValue: Double? = null,
    val latestStatus: String = "",

    val latestPrediction: PredictionResult? = null,

    val isLoading: Boolean = false,
    val message: String = ""
)
```

The ViewModel answers questions like:

```text
Can acquisition start?
Is there an active session?
Is the device connected?
Should the Start button be enabled?
What message should the user see?
```

But it should not contain low-level details like:

```text
SQL queries
Bluetooth socket code
model interpreter code
CSV row construction
```

Those belong deeper in the app.

---

## 5. Domain / Data Model Layer

The Domain/Data Model Layer defines the important objects in the app.

For our research app, the main model is:

```text
Patient
 ↓
Session
 ↓
Measurement
 ↓
Result
```

In code, this became:

```kotlin
PatientEntity
SessionEntity
MeasurementEntity
ResultEntity
```

The important relationship is:

```text
Patient has many Sessions.
Session has many Measurements.
Session has Results.
```

So the database model is:

```text
PatientEntity.id
 ↓
SessionEntity.patientId

SessionEntity.id
 ↓
MeasurementEntity.sessionId
ResultEntity.sessionId
```

This is one of the most important ideas in the whole tutorial.

A measurement should not be saved alone.

A measurement should belong to a session.

A session should belong to a patient/sample.

That protects the research context.

---

## 6. Repository Layer

The Repository Layer is the bridge between the ViewModel and the lower-level data systems.

We introduced it because the ViewModel was starting to do too much.

The repository handles:

```text
creating patients and sessions
reading values from the data source
processing values
saving measurements
loading measurements
running inference
saving results
building export text
```

A simplified repository role:

```text
ResearchViewModel
 ↓
asks for data/action

MeasurementRepository
 ↓
knows where data comes from and where it goes
```

For example:

```kotlin
suspend fun runAndSaveInferenceForSession(
    sessionId: Long
): PredictionResult? {
    val prediction = runInferenceForSession(
        sessionId = sessionId
    ) ?: return null

    val resultEntity = ResultEntity(
        sessionId = sessionId,
        label = prediction.label,
        confidence = prediction.confidence,
        createdAt = System.currentTimeMillis()
    )

    resultDao.insertResult(resultEntity)

    return prediction
}
```

The ViewModel does not need to know how the prediction is generated or saved.

It simply asks the repository to do it.

---

## 7. Device Data Source Layer

The Device Data Source Layer is responsible for getting data from a device.

We created:

```kotlin
interface DeviceDataSource {
    suspend fun connect()
    suspend fun disconnect()
    suspend fun readValue(): Double
}
```

Then we used:

```kotlin
FakeDeviceDataSource
```

for development.

Later, we can replace it with:

```text
BluetoothDeviceDataSource
WifiDeviceDataSource
UsbDeviceDataSource
```

The important idea is:

```text
Fake data and real data should enter through the same doorway.
```

That doorway is:

```text
DeviceDataSource
```

So the rest of the app does not care whether the value came from:

```text
random fake data
Bluetooth
Wi-Fi
USB
internal sensor
```

It only receives a value.

---

## 8. Processing Layer

The Processing Layer turns raw sensor data into cleaner or more meaningful data.

We created:

```kotlin
class SignalProcessor {
    ...
}
```

It can handle:

```text
baseline correction
validity checking
moving average
feature extraction
```

The processing flow is:

```text
raw value
 ↓
baseline correction
 ↓
processed value
 ↓
validity check
 ↓
features
```

A simple example:

```kotlin
val processedValue = signalProcessor.baselineCorrect(
    rawValue = rawValue,
    baseline = baseline
)
```

For a research app, this layer is very important.

The final result depends not only on the ML model, but also on:

```text
how raw data was cleaned
how baseline was removed
which features were extracted
which values were marked invalid
```

So processing should not be hidden randomly inside UI code.

It deserves its own place.

---

## 9. ML Inference Layer

The ML layer runs a trained model on the tablet.

We introduced:

```kotlin
interface ModelRunner {
    suspend fun runInference(
        features: SignalFeatures
    ): PredictionResult
}
```

Then we used:

```kotlin
FakeModelRunner
```

first.

Later, this can become:

```text
LiteRtModelRunner
```

or a real `.tflite` model runner.

The ML flow is:

```text
processed measurements
 ↓
feature extraction
 ↓
FloatArray input
 ↓
model inference
 ↓
prediction label
 ↓
confidence score
 ↓
ResultEntity
```

A very important warning from Lesson 20 was:

```text
Android preprocessing must match Python training preprocessing.
```

That includes:

```text
feature order
normalisation
input shape
data type
label order
model version
```

If these do not match, the app may run successfully but produce wrong predictions.

---

## 10. Storage Layer

The Storage Layer stores structured app data locally.

We introduced Room because simple file storage becomes inconvenient when the app has:

```text
patients
sessions
measurements
results
```

Room stores:

```text
PatientEntity
SessionEntity
MeasurementEntity
ResultEntity
```

The DAOs provide database access:

```text
PatientDao
SessionDao
MeasurementDao
ResultDao
```

The original tutorial had already introduced internal storage as a beginner-friendly auto-save method, but later warned that heavier file writing should not block the UI thread. fileciteturn1file0L397-L405 fileciteturn1file0L428-L435

Room improves the app because we can insert and query structured data instead of saving the whole measurement list manually each time.

The storage path is:

```text
MeasurementRepository
 ↓
DAO
 ↓
Room database
 ↓
local SQLite database file
```

---

## 11. Export Layer

The Export Layer turns internal app data into researcher-friendly files.

The original tutorial already introduced simple CSV export in Lesson 7. fileciteturn1file0L270-L306

Now our export is richer.

It should include:

```text
patient metadata
session metadata
measurements
processed values
result label
result confidence
timestamps
```

The export flow is:

```text
Room database
 ↓
Repository loads session data
 ↓
Repository builds CSV or JSON text
 ↓
UI opens Android save-file picker
 ↓
ViewModel writes text to selected Uri
```

This keeps responsibilities separated:

```text
Repository builds export content.
UI lets user choose where to save.
ViewModel coordinates the action.
```

CSV is good for:

```text
Excel
Python pandas
R
MATLAB
```

JSON is good for:

```text
nested structured export
metadata preservation
complete session records
```

---

## 12. The full data pipeline

Now we can describe the complete pipeline:

```text
DeviceDataSource
 ↓
raw value

SignalProcessor
 ↓
processed value
features

ModelRunner
 ↓
prediction

Room database
 ↓
Patient / Session / Measurement / Result

Export
 ↓
CSV / JSON file

Python / Excel / analysis
```

Or from the user’s perspective:

```text
Select patient
 ↓
Create session
 ↓
Connect device
 ↓
Collect data
 ↓
Stop acquisition
 ↓
Run inference
 ↓
View result
 ↓
Export data
```

This is the complete main path.

---

## 13. How the layers communicate

A good architecture has one-directional communication.

For our app:

```text
Screen
 ↓
ViewModel
 ↓
Repository
 ↓
DeviceDataSource / SignalProcessor / ModelRunner / Room
```

Avoid this:

```text
Screen directly talks to Room.
Screen directly reads Bluetooth.
Screen directly runs model inference.
Screen directly builds CSV files.
```

Why?

Because then the UI becomes too complicated.

A cleaner screen is easier to change.

A cleaner repository is easier to test.

A cleaner data source is easier to replace.

---

## 14. Which layer owns which responsibility?

A simple table:

| Layer | Owns | Should avoid |
|---|---|---|
| UI / Screen | Display and user actions | Bluetooth, SQL, ML code |
| ViewModel | UI state and app flow | Low-level storage/device details |
| Repository | Data coordination | UI layout code |
| DeviceDataSource | Device communication | Compose UI state |
| SignalProcessor | Processing and features | Android UI logic |
| ModelRunner | ML inference | Patient/session navigation |
| Room / DAO | Local database operations | Business workflow |
| Export | CSV/JSON formatting | Live acquisition control |

This is the mental model you should keep.

---

## 15. Directory structure idea

At this stage, your project could be organised like this:

```text
com.example.researchapp
 ├── ui
 │    ├── PatientListScreen.kt
 │    ├── PatientDetailScreen.kt
 │    ├── MeasurementScreen.kt
 │    ├── ResultScreen.kt
 │    └── ResearchApp.kt
 │
 ├── viewmodel
 │    └── ResearchViewModel.kt
 │
 ├── data
 │    ├── MeasurementRepository.kt
 │    ├── ResearchDatabase.kt
 │    ├── PatientDao.kt
 │    ├── SessionDao.kt
 │    ├── MeasurementDao.kt
 │    ├── ResultDao.kt
 │    ├── PatientEntity.kt
 │    ├── SessionEntity.kt
 │    ├── MeasurementEntity.kt
 │    └── ResultEntity.kt
 │
 ├── device
 │    ├── DeviceDataSource.kt
 │    ├── FakeDeviceDataSource.kt
 │    └── BluetoothDeviceDataSource.kt
 │
 ├── processing
 │    ├── SignalProcessor.kt
 │    └── SignalFeatures.kt
 │
 ├── ml
 │    ├── ModelRunner.kt
 │    ├── FakeModelRunner.kt
 │    └── LiteRtModelRunner.kt
 │
 └── export
      └── ExportFormatter.kt
```

Do not worry if your current project is not organised like this yet.

This is a target structure.

At the beginning, you may keep more files together.

As the app grows, this structure becomes helpful.

---

## 16. What should be kept flexible?

Some parts of the app are likely to change.

For example:

```text
fake data source → Bluetooth data source
simple baseline correction → more advanced preprocessing
fake model → real LiteRT model
single measurement value → multi-channel signal
one-session CSV export → full patient export
one ViewModel → multiple ViewModels
```

So we designed the app with replaceable parts:

```text
DeviceDataSource
 ↓
replace fake device with real device

SignalProcessor
 ↓
replace simple processing with real algorithm

ModelRunner
 ↓
replace fake model with real model

Repository
 ↓
change storage/export details without changing UI
```

This is the main reason for architecture.

Architecture is not just “clean code”.

It protects the app from becoming impossible to maintain when the research project changes.

---

## 17. What is still simplified?

Even after Lesson 22, the app is still a learning version.

Important simplifications include:

```text
fake device data
fake model inference
simple one-value measurements
simple processing
basic Room structure
basic CSV export
one main ViewModel
minimal error handling
no dependency injection
no testing
no real Bluetooth/Wi-Fi implementation yet
```

This is okay.

The purpose of these lessons is to build the main path first.

A good learning strategy is:

```text
first make the app understandable
then make it realistic
then make it robust
then make it production-quality
```

Trying to build the production-quality version first would be overwhelming.

---

## 18. What a future advanced version would add

Later, the app could add:

```text
real Bluetooth or Wi-Fi communication
device selection screen
runtime permission checking by Android version
session templates
patient search
Room relationships and migrations
streaming/batched saving
background services for long acquisition
model version tracking
processing version tracking
JSON export
data backup
unit tests
UI tests
dependency injection
error logging
calibration workflow
```

But those should come after the main app path is clear.

---

## 19. The most important research-app rules

If you remember only the main principles, remember these:

```text
1. Do not collect data without context.
```

Every measurement should belong to a session.

Every session should belong to a patient/sample.

```text
2. Do not put hardware code in the UI.
```

Use a data source layer.

```text
3. Do not put processing randomly inside buttons.
```

Use a processing class.

```text
4. Do not treat ML as just a button.
```

ML inference should be part of a controlled pipeline.

```text
5. Do not only save final results.
```

Save raw data, processed data, timestamps, and context when possible.

```text
6. Do not rely only on internal app storage.
```

Export data in a researcher-friendly format.

```text
7. Do not block the UI.
```

Use coroutines/background work for file, database, device, and inference operations.

The original tutorial previewed coroutines/background work for exactly this reason: real research apps should not block the UI during saving, acquisition, ML inference, or device communication. fileciteturn1file0L447-L450

---

## 20. What you learned in Lesson 22

You learned how all previous lessons fit together:

```text
Kotlin basics
 ↓
Compose UI
 ↓
state management
 ↓
ViewModel
 ↓
persistence
 ↓
coroutines
 ↓
live acquisition
 ↓
device state
 ↓
repository
 ↓
Room database
 ↓
patient/session model
 ↓
navigation
 ↓
permissions/device abstraction
 ↓
signal processing
 ↓
ML inference
 ↓
export
 ↓
complete architecture
```

The final mental model is:

```text
UI shows the research workflow.

ViewModel controls app state.

Repository coordinates data operations.

DeviceDataSource provides raw data.

SignalProcessor prepares data.

ModelRunner produces predictions.

Room stores structured records.

Export converts records into research files.
```

This is the complete main path for building your Android research app.

---

# Next step

After Lesson 22, the core tutorial path is complete.

From here, there are two possible directions:

```text
Direction A:
start implementing a clean project from scratch using this architecture

Direction B:
continue advanced lessons, such as real Bluetooth/Wi-Fi, Room relationships, LiteRT integration, testing, or dependency injection
```

For your research app, the most practical next advanced lesson would be:

```text
Lesson 23 — Turning the Architecture into a Clean Android Project Structure
```

That would take the architecture from this lesson and map it into real files, packages, and implementation order.
