# Lesson 24 — Turning the Architecture into a Clean Android Project Structure

In Lessons 1-23, we built the **conceptual foundation** of the Android research app.

The tutorial started with the idea that we should not learn all Kotlin/Android theory first, but should learn the subset needed to build a practical research app. fileciteturn0file0L6-L8

Now we start **Direction A**:

```text
Build a clean Android project from this architecture.
```

So Lesson 24 is not about adding a new feature yet.

It is about turning the architecture into a real project structure.

---

## 1. Where we are now

After Lesson 23, the full app architecture looked like this:

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

Device/data-source layer
 ├── DeviceDataSource
 └── FakeDeviceDataSource

Processing layer
 └── SignalProcessor

ML layer
 ├── ModelRunner
 └── FakeModelRunner

Storage layer
 ├── Room database
 ├── DAOs
 └── Entities

Export layer
 └── CSV / JSON export
```

Now we need to decide:

```text
Where should these files live in the Android Studio project?
```

That is the purpose of Lesson 24.

---

## 2. Why project structure matters

At the beginning, it is tempting to put everything into:

```text
MainActivity.kt
```

or maybe:

```text
ResearchScreen.kt
```

This is okay for the first few lessons.

But now the app has many responsibilities:

```text
UI screens
navigation
ViewModel state
Room database
repository
device communication
signal processing
ML inference
export
```

If everything is in one or two files, the project becomes difficult to maintain.

A clean project structure helps you answer:

```text
Where should I put this code?
What depends on what?
Which file should I edit?
Which parts are UI?
Which parts are data?
Which parts are device-specific?
```

This is especially important for your app because it is a research app, not a small demo.

---

## 3. The Android project root

In Android Studio, your project usually looks like this:

```text
ResearchApp
 ├── app
 │    ├── build.gradle.kts
 │    └── src
 │         └── main
 │              ├── AndroidManifest.xml
 │              ├── java
 │              │    └── com
 │              │         └── example
 │              │              └── researchapp
 │              └── res
 │
 ├── build.gradle.kts
 └── settings.gradle.kts
```

Even though we are writing Kotlin, the source folder may still be called:

```text
java
```

That is normal.

Your Kotlin files can still live inside:

```text
app/src/main/java/com/example/researchapp
```

So our main working folder is:

```text
app/src/main/java/com/example/researchapp
```

---

## 4. Suggested package structure

For this tutorial, use this structure:

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
 │    └── ResearchUiState.kt
 │
 ├── data
 │    ├── ResearchDatabase.kt
 │    ├── MeasurementRepository.kt
 │    │
 │    ├── entity
 │    │    ├── PatientEntity.kt
 │    │    ├── SessionEntity.kt
 │    │    ├── MeasurementEntity.kt
 │    │    └── ResultEntity.kt
 │    │
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
 │    ├── ModelRunner.kt
 │    ├── FakeModelRunner.kt
 │    └── PredictionResult.kt
 │
 └── export
      └── ExportFormatter.kt
```

This is not the only possible structure.

Important addition:

```text
runtime
|-- ResearchRuntimeState.kt
`-- ResearchRuntimeStateHolder.kt
```

Place `runtime` beside `viewmodel`, `data`, `ui`, `device`, `processing`, `ml`, and `export`.

The updated top-level package list is:

```text
com.example.researchapp
|-- ui
|-- viewmodel
|-- runtime
|-- data
|-- device
|-- processing
|-- ml
`-- export
```

But it is a good beginner-friendly structure for our current app.

---

## 5. Why these folders?

Each folder has a clear meaning.

| Folder | Purpose |
|---|---|
| `ui` | Compose screens and navigation |
| `viewmodel` | UI state and app flow |
| `runtime` | Shared temporary app/session state used across screens or ViewModels |
| `data` | Room database, DAOs, entities, repository |
| `device` | Fake or real device communication |
| `processing` | Signal processing and feature extraction |
| `ml` | ML model interface and fake/real model runner |
| `export` | CSV/JSON export formatting |

The goal is not to create many folders for no reason.

The goal is to keep different responsibilities separate.

---

## 6. `MainActivity.kt`

`MainActivity.kt` should stay small.

Its job is only to start the app UI.

Example:

```kotlin
package com.example.researchapp

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import com.example.researchapp.ui.ResearchApp

class MainActivity : ComponentActivity() {

    override fun onCreate(
        savedInstanceState: Bundle?
    ) {
        super.onCreate(savedInstanceState)

        setContent {
            ResearchApp()
        }
    }
}
```

