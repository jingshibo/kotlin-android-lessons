# Lesson 33 — Testing the Whole Fake-Data Workflow

In Lesson 32, we added session CSV export.

The app now has all the major parts of the fake research workflow:

```text
Patient data
 ↓
Session data
 ↓
Fake device acquisition
 ↓
Signal processing
 ↓
Room database saving
 ↓
Fake ML inference
 ↓
Result saving
 ↓
CSV export
```

Lesson 33 is about testing the whole app as one connected system.

This corresponds to **Step A10** in Direction A:

```text
Test the complete fake-data workflow
```

After this lesson, Direction A is essentially complete.

---

## 1. What we are testing

We are not testing real Bluetooth yet.

We are not testing a real LiteRT/TFLite model yet.

We are testing whether the app skeleton works correctly with:

```text
FakeDeviceDataSource
FakeModelRunner
Room database
SignalProcessor
ExportFormatter
Compose screens
ResearchViewModel
Navigation
```

The expected user workflow is:

```text
Open app
 ↓
Create patient/sample
 ↓
Open patient
 ↓
Create session
 ↓
Open measurement screen
 ↓
Connect fake device
 ↓
Start acquisition
 ↓
Stop acquisition
 ↓
Run fake inference
 ↓
View result
 ↓
Export CSV
```

This is the first complete fake-data research app workflow.

---

## 2. Why testing the fake workflow matters

Before using real hardware or a real model, we need to confirm that the app structure works.

If the fake workflow does not work, real hardware will be much harder to debug.

For example, imagine you add real Bluetooth immediately and the app fails.

You may not know whether the problem is:

```text
Bluetooth connection
permissions
data parsing
repository logic
Room database
ViewModel state
navigation
UI button state
ML inference
export
```

That is too many possible problems.

By testing the fake workflow first, we prove that:

```text
UI works
navigation works
ViewModel works
repository works
Room saving works
processing works
fake inference works
export works
```

Then later, when you replace the fake device with real Bluetooth, you know the rest of the app is already reliable.

---

## 3. Expected project structure before testing

Before testing, your project should roughly look like this:

```text
com.example.researchapp
 ├── MainActivity.kt
 │
 ├── ui
 │    ├── ResearchApp.kt
 │    ├── PatientListScreen.kt
 │    ├── PatientDetailScreen.kt
 │    ├── MeasurementScreen.kt
 │    └── ResultScreen.kt
 │
 ├── viewmodel
 │    ├── ResearchViewModel.kt
 │    ├── ResearchUiState.kt
 │    └── AppStateEnums.kt
 │
 ├── data
 │    ├── ResearchDatabase.kt
 │    ├── MeasurementRepository.kt
 │    ├── entity
 │    │    ├── PatientEntity.kt
 │    │    ├── SessionEntity.kt
 │    │    ├── MeasurementEntity.kt
 │    │    └── ResultEntity.kt
 │    └── dao
 │         ├── PatientDao.kt
 │         ├── SessionDao.kt
 │         ├── MeasurementDao.kt
 │         └── ResultDao.kt
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
 └── export
      └── ExportFormatter.kt
```

This is the complete Direction A skeleton.

---

## 4. Test 1 — App opens correctly

The first test is simple.

Run the app.

You should see:

```text
Patient List
```

The screen should contain:

```text
New patient/sample code text field
Create Patient button
Existing patients section
```

At the beginning, if the database is empty, it may show:

```text
No patients yet
```

That is expected.

If the app crashes immediately, the likely problems are:

```text
MainActivity is not calling ResearchApp()
ResearchApp has a navigation setup problem
Navigation Compose dependency is missing
Room database setup has a dependency/import issue
ViewModel creation has a problem
```

At this stage, do not continue to hardware or ML testing.

First make sure the app can open.

---

## 5. Test 2 — Create a patient/sample

In the patient code field, enter:

```text
P001
```

or, for your textile/uniform example, something like:

```text
D1-ETO-W0-U1-S1
```

Then click:

```text
Create Patient
```

Expected result:

```text
Patient is created
Patient list updates
The new patient appears in the list
```

You should see a card or row like:

