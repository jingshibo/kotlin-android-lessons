# Kotlin and Android Research App Tutorial Summary

## Big picture

This tutorial is easier to understand if it is split into two parts:

```text
Part 1: Lessons 1-22
-> Foundation, concepts, and architecture theory

Part 2: Lessons 23-33
-> Implementation of the clean fake-data research app skeleton
```

So yes: **Lessons 1-22 are mainly the basic theory and learning foundation**, while **Lessons 23-33 are the implementation phase**.

The order is logical:

```text
-> learn Kotlin and Android basics
-> understand the research-app workflow
-> design the architecture
-> create the project files
-> implement the fake-data workflow
-> test the full skeleton
```

## Complete roadmap

The complete learning route is:

```text
Part 1:
-> basic Kotlin
-> basic Android UI
-> research measurement screen
-> measurement history
-> CSV export
-> ViewModel architecture
-> simple internal autosave
-> background work
-> live data stream
-> app states
-> repository
-> Room database
-> patient/session model
-> navigation
-> permissions/device communication
-> real data source
-> signal processing
-> ML inference
-> complete export
-> architecture review

Part 2: 
-> clean project structure
-> data model files
-> Room DAO/database files
-> repository implementation
-> fake device implementation
-> signal-processing implementation
-> fake ML implementation
-> Compose screen implementation
-> navigation/ViewModel wiring
-> session CSV export implementation
-> complete fake workflow test
```

This route can be understood in two parts:

```text
Part 1: Lessons 1-22
-> basic Kotlin
-> Android UI
-> research app concepts
-> state, storage, repository, Room, navigation, device, processing, ML, export
-> final architecture review

Part 2: Lessons 23-33
-> clean project structure
-> concrete Kotlin files
-> fake device, processing, fake ML
-> screens, navigation, export
-> full fake-data workflow test
```

So the relationship is:

```text
Lessons 1-22 explain the route.
Lessons 23-33 start building the route.
```


## Overall purpose

This tutorial is a practical Kotlin and Android learning path for building a research data-collection app.

The goal is not to learn all Kotlin or all Android theory first. The tutorial teaches only the parts needed to gradually build a useful tablet-style app for research work.

The final direction is an app that can eventually:

```text
-> manage patients/samples
-> create measurement sessions
-> connect to a device
-> collect live data
-> process sensor values
-> run local ML inference
-> save structured data
-> export research files
```

## Part 1 - Foundation and architecture theory

**Lessons 1-22 explain the concepts before the clean implementation begins.**

This part starts with basic Kotlin, moves into Jetpack Compose, then introduces app state, persistence, background work, repository structure, Room, navigation, device abstraction, signal processing, ML inference, export, and the final architecture review.

The main purpose of Part 1 is to answer:

```text
What kind of app are we building?
What Kotlin/Android concepts are needed?
What architecture should the app follow?
Why should each responsibility live in a separate layer?
```

### Lesson 1 - Kotlin basics

Short summary:
The introduction explains that the goal is to learn only the Kotlin needed for Android research-app development, but there is not yet any code foundation. Lesson 1 starts that foundation with simple values, text, calculations, and functions.

Covered:

- `val` and `var`
- basic types: `Int`, `Double`, `Float`, `String`, `Boolean`
- strings and string interpolation with `$variable` and `${expression}`
- printing with `println`
- arithmetic and type conversion
- functions with `fun`
- return values
- short function syntax
- simple research examples such as calculating a mean value

The lesson focuses on writing simple Kotlin code comfortably.

### Lesson 2 - Control flow and collections

Short summary:
Lesson 1 covered basic values and functions, but it did not yet show how code makes decisions or handles repeated readings. Lesson 2 adds control flow and collections so research data can be filtered, counted, and processed.

Covered:

- `if`
- `when`
- logical operators: `&&`, `||`, `!`
- ranges: `1..5`, `until`, `step`, `downTo`
- `for` loops
- `while` loops
- `break` and `continue`
- `List`
- `MutableList`
- `filter`
- `map`
- `forEach`
- accessing list elements
- calculating mean/min/max/sum
- arrays such as `FloatArray`, `DoubleArray`, and `IntArray`

The research-app focus is processing sensor readings, rejecting invalid values, calculating statistics, and determining measurement status.

### Lesson 3 - Classes, data classes, null safety, and enums