This file should not contain:

```text
Room database code
Bluetooth code
signal processing code
ML inference code
CSV export code
```

It should only start the Compose app.

---

## 7. `ui` folder

The `ui` folder contains the visible screens.

```text
ui
 ├── ResearchApp.kt
 ├── PatientListScreen.kt
 ├── PatientDetailScreen.kt
 ├── MeasurementScreen.kt
 └── ResultScreen.kt
```

### `ResearchApp.kt`

This file owns the navigation graph.

It decides which screen is shown.

Example structure:

```kotlin
package com.example.researchapp.ui

import androidx.compose.runtime.Composable
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController

@Composable
fun ResearchApp() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = "patient_list"
    ) {
        composable("patient_list") {
            PatientListScreen()
        }

        composable("patient_detail/{patientId}") {
            PatientDetailScreen()
        }

        composable("measurement/{sessionId}") {
            MeasurementScreen()
        }

        composable("result/{sessionId}") {
            ResultScreen()
        }
    }
}
```

This is only a skeleton.

We will improve it later.

The important point is:

```text
ResearchApp.kt controls navigation.
Individual screens display UI.
```

---

## 8. Individual screen files

Each screen should have its own file.

For example:

```text
PatientListScreen.kt
```

contains:

```kotlin
package com.example.researchapp.ui

import androidx.compose.material3.Text
import androidx.compose.runtime.Composable

@Composable
fun PatientListScreen() {
    Text("Patient List Screen")
}
```

`PatientDetailScreen.kt`:

```kotlin
package com.example.researchapp.ui

import androidx.compose.material3.Text
import androidx.compose.runtime.Composable

@Composable
fun PatientDetailScreen() {
    Text("Patient Detail Screen")
}
```

`MeasurementScreen.kt`:

```kotlin
package com.example.researchapp.ui

import androidx.compose.material3.Text
import androidx.compose.runtime.Composable

@Composable
fun MeasurementScreen() {
    Text("Measurement Screen")
}
```

`ResultScreen.kt`:

```kotlin
package com.example.researchapp.ui

import androidx.compose.material3.Text
import androidx.compose.runtime.Composable

@Composable
fun ResultScreen() {
    Text("Result Screen")
}
```

At this stage, these are just placeholders.

That is okay.

In Direction A, we build the skeleton first.

Then we fill each part step by step.

---

## 9. `viewmodel` folder

The `viewmodel` folder contains:

```text
viewmodel
 ├── ResearchViewModel.kt
 └── ResearchUiState.kt
```

### `ResearchUiState.kt`

This file defines the screen state.

```kotlin
package com.example.researchapp.viewmodel

data class ResearchUiState(
    val patientCode: String = "",
    val sessionName: String = "",
    val message: String = ""
)
```

This is a simplified starting version.

Later, we will expand it with:

```text
currentPatientId
currentSessionId
deviceConnectionState
acquisitionState
measurements
latestRawValue
latestProcessedValue
latestPrediction
isLoading
```

But do not add everything at once.

Start simple.

---

### `ResearchViewModel.kt`

This file controls app state and user actions.

```kotlin
package com.example.researchapp.viewmodel

import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.setValue
import androidx.lifecycle.ViewModel

class ResearchViewModel : ViewModel() {

    var uiState by mutableStateOf(
        ResearchUiState()
    )
        private set

    fun updatePatientCode(
        newPatientCode: String
    ) {
        uiState = uiState.copy(
            patientCode = newPatientCode
        )
    }

    fun updateSessionName(
        newSessionName: String
    ) {
        uiState = uiState.copy(
            sessionName = newSessionName
        )
    }
}
```

The ViewModel should not directly contain UI layout code.

It should manage state and actions.

---

## 10. `runtime` folder

The `runtime` folder contains temporary app state that must be shared across more than one screen or more than one ViewModel.

This is different from `ResearchUiState`.

`ResearchUiState` is mostly for what one screen needs to display.

But some state belongs to the current running research workflow:

```text
selected patient ID
selected session ID
current device connection state
current acquisition state
latest live raw value
latest processed value
```

If only one screen needs a value, keep it in that screen's ViewModel.

If several screens or ViewModels need the same temporary value, put the real shared value in a runtime state holder.

Suggested files:

```text
runtime
|-- ResearchRuntimeState.kt
`-- ResearchRuntimeStateHolder.kt
```

### `ResearchRuntimeState.kt`

This file defines shared temporary app/session state.

```kotlin
package com.example.researchapp.runtime

data class ResearchRuntimeState(
    val currentPatientId: Long? = null,
    val currentSessionId: Long? = null,
    val isDeviceConnected: Boolean = false,
    val isAcquiring: Boolean = false,
    val latestRawValue: Double? = null,
    val latestProcessedValue: Double? = null
)
```

### `ResearchRuntimeStateHolder.kt`

This file owns and updates that shared runtime state.

```kotlin
package com.example.researchapp.runtime

import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update

class ResearchRuntimeStateHolder {

    private val _runtimeState =
        MutableStateFlow(ResearchRuntimeState())

    val runtimeState: StateFlow<ResearchRuntimeState> =
        _runtimeState.asStateFlow()

    fun selectPatient(patientId: Long) {
        _runtimeState.update { currentState ->
            currentState.copy(
                currentPatientId = patientId,
                currentSessionId = null
            )
        }
    }

    fun selectSession(sessionId: Long) {
        _runtimeState.update { currentState ->
            currentState.copy(
                currentSessionId = sessionId
            )
        }
    }

    fun setDeviceConnected(isConnected: Boolean) {
        _runtimeState.update { currentState ->
            currentState.copy(
                isDeviceConnected = isConnected
            )
        }
    }
}
```

### What functions belong in `ResearchRuntimeStateHolder`?

Now that we have seen the holder, we can ask a better design question:

```text
Which functions should live inside ResearchRuntimeStateHolder?
```

The criterion is not only:

```text
How many screens share this function?
```

The better criterion is:

```text
Is this function an operation on ResearchRuntimeState?
```

其实道理很简单，既然这个状态变量是私有唯一的，而你又需要去修改这个状态变量，那凡是涉及修改这个状态变量的操作函数自然都需要放到这个文件中去，不然怎么看到这个状态变量并进行修改呢。


If a function directly owns or protects changes to `ResearchRuntimeState`, it can belong in `ResearchRuntimeStateHolder`, even if currently only one ViewModel calls it.

For example:

```kotlin
fun selectPatient(patientId: Long) {
    _runtimeState.update { currentState ->
        currentState.copy(
            currentPatientId = patientId,
            currentSessionId = null
        )
    }
}
```

This belongs in `ResearchRuntimeStateHolder` because it modifies the shared _runtimeState variable safely.

But a screen workflow function should stay in the ViewModel:

```kotlin
fun onCreatePatientClick() {
    // validate text field
    // call repository
    // update screen message
    // ask runtime holder to select the patient if needed
}
```

That function is about a screen event, not just about changing `ResearchRuntimeState`.

So the split is:

```text
ResearchRuntimeStateHolder
    -> owns ResearchRuntimeState
    -> updates shared runtime values
    -> protects shared runtime rules

ViewModel
    -> handles screen events
    -> reads screen input
    -> calls repository
    -> calls runtime holder when shared runtime state must change
    -> prepares UI messages/state for that screen
```

A function can belong in `ResearchRuntimeStateHolder` even if only one ViewModel calls it today.

So the idea is clear:

```
Put functions in ResearchRuntimeStateHolder if they are operations of ResearchRuntimeState.
```

But add one nuance:

```
Do not put screen-specific workflow functions there unless they are truly modifying shared runtime-states.
```

The mental model is:

```text
ViewModel
    -> state and actions dedicately for one screen

runtime state holder
    -> temporary state shared across screens/ViewModels

Room/database
    -> permanent saved research data
```

For now, we only prepare this folder as an architectural home.

We are not adding dependency injection in this lesson.

---

## 11. `data` folder

The `data` folder contains the local database and repository.

```text
data
 ├── ResearchDatabase.kt
 ├── MeasurementRepository.kt
 ├── entity
 └── dao
```

This is where Room-related code will go.

For now, we only create the files.

We do not need to fully implement Room in Lesson 24.

That will be Lesson 26.

---

## 12. `data/entity` folder

This folder contains Room entities:

```text
entity
 ├── PatientEntity.kt
 ├── SessionEntity.kt
 ├── MeasurementEntity.kt
 └── ResultEntity.kt
```

These files represent database tables.

For example:

```text
PatientEntity
 ↓