```text
Patient code: P001
Patient ID: 1
```

The exact ID may be different.

That is okay.

The important thing is:

```text
The patient appears after creation.
```

---

## 6. What this test checks

Creating a patient checks this path:

```text
PatientListScreen
 ↓
onCreatePatientClick
 ↓
ResearchViewModel.createPatient()
 ↓
MeasurementRepository.createPatient()
 ↓
PatientDao.insertPatient()
 ↓
Room database
 ↓
ViewModel.loadPatients()
 ↓
PatientListScreen updates
```

So one small user action tests many layers.

If this test fails, the problem may be in:

```text
TextField state
Create button callback
ViewModel function
Repository function
PatientDao
Room database
Patient list reload
UI state update
```

---

## 7. Common problem: button disabled

If the Create Patient button is disabled, check this logic in `PatientListScreen`:

```kotlin
Button(
    onClick = onCreatePatientClick,
    enabled = patientCode.isNotBlank()
) {
    Text("Create Patient")
}
```

The button is enabled only when:

```text
patientCode is not blank
```

So make sure the text field is connected to:

```kotlin
onPatientCodeChange = viewModel::updatePatientCode
```

and that the ViewModel function updates:

```kotlin
uiState.patientCode
```

If typing does not update state, the button may stay disabled.

---

## 8. Test 3 — Open patient detail

Click the patient card.

Expected result:

```text
Patient Detail screen opens
```

It should show:

```text
Patient Detail
Patient ID: ...
Patient code: P001
New session name field
Create Session button
Sessions section
```

If there are no sessions yet, it may show:

```text
No sessions yet
```

That is expected.

---

## 9. What this test checks

Opening a patient checks this path:

```text
PatientListScreen patient click
 ↓
navController.navigate("patient_detail/{patientId}")
 ↓
ResearchApp Patient Detail route
 ↓
viewModel.loadPatientDetail(patientId)
 ↓
MeasurementRepository.getPatientById()
 ↓
MeasurementRepository.getSessionsForPatient()
 ↓
PatientDetailScreen displays state
```

This confirms that navigation with `patientId` works.

---

## 10. Common problem: patient detail screen is blank

If the patient detail screen is blank, check whether the route argument is correct.

The route should look like:

```kotlin
composable(
    route = "${Routes.PATIENT_DETAIL}/{patientId}"
)
```

And navigation should look like:

```kotlin
navController.navigate(
    "${Routes.PATIENT_DETAIL}/$patientId"
)
```

These two must match.

If the route says:

```text
patient_detail/{id}
```

but navigation sends:

```text
patient_detail/1
```

and the code tries to read:

```kotlin
getString("patientId")
```

then the argument name does not match.

The name inside `{...}` and the name used in `getString(...)` must be the same.

---

## 11. Test 4 — Create a session

On the Patient Detail screen, enter a session name such as:

```text
Baseline
```

Then click:

```text
Create Session
```

Expected result:

```text
Session is created
Session list updates
New session appears
```

You should see something like:

```text
Session: Baseline
Session ID: 1
Status: Not ended
```

The exact session ID may be different.

---

## 12. What this test checks

Creating a session checks:

```text
PatientDetailScreen
 ↓
onCreateSessionClick
 ↓
ResearchViewModel.createSessionForCurrentPatient()
 ↓
MeasurementRepository.createSession()
 ↓
SessionDao.insertSession()
 ↓
Room database
 ↓
ViewModel.loadPatientDetail()
 ↓
Session list updates
```

It also checks that the selected patient ID is stored correctly in:

```kotlin
currentPatientId
```

If `currentPatientId` is null, the ViewModel cannot create a session.

---

## 13. Common problem: “No patient selected”

If you see:

```text
No patient selected
```

then this line is probably null:

```kotlin
val patientId = uiState.currentPatientId
```

That means `loadPatientDetail(patientId)` did not correctly update:

```kotlin
currentPatientId = patient.id
```

Check that your ViewModel includes this update:

```kotlin
uiState = uiState.copy(
    currentPatientId = patient.id,
    currentPatientCode = patient.patientCode,
    sessionItems = sessionItems,
    isLoading = false,
    message = ""
)
```