Short summary:
Lesson 2 showed how to process simple lists of values, but the app still lacks a way to represent real research records as structured objects. Lesson 3 adds classes, data classes, null safety, and enums so measurements can carry meaningful information safely.

Covered:

- classes and objects
- constructors
- `val` and `var` inside classes
- functions inside classes
- `data class`
- named arguments
- `copy()`
- lists of objects
- nullable types with `?`
- safe call `?.`
- Elvis operator `?:`
- non-null assertion `!!`
- default parameter values
- `enum class`
- `when` with enums

This lesson introduces structured research data, for example `Measurement(sampleId, value, timestamp)`, and explains why null safety is important in Android because devices, readings, files, and database results may not exist yet.

### Lesson 4 - First Android app with Jetpack Compose

Short summary:
Lessons 1-3 built enough Kotlin grammar to understand basic code, but nothing has run inside Android yet. Lesson 4 moves into Android Studio and Jetpack Compose by building the first interactive research screen.

Covered:

- creating an Android Studio project
- understanding `MainActivity.kt`
- `onCreate()`
- `setContent { ... }`
- `@Composable`
- basic Compose UI elements: `Text`, `Button`, `TextField`, `Column`
- simple state with `remember` and `mutableStateOf`
- nullable measurement values in the UI
- button click logic
- generating a fake/random measurement value

This is where the tutorial moves from pure Kotlin console examples into an actual Android screen.

### Lesson 5 - Compose layout and UI components

Short summary:
Lesson 4 created a basic Compose screen, but the interface is still too plain for a practical tablet research app. Lesson 5 improves the layout with rows, spacing, input fields, status display, and clearer screen organization.

Covered:

- improving the plain app screen into a more usable tablet-style research interface
- `Column`
- `Row`
- `Spacer`
- `Modifier`
- padding
- alignment
- `OutlinedTextField`
- displaying status messages conditionally
- start/reset buttons
- showing connection status, sample ID, current value, and measurement count

The main goal is to make the app layout clearer and closer to a real research data-collection interface.

### Lesson 6 - Measurement history with data class, MutableList, and LazyColumn

Short summary:
Lesson 5 made the screen clearer, but the app still only focuses on the current value instead of a record of measurements. Lesson 6 adds measurement history, a list display, and simple statistics so the app behaves more like a data-collection tool.

Covered:

- creating a `Measurement` data class
- storing multiple measurements instead of only the latest one
- using `mutableStateListOf<Measurement>()`
- adding new measurements
- displaying a measurement history
- using `LazyColumn`
- calculating mean/min/max from recorded values
- showing a table-like measurement log
- separating parts of the UI into smaller composable functions

This lesson makes the app feel more like a real data-collection tool because it stores the full measurement history.

### Lesson 6 note - val, var, remember, and recomposition

Short summary:
This companion note explains why calculated values such as `latestMeasurement`, `meanText`, `minText`, and `maxText` use `val`, while editable Compose state such as `sampleId`, `isConnected`, `measurementValue`, and `measurementCount` use `var`.

Covered:

- why changing over time does not automatically mean `var`
- why local calculated values can be `val` during one recomposition
- why Compose reruns composable functions when state changes
- how `remember` prevents state from resetting during recomposition
- why `var by remember { mutableStateOf(...) }` is used for reassigned UI state
- why `val measurementList = remember { mutableStateListOf<Measurement>() }` can still allow list contents to change
- when to use plain `val`, `val` with `remember`, and `var` with `remember`

### Lesson 7 - Saving and exporting measurements as CSV

Short summary:
Lesson 6 keeps a measurement history inside the running app, but that data is not yet useful outside Android. Lesson 7 adds CSV export so measurements can be saved for Excel, Python, backup, and later research analysis.

Covered:

- converting measurement objects into CSV text
- creating a CSV string with headers and rows
- using Android's document picker
- `ActivityResultContracts.CreateDocument("text/csv")`
- writing CSV data to a user-selected file
- adding an Export CSV button
- creating meaningful file names such as `D1-ETO-W0-U1-S1_measurements.csv`

This lesson adds one of the most important research-app functions: exporting data for Excel, Python, backup, or later analysis.

### Lesson 8 - App architecture and ViewModel