patients table

SessionEntity
 ↓
sessions table

MeasurementEntity
 ↓
measurements table

ResultEntity
 ↓
results table
```

We will implement these properly in Lesson 25.

For now, the important idea is:

```text
Entity files describe what data we store.
```

---

## 13. `data/dao` folder

This folder contains DAO interfaces:

```text
dao
 ├── PatientDao.kt
 ├── SessionDao.kt
 ├── MeasurementDao.kt
 └── ResultDao.kt
```

DAO means:

```text
Data Access Object
```

These files describe how we read and write the database.

For example:

```text
PatientDao
 ↓
insert patient
get all patients

SessionDao
 ↓
insert session
get sessions for patient

MeasurementDao
 ↓
insert measurement
get measurements for session

ResultDao
 ↓
insert result
get results for session
```

We will implement these later.

---

## 14. `MeasurementRepository.kt`

The repository is the bridge between the ViewModel and the data/device/processing/ML layers.

Its final responsibility will be:

```text
create patient
create session
connect device
read measurement
process measurement
save measurement
run inference
save result
build export text
```

For now, create the file:

```kotlin
package com.example.researchapp.data

class MeasurementRepository {
}
```

This looks empty, but that is fine.

In Direction A, we are setting up the project step by step.

### A note about shared repository instances

Sometimes you may see repository code written like this:

```kotlin
class DeviceRepository {

    companion object {
        val instance: DeviceRepository by lazy {
            DeviceRepository()
        }
    }
}
```

This creates one shared `DeviceRepository` object that can be accessed through the class name:

```kotlin
val repository = DeviceRepository.instance
```

The pieces mean:

```text
companion object
    -> attach this value to the class itself
    -> call it with DeviceRepository.instance

val instance: DeviceRepository
    -> instance is a DeviceRepository value

by lazy { DeviceRepository() }
    -> do not create the object immediately
    -> create it the first time someone asks for it
    -> return the same object on later calls
```

So the flow is:

```text
App starts
    -> DeviceRepository.instance has not created anything yet

First call to DeviceRepository.instance
    -> lazy block runs
    -> DeviceRepository() creates the object
    -> that object is stored

Later calls to DeviceRepository.instance
    -> return the same stored object
```

This is a simple singleton-style pattern.

Singleton means:

```text
one shared object used from many places
```

This can be acceptable for a small fake repository.

But be careful with this pattern in Android if the repository needs:

```text
Context
Room database
device connection state
coroutines
lifecycle-aware behavior
```

For those cases, a ViewModel, application-level setup, or dependency injection is usually cleaner.

### What dependency injection means

Dependency injection sounds advanced, but the basic idea is simple.

A dependency is an object a class needs to do its job.

Injection means giving that object to the class from outside.

Without dependency injection, a repository creates its own dependencies:

```kotlin
class MeasurementRepository {

    private val deviceDataSource =
        FakeDeviceDataSource()

    private val signalProcessor =
        SignalProcessor()
}
```

With dependency injection, the repository receives them:

```kotlin
class MeasurementRepository(
    private val deviceDataSource: DeviceDataSource,
    private val signalProcessor: SignalProcessor
)
```

Then another part of the app decides what to provide:

```kotlin
val repository = MeasurementRepository(
    deviceDataSource = FakeDeviceDataSource(),
    signalProcessor = SignalProcessor()
)
```

The repository no longer says:

```text
I will build my own device source.
```

It says:

```text
I need a device source.
Please give me one.
```

This makes the code easier to test.

For example, the real app might use:

```kotlin
val repository = MeasurementRepository(
    deviceDataSource = RealBluetoothDeviceDataSource(),
    signalProcessor = SignalProcessor()
)
```

A test might use:

```kotlin
val repository = MeasurementRepository(
    deviceDataSource = FakeDeviceDataSource(),
    signalProcessor = SignalProcessor()
)
```

Same repository class.

Different objects passed in.

Short version:

```text
Dependency
    -> object this class needs

Injection
    -> passing that object in from outside
```

In larger Android apps, dependency injection is often handled with tools such as Hilt or Dagger.

For this tutorial, the important idea is the constructor pattern:

```kotlin
class MeasurementRepository(
    private val database: ResearchDatabase,
    private val deviceDataSource: DeviceDataSource,
    private val signalProcessor: SignalProcessor,
    private val modelRunner: ModelRunner
)
```

We are not fully adding dependency injection in Lesson 24.

But this is the architecture idea behind it:

```text
Do not make every class build all of its own tools.
Give classes the tools they need from the outside.
```

---

## 15. `device` folder

The `device` folder contains:

```text
device
 ├── DeviceDataSource.kt
 └── FakeDeviceDataSource.kt