---

## 14. Test 5 — Open measurement screen

Click the session card.

Expected result:

```text
Measurement Screen opens
```

It should show:

```text
Measurement Screen
Session ID: ...
Device status: Disconnected
Acquisition status: Idle
Latest raw value: No data
Latest processed value: No data
Measurement count: 0
```

This is correct before acquisition starts.

---

## 15. What this test checks

Opening the measurement screen checks:

```text
PatientDetailScreen session click
 ↓
navController.navigate("measurement/{sessionId}")
 ↓
ResearchApp Measurement route
 ↓
viewModel.loadSession(sessionId)
 ↓
MeasurementRepository.getMeasurementsForSession()
 ↓
MeasurementRepository.getLatestResultForSession()
 ↓
MeasurementScreen displays state
```

This confirms that the app can load a session by ID.

---

## 16. Test 6 — Connect fake device

On the Measurement screen, click:

```text
Connect
```

Expected result:

```text
Device status changes to Connecting
then Connected
```

Because `FakeDeviceDataSource.connect()` uses:

```kotlin
delay(1000)
```

there should be a short delay.

After connection, the Start button should become enabled if a valid session exists.

---

## 17. What this test checks

Connecting the fake device checks:

```text
MeasurementScreen Connect button
 ↓
ResearchViewModel.connectDevice()
 ↓
MeasurementRepository.connectDevice()
 ↓
FakeDeviceDataSource.connect()
 ↓
deviceConnectionState = CONNECTED
 ↓
MeasurementScreen updates
```

This confirms that the fake device source is connected to the ViewModel correctly.

---

## 18. Common problem: Start button still disabled

The Start button depends on:

```kotlin
fun canStartAcquisition(): Boolean {
    return uiState.currentSessionId != null &&
        uiState.deviceConnectionState == DeviceConnectionState.CONNECTED &&
        uiState.acquisitionState != AcquisitionState.RECORDING
}
```

So if Start is disabled, check:

```text
Is currentSessionId not null?
Is deviceConnectionState CONNECTED?
Is acquisitionState not RECORDING?
```

The most common issue is:

```text
currentSessionId was not set when loading the measurement screen.
```

In `loadSession(sessionId)`, make sure you have:

```kotlin
uiState = uiState.copy(
    currentSessionId = sessionId,
    isLoading = true,
    message = "Loading session..."
)
```

---

## 19. Test 7 — Start acquisition

Click:

```text
Start
```

Expected result:

```text
Acquisition status becomes Recording
Measurement values appear repeatedly
Measurement count increases
Latest raw value updates
Latest processed value updates
Latest status updates
```

Because the fake device reads one value every second, the screen should update gradually.

Example display:

```text
Latest raw value: 3.284
Latest processed value: 3.084
Latest status: OK
Measurement count: 1
```

Then:

```text
Latest raw value: 1.732
Latest processed value: 1.532
Latest status: OK
Measurement count: 2
```

---

## 20. What this test checks

Starting acquisition checks the most important path:

```text
MeasurementScreen Start button
 ↓
ResearchViewModel.startAcquisition()
 ↓
MeasurementRepository.readAndSaveMeasurement()
 ↓
FakeDeviceDataSource.readValue()
 ↓
SignalProcessor.baselineCorrect()
 ↓
SignalProcessor.isValidValue()
 ↓
MeasurementEntity created
 ↓
MeasurementDao.insertMeasurement()
 ↓
Room database
 ↓
ViewModel updates uiState
 ↓
MeasurementScreen updates
```

This is the core data-acquisition pipeline.

If this works, your architecture is functioning.

---

## 21. Common problem: “Device is not connected”

If acquisition fails with an error related to:

```text
Device is not connected
```

then `readValue()` was called before `connect()` finished.

Check that:

```text
You clicked Connect first.
Device status became Connected.
Start button is only enabled when connected.
```

The fake device protects us from reading when disconnected:

```kotlin
if (!connected) {
    throw IllegalStateException(
        "Device is not connected"
    )
}
```

That is good behaviour.

The UI should prevent this situation, but the device layer also protects itself.