Short summary:
Lesson 7 adds useful export behavior, but too much state and logic still lives directly inside the screen. Lesson 8 introduces ViewModel architecture so the UI can focus on display while app logic and state move into a cleaner layer.

Covered:

- why putting everything inside one composable becomes messy
- separating UI code from app logic/state
- introducing `ViewModel`
- creating a `ResearchUiState` data class
- moving app state from `ResearchScreen()` into `ResearchViewModel`
- exposing state to the UI
- passing functions such as `viewModel::addMeasurement`
- keeping the UI mostly responsible for display, not business logic

This is the first architecture lesson. It introduces the idea that the screen should display state, while the ViewModel manages the state and logic.

### Lesson 9 - Simple data persistence: auto-save and reload previous session

Short summary:
Lesson 8 separates UI from state, but the app still loses in-memory measurements if it closes before export. Lesson 9 adds internal autosave and reload so data can survive app restarts, while CSV export remains the user-facing analysis file.

Covered:

- the limitation of `ViewModel`: it is not permanent storage
- saving measurements automatically
- reloading previous measurements when the app starts
- using app-specific internal storage
- difference between internal auto-save and CSV export
- adding load/save functions to the ViewModel
- using `LaunchedEffect`
- warning that file writing on the UI thread is acceptable only for a small learning prototype
- explaining that heavier saving should later move to coroutines/background work
- mentioning Room as something useful later, but not necessary yet

This lesson makes the app more robust because data is not lost if the user forgets to export manually.

### Lesson 10 - Coroutines and background work

Short summary:
Lesson 9 adds persistence, but its simple saving approach can block the UI if work becomes heavier. Lesson 10 introduces coroutines and background work so saving, loading, acquisition, device communication, processing, and inference can run without freezing the screen.

Covered:

- why long-running work should not block the UI thread
- the main/UI thread idea
- basic coroutine concept
- `suspend`
- `viewModelScope.launch`
- `Dispatchers.IO`
- `withContext(Dispatchers.IO)`
- background file saving
- background loading of previous data
- updating UI state before and after background work
- error handling with user messages

This lesson prepares the app for file saving, loading previous sessions, device communication, signal processing, ML inference, and long-running acquisition without freezing the interface.

### Lesson 11 - Simulated live data stream

Short summary:
Lesson 10 prepares the app for background tasks, but measurements are still not arriving as a live stream. Lesson 11 adds simulated repeated acquisition so the app can practice start/stop recording before real hardware is connected.

Covered:

- moving from one manual measurement to repeated acquisition
- fake sensor/live data stream
- start acquisition
- stop acquisition
- coroutine loop
- `delay()` between samples
- adding measurements over time
- updating the latest value while recording
- stopping acquisition safely
- keeping the UI responsive during repeated measurement

This lesson builds the acquisition pattern with fake data first, so the app logic can be tested before real hardware is connected.

### Lesson 12 - Device connection state and acquisition flow

Short summary:
Lesson 11 creates a simulated live stream, but the app still lacks a clear model of connection and recording states. Lesson 12 adds device and acquisition states so buttons, messages, and actions follow a realistic measurement flow.

Covered:

- device connection state
- acquisition state
- states such as disconnected, connecting, connected, idle, recording, stopped, and error
- enabling and disabling buttons based on state
- preventing invalid actions
- showing connection and acquisition status messages
- separating UI state from acquisition logic
- creating a more realistic measurement workflow

This lesson makes the app behave more like a real measurement tool by only allowing actions that make sense in the current state.

### Lesson 13 - Repository layer

Short summary:
Lesson 12 improves the acquisition flow, but the ViewModel is still at risk of becoming responsible for too much data work. Lesson 13 introduces a repository layer to keep measurement creation, loading, saving, and future data-source logic out of the UI state layer.

Covered:

- why the ViewModel should not do all data work
- introducing `MeasurementRepository`
- moving measurement creation into the repository
- saving measurements through the repository
- loading measurements through the repository
- hiding whether data comes from fake generation, storage, or another source
- preparing for future device, database, export, and ML logic

This lesson introduces a cleaner architecture where the UI talks to the ViewModel, and the ViewModel asks the repository to handle data-related work.

### Lesson 14 - Room database introduction

Short summary:
Lesson 13 separates data logic into a repository, but the app still needs more reliable structured storage than simple files for serious research records. Lesson 14 introduces Room with entities, DAOs, and a database.