```

### `DeviceDataSource.kt`

This is the interface from Lesson 19:

```kotlin
package com.example.researchapp.device

interface DeviceDataSource {
    suspend fun connect()
    suspend fun disconnect()
    suspend fun readValue(): Double
}
```

This is the doorway for fake or real device data.

---

### `FakeDeviceDataSource.kt`

```kotlin
package com.example.researchapp.device

import kotlinx.coroutines.delay
import kotlin.random.Random

class FakeDeviceDataSource : DeviceDataSource {

    private var connected: Boolean = false

    override suspend fun connect() {
        delay(1000)
        connected = true
    }

    override suspend fun disconnect() {
        connected = false
    }

    override suspend fun readValue(): Double {
        if (!connected) {
            throw IllegalStateException(
                "Device is not connected"
            )
        }

        delay(1000)

        return Random.nextDouble(
            0.0,
            5.0
        )
    }
}
```

This fake source lets us test the app without real hardware.

Later, we can add:

```text
BluetoothDeviceDataSource.kt
```

or:

```text
WifiDeviceDataSource.kt
```

But not yet.

---

## 16. `processing` folder

The `processing` folder contains signal-processing logic.

```text
processing
 ├── SignalProcessor.kt
 └── SignalFeatures.kt
```

### `SignalFeatures.kt`

```kotlin
package com.example.researchapp.processing

data class SignalFeatures(
    val mean: Double,
    val minimum: Double,
    val maximum: Double,
    val range: Double
)
```

### `SignalProcessor.kt`

```kotlin
package com.example.researchapp.processing

class SignalProcessor {

    fun baselineCorrect(
        rawValue: Double,
        baseline: Double
    ): Double {
        return rawValue - baseline
    }

    fun isValidValue(
        value: Double
    ): Boolean {
        return value in 0.0..10.0
    }

    fun extractFeatures(
        values: List<Double>
    ): SignalFeatures? {
        if (values.isEmpty()) {
            return null
        }

        val minimum = values.min()
        val maximum = values.max()

        return SignalFeatures(
            mean = values.average(),
            minimum = minimum,
            maximum = maximum,
            range = maximum - minimum
        )
    }
}
```

This keeps processing separate from the UI.

That is very important.

---

## 17. `ml` folder

The `ml` folder contains model-related code.

```text
ml
 ├── ModelRunner.kt
 ├── FakeModelRunner.kt
 └── PredictionResult.kt
```

### `PredictionResult.kt`

```kotlin
package com.example.researchapp.ml

data class PredictionResult(
    val label: String,
    val confidence: Double
)
```

### `ModelRunner.kt`

```kotlin
package com.example.researchapp.ml

import com.example.researchapp.processing.SignalFeatures

interface ModelRunner {
    suspend fun runInference(
        features: SignalFeatures
    ): PredictionResult
}
```

### `FakeModelRunner.kt`

```kotlin
package com.example.researchapp.ml

import com.example.researchapp.processing.SignalFeatures

class FakeModelRunner : ModelRunner {

    override suspend fun runInference(
        features: SignalFeatures
    ): PredictionResult {
        val label = if (features.mean > 2.5) {
            "Positive"
        } else {
            "Negative"
        }

        val confidence = if (features.mean > 2.5) {
            0.90
        } else {
            0.85
        }

        return PredictionResult(
            label = label,
            confidence = confidence
        )
    }
}
```

This gives us a fake ML result before using a real LiteRT/TFLite model.

---

## 18. `export` folder

The `export` folder contains export formatting code.

```text
export
 └── ExportFormatter.kt
```

For now:

```kotlin
package com.example.researchapp.export