---

## 22. Common problem: acquisition does not stop

The acquisition loop uses:

```kotlin
while (uiState.acquisitionState == AcquisitionState.RECORDING) {
    ...
}
```

When the Stop button is clicked, `stopAcquisition()` should set:

```kotlin
acquisitionState = AcquisitionState.STOPPED
```

Then the loop should stop.

If the loop does not stop, check that `stopAcquisition()` is actually connected:

```kotlin
onStopAcquisitionClick = viewModel::stopAcquisition
```

Also check that the Stop button is enabled only when recording:

```kotlin
canStopAcquisition = viewModel.canStopAcquisition()
```

---

## 23. Test 8 — Stop acquisition

After collecting several measurements, click:

```text
Stop
```

Expected result:

```text
Acquisition status changes to Stopped
Measurement count stops increasing
Session is marked as ended
Run Inference button becomes available
```

This also calls:

```kotlin
measurementRepository.endSession(
    sessionId = sessionId
)
```

So the session’s `endedAt` field should be updated.

---

## 24. What this test checks

Stopping acquisition checks:

```text
MeasurementScreen Stop button
 ↓
ResearchViewModel.stopAcquisition()
 ↓
acquisitionState = STOPPED
 ↓
MeasurementRepository.endSession()
 ↓
SessionDao.endSession()
 ↓
Room updates endedAt
```

This confirms that the app can close a session properly.

---

## 25. Test 9 — Run fake inference

After collecting at least one valid measurement, click:

```text
Run Inference
```

Expected result:

```text
Message: Running inference...
then
Message: Inference complete
```

The app should produce a fake result:

```text
Positive
```

or:

```text
Negative
```

depending on the mean of recent processed values.

The fake rule is:

```text
mean > 2.5
 ↓
Positive

mean <= 2.5
 ↓
Negative
```

---

## 26. What this test checks

Running inference checks:

```text
MeasurementScreen Run Inference button
 ↓
ResearchViewModel.runInferenceForCurrentSession()
 ↓
MeasurementRepository.runAndSaveInferenceForSession()
 ↓
MeasurementRepository.extractFeaturesForSession()
 ↓
SignalProcessor.extractFeatures()
 ↓
FakeModelRunner.runInference()
 ↓
ResultDao.insertResult()
 ↓
latestPrediction updated
```

This confirms that processing, feature extraction, fake ML, and result saving work together.

---

## 27. Common problem: “Not enough data for inference”

If you see:

```text
Not enough data for inference
```

it means:

```text
extractFeaturesForSession() returned null
```

The most likely reason is:

```text
There are no valid measurements.
```

If your `getRecentProcessedValuesForSession()` filters by:

```kotlin
.filter { it.status == "OK" }
```

then invalid measurements are excluded.

Check the latest status on the Measurement screen.

If many values are invalid, your baseline or valid range may be too strict.

For the fake app, valid values should usually exist because the fake source returns:

```text
0.0 to 5.0
```

and the baseline correction subtracts:

```text
0.2
```

So only raw values below `0.2` become negative and invalid.

---

## 28. Test 10 — View result

Click:

```text
View Result
```

Expected result:

```text
Result Screen opens
Prediction is shown
Confidence is shown
Measurement count is shown
```

Example:

```text
Prediction: Positive
Confidence: 0.90
Measurement count: 8
```

This confirms the result is carried through UI state.

---

## 29. What this test checks

Viewing the result checks:

```text
MeasurementScreen View Result button
 ↓
navController.navigate("result/{sessionId}")
 ↓
ResearchApp Result route
 ↓
viewModel.loadSession(sessionId)
 ↓
latest result loaded from Room
 ↓
ResultScreen displays prediction
```

This confirms that saved results can be loaded again.

---

## 30. Test 11 — Export CSV

On the Result screen, click:

```text
Export Session CSV
```

Expected result:

```text
Android file picker opens
Suggested filename appears
User chooses location
CSV file is saved
Message becomes Export complete
```

The filename may look like:

```text
P001_Baseline_session_1.csv
```

The CSV should include:

```text
patient metadata
session metadata
latest result
measurement table
```