Covered:

- why Room is useful for structured local storage
- entity
- DAO
- database class
- inserting measurements
- reading previous measurements
- comparing Room with simple internal file storage
- comparing Room storage with CSV export
- preparing for larger research datasets

This lesson explains why a serious research app eventually needs structured database storage instead of only saving one CSV-style file.

### Lesson 15 - Patient and session data model

Short summary:
Lesson 14 introduces Room storage, but storing only measurements is not enough research context. Lesson 15 expands the model to patients, sessions, measurements, and results so every value belongs to the correct research record.

Covered:

- moving beyond one sample ID
- `Patient`
- `Session`
- `Measurement`
- `Result`
- `PatientEntity`
- `SessionEntity`
- `MeasurementEntity`
- `ResultEntity`
- relationships between patients, sessions, measurements, and results
- DAOs for the main research objects
- the idea that measurements should belong to sessions
- the idea that sessions should belong to patients or samples

This lesson turns the app from a simple measurement logger into a more realistic research data system.

### Lesson 16 - Multi-screen app navigation

Short summary:
Lesson 15 creates a richer patient/session data model, but one large screen no longer fits the workflow. Lesson 16 splits the app into multiple screens and passes IDs between them so the UI matches the research structure.

Covered:

- moving from one large `ResearchScreen` to multiple screens
- `PatientListScreen`
- `PatientDetailScreen`
- `MeasurementScreen`
- `ResultScreen`
- Navigation Compose
- `NavController`
- `NavHost`
- routes
- route arguments
- callbacks from screens
- back navigation
- passing IDs between screens

This lesson makes the app structure match the research workflow: choose a patient, open a session, collect measurements, and view results.

### Lesson 17 - Permissions and real device communication overview

Short summary:
Lesson 16 gives the app a multi-screen flow, but the measurement screen still is not ready for real device access. Lesson 17 adds the permission and connection overview needed before Bluetooth, Wi-Fi, USB, or sensor communication can be added.

Covered:

- why Android permissions matter
- manifest permissions
- runtime permissions
- Bluetooth permission concepts
- Wi-Fi/network permission concepts
- permission state in UI state
- permission request UI
- enabling connection only after permission is granted
- connection state before real hardware is added
- where real device code should belong in the architecture

This lesson prepares the app for real Bluetooth, Wi-Fi, USB, or sensor communication while keeping the first version beginner-friendly.

### Lesson 18 - Connecting a real data source

Short summary:
Lesson 17 explains permissions and real-device preparation, but the app still needs a clean way to swap fake data for real input. Lesson 18 introduces a data-source boundary so Bluetooth, Wi-Fi, USB, sensors, or replay data can replace random values later.

Covered:

- replacing direct random-value generation with a data-source boundary
- `DeviceDataSource` interface
- `connect()`
- `disconnect()`
- `readValue()`
- `FakeDeviceDataSource`
- preparing for `BluetoothDeviceDataSource`
- preparing for `WifiDeviceDataSource`
- handling connected/disconnected state
- keeping hardware communication out of the UI
- letting the repository ask the data source for measurements

This lesson creates the structure that lets fake data be replaced by real device input without rewriting the whole app.

### Lesson 19 - Signal processing pipeline

Short summary:
Lesson 18 creates a device-data boundary, but raw incoming values are still not being cleaned or transformed before use. Lesson 19 adds a signal-processing layer so raw values, processed values, validity checks, and features are handled separately from the UI.

Covered:

- raw data vs processed data
- why real sensor values may need processing
- keeping signal processing out of the UI
- introducing `SignalProcessor`
- baseline correction
- value validation
- status values such as `OK` and `INVALID`
- processed values
- simple feature extraction
- saving raw values separately from processed values when possible

This lesson adds the research-data habit of processing values in a separate layer before saving, exporting, or classifying them.

### Lesson 20 - On-device ML inference

Short summary:
Lesson 19 produces processed values and features, but the app still does not turn them into a prediction or classification result. Lesson 20 introduces the model-runner structure and fake inference so the app can test the full prediction workflow before using a real LiteRT/TFLite model.

Covered:

- what on-device inference means
- model input
- model output
- `ModelRunner` interface
- `PredictionResult`
- `FakeModelRunner`
- using recent measurements as model input
- using extracted features as model input
- running inference through the repository
- saving prediction results
- showing prediction and confidence in the UI
- preparing for a real LiteRT/TFLite model later