class ExportFormatter {
}
```

Later, this class can build:

```text
session CSV
patient CSV
JSON export
```

The export logic should not live inside the UI.

The UI should only let the user choose where to save the file.

---

## 19. What should we implement first?

Do not implement everything at once.

A good Direction A order is:

```text
1. Create project folders
2. Create placeholder screen files
3. Create UiState and ViewModel
4. Create entity files
5. Create DAO files
6. Create Room database
7. Create repository
8. Connect fake device source
9. Add processing
10. Add fake ML
11. Add export
12. Test the complete fake workflow
```

Lesson 24 focuses mostly on step 1 and the file structure.

---

## 20. Why we start with placeholders

You may wonder:

```text
Why create empty or simple files first?
```

Because this helps you see the whole project shape.

For example, before implementing Room, you already know where Room files will live.

Before implementing ML, you already know where ML files will live.

Before implementing Bluetooth, you already know where the device source will live.

This prevents the common beginner problem:

```text
I wrote working code,
but now I do not know where anything belongs.
```

A clean skeleton gives you a map.

---

## 21. Common mistake: too much code in UI

A common beginner structure is:

```text
MeasurementScreen.kt
 ├── UI buttons
 ├── Bluetooth code
 ├── Room insert code
 ├── signal processing
 ├── model inference
 └── CSV export
```

This is bad because the screen becomes too powerful.

A better structure is:

```text
MeasurementScreen
 ↓
calls ViewModel

ViewModel
 ↓
calls Repository

Repository
 ↓
uses DeviceDataSource
uses SignalProcessor
uses ModelRunner
uses Room DAOs
uses ExportFormatter
```

That is the architecture we are building.

---

## 22. Common mistake: starting with real Bluetooth too early

Another common mistake is trying to implement real Bluetooth before the app skeleton exists.

That can become frustrating because Bluetooth involves:

```text
permissions
scanning
connection
streams
parsing
errors
Android version differences
```

Instead, our path is:

```text
FakeDeviceDataSource first
 ↓
complete app workflow
 ↓
replace fake device with real Bluetooth/Wi-Fi later
```

This is a safer learning path.

---

## 23. Common mistake: starting with real ML too early

Similarly, do not start with the real model immediately.

A real model requires:

```text
.tflite model file
input shape
feature order
normalisation
output interpretation
label mapping
model version
```

Instead, our path is:

```text
FakeModelRunner first
 ↓
complete inference workflow
 ↓
replace fake model with real LiteRT model later
```

That means the UI, ViewModel, Repository, and Result screen can be tested before the real model is ready.

---

## 24. Current skeleton after Lesson 24

After Lesson 24, your project should have this shape:

```text
com.example.researchapp
 ├── MainActivity.kt
 ├── ui
 ├── viewmodel
 ├── data
 ├── device
 ├── processing
 ├── ml
 └── export
```

The app may not do much yet.

The skeleton should also include the shared runtime folder:

```text
runtime
|-- ResearchRuntimeState.kt
`-- ResearchRuntimeStateHolder.kt
```

It may only show placeholder screens.

That is okay.

The goal of Lesson 24 is not to finish the app.

The goal is to create a clean foundation.

---

## 25. What you learned in Lesson 24

You learned how to map the architecture into real Android project folders:

```text
UI screens
 ↓
ui

state and flow
 ↓
viewmodel

shared temporary runtime state
 -> runtime

Room and repository
 ↓
data

hardware communication
 ↓
device

signal processing
 ↓
processing

ML inference
 ↓
ml

CSV/JSON export
 ↓
export
```

You also learned two setup ideas that affect how these folders connect:

```text
companion object + by lazy
    -> create one shared instance only when it is first used

dependency injection
    -> pass needed objects in from outside
```

The most important mental model is:

```text
Each responsibility should have a home.
```

When you add new code, ask:

```text
Is this UI code?
Is this state-management code?
Is this shared runtime state used across screens/ViewModels?
Is this database code?
Is this device code?
Is this processing code?
Is this ML code?
Is this export code?
Is this a shared instance that should be created once?
Is this a dependency that should be passed in from outside?
```

Then place it in the correct folder.

This is how we prevent the app from becoming messy.

For shared temporary state, use this extra rule:

```text
If only one screen needs it:
    keep it in that screen's ViewModel.

If several screens or ViewModels need it:
    put the real shared value and shared operations in runtime.

If it must survive app restart:
    save it through Room, DataStore, or files.
```

---

# Lesson 25 preview

In Lesson 25, we will start implementing the real project files.

The next step is:

```text
Creating the Core Data Model Files
```

We will create:

```text
PatientEntity
SessionEntity
MeasurementEntity
ResultEntity
```

and explain exactly what each field means.

This will turn the research data model from Lesson 16 into real Kotlin files inside the new project structure.