---

## 31. What this test checks

Export checks:

```text
ResultScreen Export button
 ↓
ResearchViewModel.prepareSessionCsvExport()
 ↓
MeasurementRepository.buildSessionCsvExport()
 ↓
ExportFormatter.buildSessionCsv()
 ↓
pendingExportText stored in uiState
 ↓
CreateDocument launcher opens
 ↓
writeTextToUri()
 ↓
CSV file saved
```

This confirms that the export layer is connected.

---

## 32. Example exported CSV

The file may look like this:

```csv
Patient Code,P001
Patient ID,1
Session ID,1
Session Name,Baseline
Started At,1787770000000
Ended At,1787770060000
Result Label,Positive
Result Confidence,0.9

repetition,timestamp,rawValue,processedValue,status
1,1787770001000,2.438,2.238,OK
2,1787770002000,2.562,2.362,OK
3,1787770003000,0.100,-0.100,INVALID
```

This is a good research export because it includes both:

```text
context
```

and:

```text
measurement data
```

The app is not only exporting final predictions.

It is exporting the trace behind the prediction.

---

## 33. Full fake workflow checklist

Use this checklist to test the app manually.

```text
[ ] App opens to Patient List screen
[ ] Create Patient button works
[ ] New patient appears in patient list
[ ] Clicking patient opens Patient Detail screen
[ ] Create Session button works
[ ] New session appears in session list
[ ] Clicking session opens Measurement screen
[ ] Connect button changes device status to Connected
[ ] Start button starts acquisition
[ ] Measurement count increases
[ ] Raw value updates
[ ] Processed value updates
[ ] Status updates
[ ] Stop button stops acquisition
[ ] Run Inference button creates prediction
[ ] View Result opens Result screen
[ ] Result screen shows prediction and confidence
[ ] Export button opens file picker
[ ] CSV file is saved
[ ] CSV contains patient, session, result, and measurements
```

If all of these pass, Direction A is successful.

---

## 34. Layer-by-layer testing strategy

If something fails, do not panic.

Test one layer at a time.

The layers are:

```text
UI
 ↓
ViewModel
 ↓
Repository
 ↓
Room / Device / Processing / ML / Export
```

Use this debugging question:

```text
Which layer is the first layer where the expected data disappears?
```

For example:

```text
Did the button click happen?
Did the ViewModel function run?
Did the repository function run?
Did the DAO insert happen?
Did uiState update?
Did the screen recompose?
```

This is much better than randomly changing code.

---

## 35. Debugging with simple messages

During learning, you can use `message` in `ResearchUiState` as a simple debug display.

For example:

```kotlin
uiState = uiState.copy(
    message = "Patient created"
)
```

or:

```kotlin
uiState = uiState.copy(
    message = "Measurement saved"
)
```

Then display the message on the screen.

This helps you see which part of the app is running.

Later, you can use proper logs, snackbars, or error screens.

But for now, a message field is simple and useful.

---

## 36. Debugging with logs

You can also add Android logs.

For example:

```kotlin
import android.util.Log
```

Then:

```kotlin
Log.d(
    "ResearchApp",
    "Measurement saved: raw=${measurement.rawValue}, processed=${measurement.processedValue}"
)
```

In Android Studio, check Logcat.

Useful places to add logs:

```text
createPatient()
createSessionForCurrentPatient()
connectDevice()
startAcquisition()
readAndSaveMeasurement()
runInferenceForCurrentSession()
prepareSessionCsvExport()
```

Do not rely on logs as the final app interface.

Use them for debugging.

---

## 37. Common compile issue: Room imports

Earlier lessons used Room 3-style imports:

```kotlin
import androidx.room3.Entity
import androidx.room3.PrimaryKey
import androidx.room3.Dao
import androidx.room3.Insert
import androidx.room3.Query
import androidx.room3.Database
import androidx.room3.RoomDatabase
import androidx.room3.Room
```

If your project uses Room 2 instead, the imports are different:

```kotlin
import androidx.room.Entity
import androidx.room.PrimaryKey
import androidx.room.Dao
import androidx.room.Insert
import androidx.room.Query
import androidx.room.Database
import androidx.room.RoomDatabase
import androidx.room.Room
```