This lesson turns the app toward an edge-AI research workflow while still using fake inference first.

### Lesson 21 - Exporting complete research data

Short summary:
Lesson 20 adds prediction results, but the original CSV export idea is now too simple for the full research data model. Lesson 21 expands export to include patient, session, raw measurement, processed value, status, and prediction information.

Covered:

- why export still matters when Room exists
- exporting patient metadata
- exporting session metadata
- exporting measurement rows
- exporting raw values
- exporting processed values
- exporting status values
- exporting prediction label and confidence
- comparing CSV and JSON
- creating a session-level export
- making exported data useful for Python, Excel, backup, model validation, and collaboration

This lesson expands export from a simple measurement list into a complete research-data export.

### Lesson 22 - Final research app architecture review

Short summary:
Lesson 21 completes the main data path through export, but the tutorial still needs one clear map of how all parts fit together. Lesson 22 reviews the full architecture across UI, ViewModel, repository, data model, device source, processing, inference, storage, and export.

Covered:

- full app purpose
- presentation layer
- application/ViewModel layer
- domain/data model layer
- repository layer
- device/data-source layer
- signal processing layer
- ML inference layer
- Room database storage
- CSV/JSON export
- multi-screen app flow
- patient/session/measurement/result relationships
- how all previous lessons fit together

This lesson connects the whole tutorial into one maintainable research-app architecture.
## Part 2 - Implementation of the app skeleton

**Lessons 23-33 implement Direction A.**

This part turns the architecture from Lessons 1-22 into actual Android project files and a complete fake-data research workflow.

The main purpose of Part 2 is to answer:

```text
Which files should exist?
Which package should each file belong to?
How do Room, repository, fake device, processing, fake ML, screens, navigation, and export connect?
Can the whole fake workflow run from start to finish?
```

### Lesson 23 - Clean Android project structure

Short summary:
Lesson 22 reviews the full architecture, but the app still needs a real Android Studio file structure. Lesson 23 turns the architecture into packages and files so the project can be built cleanly.

Covered:

- project root structure
- package structure under `com.example.researchapp`
- `ui`
- `viewmodel`
- `data`
- `data/entity`
- `data/dao`
- `device`
- `processing`
- `ml`
- `export`
- keeping `MainActivity.kt` small
- separating screens into individual files
- starting with placeholders before real Bluetooth or real ML
- avoiding too much code in the UI

This lesson starts Direction A by turning the architecture into a concrete Android project layout.

### Lesson 24 - Creating the core data model files

Short summary:
Lesson 23 creates the folder structure, but the app still needs the actual data objects that Room will store. Lesson 24 creates the core entity files for patients, sessions, measurements, and results.

Covered:

- `PatientEntity.kt`
- `SessionEntity.kt`
- `MeasurementEntity.kt`
- `ResultEntity.kt`
- `@Entity`
- `@PrimaryKey(autoGenerate = true)`
- patient/sample codes
- session names
- nullable `endedAt`
- raw values and processed values
- measurement status
- result label and confidence
- database IDs versus user-facing codes
- why research data needs context

This lesson makes the research data model real Kotlin code.

### Lesson 25 - Building the Room database layer

Short summary:
Lesson 24 creates the entities, but the app still needs database access functions. Lesson 25 builds the DAO files and the `ResearchDatabase` class so the app can insert, read, update, and query research data.

Covered:

- `PatientDao.kt`
- `SessionDao.kt`
- `MeasurementDao.kt`
- `ResultDao.kt`
- `ResearchDatabase.kt`
- DAO meaning: Data Access Object
- `@Dao`
- `@Insert`
- `@Query`
- inserting patients, sessions, measurements, and results
- reading patients and sessions
- reading measurements for a session
- reading latest result for a session
- ending a session by updating `endedAt`
- why DAO functions are `suspend`
- Room 3 import/package note
- basic Gradle dependency idea

This lesson defines the structured local database layer for the app.

### Lesson 26 - Building the repository layer

Short summary:
Lesson 25 creates the database layer, but the ViewModel should not call every DAO directly. Lesson 26 builds the repository layer that coordinates database operations and prepares the app for device, processing, ML, and export work.

Covered:

- `MeasurementRepository.kt`
- where the repository fits
- why the app needs a repository
- creating and using the Room database
- getting DAO objects from the database
- patient functions
- session functions
- measurement functions
- result functions
- helper function for creating a patient and session together
- deciding whether one repository is enough
- avoiding UI code inside the repository
- keeping navigation decisions out of the repository
- basic ViewModel error-handling idea
- preparing the repository for fake device, processing, ML, and export

This lesson connects the database layer to the rest of the app through a cleaner data-coordination layer.

### Lesson 27 - Adding the fake device source

Short summary:
Lesson 26 builds the repository, but the app still needs a source of measurement values. Lesson 27 adds a fake device source so acquisition can be tested before real Bluetooth, Wi-Fi, USB, or sensors are added.

Covered:

- why fake device data comes first
- `DeviceDataSource.kt`
- `FakeDeviceDataSource.kt`
- `connect()`
- `disconnect()`
- `readValue()`
- using an interface for replaceable data sources
- fake connection state
- using `delay()` to simulate real connection/read time
- checking connection before reading values
- adding device functions to the repository
- creating measurements from fake device values
- saving fake measurements to Room
- keeping the fake device independent from sessions and storage
- error behavior when reading while disconnected

This lesson lets the app test the acquisition path without real hardware.

### Lesson 28 - Adding signal processing

Short summary:
Lesson 27 saves fake raw values, but research data usually needs cleaning or transformation before inference or export. Lesson 28 adds a signal-processing layer for baseline correction, validity checks, moving average, and feature extraction.

Covered:

- `processing` package
- `SignalFeatures.kt`
- `SignalProcessor.kt`
- raw value versus processed value
- `baselineCorrect()`
- `isValidValue()`
- `movingAverage()`
- `extractFeatures()`
- updating the repository constructor
- adding a baseline value
- saving raw and processed values separately
- marking invalid measurements
- deciding whether to save invalid values
- recent processed values helper
- feature extraction helper
- keeping processing out of UI code

This lesson adds the processing stage between device data and saved/inferred research data.

### Lesson 29 - Adding fake ML inference

Short summary:
Lesson 28 extracts processed values and features, but the app still needs a prediction path. Lesson 29 adds fake ML inference so the app can test prediction, confidence, and result saving before a real LiteRT/TFLite model is integrated.

Covered:

- why fake ML comes first
- where ML code belongs
- `PredictionResult.kt`
- `ModelRunner.kt`
- `FakeModelRunner.kt`
- using a `ModelRunner` interface
- fake inference delay
- using `SignalFeatures` as model input
- mean-value threshold logic
- repository inference function
- nullable `PredictionResult?` when there is not enough data
- saving `ResultEntity` to Room
- returning UI-friendly prediction data
- using processed values rather than raw values
- deciding whether invalid measurements should be used
- preparing UI state for the current prediction

This lesson completes the first fake inference path from saved measurements to stored result.

### Lesson 30 - Building the Compose screens

Short summary:
Lesson 29 completes fake ML logic, but the app still needs separate screens that match the research workflow. Lesson 30 builds the Compose screen files for patient list, patient detail, measurement, and result display.

Covered:

- `PatientListScreen.kt`
- `PatientDetailScreen.kt`
- `MeasurementScreen.kt`
- `ResultScreen.kt`
- screen responsibilities
- passing data into screens
- passing callbacks out of screens
- why callbacks keep screens reusable
- patient list items and rows
- session list items and rows
- measurement status display
- connect/start/stop/inference buttons
- result display
- `LazyColumn` for lists
- avoiding raw timestamp display in the beginner UI

This lesson turns the app structure into visible Compose screens.

### Lesson 31 - Connecting navigation and ViewModel

Short summary:
Lesson 30 creates the screen files, but the screens still need to be connected to real app state and navigation. Lesson 31 connects Navigation Compose, `ResearchViewModel`, repository functions, device state, acquisition, and fake inference.

Covered:

- `ResearchApp.kt`
- navigation routes
- `NavHost`
- `NavController`
- state enums
- `ResearchUiState`
- UI item classes
- `ResearchViewModel`
- patient input functions
- loading patients from Room
- creating patients
- loading patient detail and sessions
- creating sessions
- loading session measurements
- display helper functions
- button-state helper functions
- connecting and disconnecting fake device
- starting acquisition
- stopping acquisition
- running fake inference