The rule is:

```text
Your imports must match your Gradle Room dependency.
```

Do not mix:

```text
androidx.room3
```

and:

```text
androidx.room
```

in the same project unless you really know what you are doing.

---

## 38. Common compile issue: missing Navigation dependency

If Android Studio cannot find:

```kotlin
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
```

then the Navigation Compose dependency is missing.

The app needs Navigation Compose to use:

```text
NavHost
composable routes
navController.navigate()
```

Without that dependency, `ResearchApp.kt` will not compile.

---

## 39. Common compile issue: missing lifecycle ViewModel Compose dependency

If Android Studio cannot find:

```kotlin
import androidx.lifecycle.viewmodel.compose.viewModel
```

then the lifecycle ViewModel Compose dependency may be missing.

That import is used here:

```kotlin
@Composable
fun ResearchApp(
    viewModel: ResearchViewModel = viewModel()
)
```

Without it, Compose cannot create the ViewModel in that way.

---

## 40. Common runtime issue: database schema changed

If you changed entity fields while developing, Room may complain about schema mismatch.

For example, you may see errors after changing:

```text
MeasurementEntity
SessionEntity
ResultEntity
```

In a learning project, the simplest fix is often:

```text
uninstall the app from the emulator/tablet
reinstall the app
```

This deletes the old database and creates a fresh one.

However, in a real research deployment, do not casually delete the app because collected data may be lost.

For real use, you need:

```text
Room migrations
backup/export before schema changes
careful version control
```

For the tutorial skeleton, reinstalling during development is acceptable.

---

## 41. Common runtime issue: database on main thread

Room database work should not run on the UI thread.

That is why our DAO and repository functions are mostly:

```kotlin
suspend fun
```

and the ViewModel calls them inside:

```kotlin
viewModelScope.launch {
    ...
}
```

Avoid calling repository suspend functions directly from Composable functions.

The correct path is:

```text
Composable
 ↓
callback
 ↓
ViewModel function
 ↓
viewModelScope.launch
 ↓
Repository suspend function
```

---

## 42. Common logic issue: repeated loading with `LaunchedEffect`

In `ResearchApp.kt`, we used:

```kotlin
LaunchedEffect(Unit) {
    viewModel.loadPatients()
}
```

or:

```kotlin
LaunchedEffect(sessionId) {
    viewModel.loadSession(sessionId)
}
```

This is usually okay.

But be careful not to put loading calls directly in the composable body without `LaunchedEffect`.

Avoid this:

```kotlin
composable(Routes.PATIENT_LIST) {
    viewModel.loadPatients()

    PatientListScreen(...)
}
```

That can trigger repeated loading during recomposition.

Use:

```kotlin
LaunchedEffect(Unit) {
    viewModel.loadPatients()
}
```

so the load happens at the right time.

---

## 43. Common logic issue: acquisition loop updates too frequently

Our fake device reads every second because `FakeDeviceDataSource.readValue()` contains:

```kotlin
delay(1000)
```

So the loop is naturally slow.

But if you later use a real device that streams very fast, you should not update UI and Room for every tiny sample without thinking.

For real high-rate data, you may need:

```text
buffering
windowing
batch saving
background service
downsampling for live display
separate raw-data file storage
```

But this is not needed for the fake Direction A skeleton.

---

## 44. Common logic issue: repetition number

In Lesson 31, repetition was calculated as:

```kotlin
val repetition = uiState.measurements.size + 1
```

This is fine for the fake beginner workflow.

But in a real app, you may want a clearer measurement index.

Possible future options:

```text
sampleIndex
windowIndex
repetitionNumber
insertionNumber
trialNumber
```

For your textile measurement protocol, for example, repetition may have a specific meaning:

```text
5 repetitions per insertion
10 repetitions per patch
two faces
```

So later, the data model may need fields such as:

```text
sampleId
face
insertionIndex
repetitionIndex
deviceId
insecticide
washCount
uniformId
```

For now, repetition is only a simple placeholder.

---

## 45. Common logic issue: one value is too simple

Our current `MeasurementEntity` stores:

```kotlin
val rawValue: Double
val processedValue: Double
```

This is intentionally simple.

A real sensor may produce:

```text
multiple channels
frequency points
time-series windows
complex values
image-like data
metadata
```

For example, your research app may later store:

```text
VNA frequency response
sensor array values
multi-channel textile readings
time-series windows
classification features
```

So this beginner app does not yet represent your final data structure.

Its purpose is to teach the architecture.

Once the app structure is clear, the measurement model can be expanded.

---

## 46. What Direction A has achieved

Direction A started with this goal:

```text
Build a clean Android project from the architecture.
```

We have now built the clean skeleton:

```text
Project structure
Core entities
Room DAOs
Room database
Repository
Fake device source
SignalProcessor
FakeModelRunner
Compose screens
Navigation and ViewModel
CSV export
End-to-end fake workflow test
```

The app is not final.

But it has the correct shape.

This is much better than starting with one huge `MainActivity.kt`.

---

## 47. Final architecture after Direction A

The final Direction A architecture is:

```text
MainActivity
 ↓
ResearchApp
 ↓
NavHost
 ↓
PatientListScreen
PatientDetailScreen
MeasurementScreen
ResultScreen
 ↓
ResearchViewModel
 ↓
MeasurementRepository
 ├── ResearchDatabase
 │    ├── PatientDao
 │    ├── SessionDao
 │    ├── MeasurementDao
 │    └── ResultDao
 │
 ├── DeviceDataSource
 │    └── FakeDeviceDataSource
 │
 ├── SignalProcessor
 │    └── SignalFeatures
 │
 ├── ModelRunner
 │    └── FakeModelRunner
 │
 └── ExportFormatter
```

The end-to-end data flow is:

```text
User creates patient
 ↓
PatientEntity saved to Room

User creates session
 ↓
SessionEntity saved to Room

User starts acquisition
 ↓
FakeDeviceDataSource generates raw value
 ↓
SignalProcessor creates processed value and status
 ↓
MeasurementEntity saved to Room

User runs inference
 ↓
Repository loads valid processed values
 ↓
SignalProcessor extracts features
 ↓
FakeModelRunner creates PredictionResult
 ↓
ResultEntity saved to Room

User exports session
 ↓
Repository loads patient, session, measurements, result
 ↓
ExportFormatter creates CSV
 ↓
Android file picker saves file
```

This is a complete fake-data research app pipeline.

---

## 48. What you learned in Lesson 33

You learned how to test the complete fake workflow:

```text
create patient
create session
connect fake device
collect processed measurements
stop acquisition
run fake inference
view result
export CSV
```

You also learned how to debug the app layer by layer:

```text
UI
ViewModel
Repository
Room
Device
Processing
ML
Export
```

The most important mental model is:

```text
Do not debug the whole app blindly.
Follow the data path layer by layer.
```

When something fails, ask:

```text
Did the screen call the callback?
Did the ViewModel function run?
Did the repository function run?
Did the DAO/database operation work?
Did uiState update?
Did the screen display the new state?
```

This is the correct way to debug a structured Android research app.

---

## 49. Direction A summary

Direction A is now complete at the tutorial level.

You have learned how to turn the architecture from Lessons 1–22 into a clean fake-data Android app skeleton.

The app now has:

```text
a research data model
local Room storage
fake acquisition
simple signal processing
fake inference
result saving
CSV export
multi-screen UI
ViewModel-based app flow
```

This gives you a foundation that can later be upgraded.

---

## 50. Next possible direction

The next tutorial direction should be Direction B, where we start replacing the fake parts with more realistic implementations.

A good next lesson would be:

```text
Lesson 34 — Preparing the Project for Real Device Communication
```

or:

```text
Lesson 34 — From Fake Device to Real Bluetooth/Wi-Fi Device Source
```

That would begin the transition from:

```text
FakeDeviceDataSource
```

to:

```text
BluetoothDeviceDataSource
```

or:

```text
WifiDeviceDataSource
```

But the important point is:

```text
We now have a clean app structure to plug real hardware into.
```