This lesson connects the screens to the actual fake-data workflow.

### Lesson 32 - Adding session CSV export

Short summary:
Lesson 31 connects the app workflow, but the completed session data still needs to leave the app in a researcher-friendly format. Lesson 32 adds session-level CSV export with patient metadata, session metadata, latest result, and measurement rows.

Covered:

- why export still matters
- exporting one complete session
- `export` package
- `ExportFormatter.kt`
- CSV escaping
- repository export function
- export filename helper
- pending export text in UI state
- ViewModel export preparation
- export-complete state update
- Android file picker / CreateDocument flow
- connecting the Result screen export button
- full export flow
- why the repository builds text but does not write the file directly
- including invalid measurements
- including raw and processed values
- model-version idea for future traceability

This lesson restores export as part of the full Room-backed research workflow.

### Lesson 33 - Testing the whole fake-data workflow

Short summary:
Lesson 32 adds session CSV export, so the app now has all the main fake workflow parts. Lesson 33 tests the whole connected system from app launch through patient/session creation, fake acquisition, inference, result display, and CSV export.

Covered:

- testing the complete fake-data workflow
- checking the expected project structure
- app opens to Patient List
- create patient/sample
- open patient detail
- create session
- open measurement screen
- connect fake device
- start acquisition
- stop acquisition
- run fake inference
- view result
- export CSV
- common problems and debugging checks
- disabled button checks
- route argument checks
- selected patient/session checks
- not enough data for inference
- layer-by-layer testing strategy
- debugging with messages and logs

This lesson completes Direction A by verifying the first working fake-data research app skeleton.
## How Part 1 connects to Part 2

The relationship is:

| Part 1 concept | Part 2 implementation |
|---|---|
| App architecture | Clean project packages |
| Research data model | `PatientEntity`, `SessionEntity`, `MeasurementEntity`, `ResultEntity` |
| Room concept | DAO files and `ResearchDatabase` |
| Repository idea | `MeasurementRepository` |
| Fake data strategy | `FakeDeviceDataSource` |
| Signal processing theory | `SignalProcessor` and `SignalFeatures` |
| ML inference concept | `ModelRunner`, `FakeModelRunner`, `PredictionResult` |
| Multi-screen flow | `PatientListScreen`, `PatientDetailScreen`, `MeasurementScreen`, `ResultScreen` |
| ViewModel state | `ResearchViewModel` and `ResearchUiState` |
| Export concept | `ExportFormatter` and session CSV export |
| Architecture review | Complete fake workflow test |

In simple terms:

```text
Part 1 teaches the ideas.
Part 2 turns the ideas into files.
```

## Architecture after Lesson 33

The app skeleton after Lesson 33 looks like this:

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

The runtime flow is:

```text
Screen
-> ResearchViewModel
-> MeasurementRepository
-> Room / FakeDeviceDataSource / SignalProcessor / FakeModelRunner / ExportFormatter
```

The research data flow is:

```text
Patient
-> Session
-> Measurements
-> Processed values
-> Features
-> Prediction
-> Result
-> CSV export
```

## Current milestone

By Lesson 33, Direction A is essentially complete.

The tutorial has reached this milestone:

```text
working fake-data research app skeleton
```

That means the app can conceptually:

- create a patient/sample
- create a session
- connect to a fake device
- collect fake measurements
- process values
- save data in Room
- run fake ML inference
- save a result
- show the result
- export the session as CSV
- test the whole workflow manually

## What comes next

The next stage should replace fake or simplified pieces one at a time, without destroying the structure already built.

Recommended next direction:

```text
1. Compile and run the project in Android Studio.
2. Fix Gradle, Room, KSP, or import issues.
3. Test the complete fake workflow on emulator/tablet.
4. Replace FakeDeviceDataSource with real device input.
5. Improve SignalProcessor for the real research signal.
6. Replace FakeModelRunner with a real LiteRT/TFLite model runner.
7. Add model version and processing version tracking.
8. Improve CSV/JSON export.
9. Add automated tests.
10. Add dependency injection, migrations, logging, and stronger error handling.
```

The main principle stays the same:

```text
Keep UI simple.
Keep data contextual.
Keep hardware, processing, ML, storage, and export in separate layers.
```
