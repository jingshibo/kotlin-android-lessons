# Lesson 25 — Turning the Architecture into a Clean Android Project Structure

In Lessons 1-24, we built the **conceptual foundation** of the Android research app.

The tutorial started with the idea that we should not learn all Kotlin/Android theory first, but should learn the subset needed to build a practical research app. fileciteturn0file0L6-L8

Now we start **Direction A**:

```text
Build a clean Android project from this architecture.
```

So Lesson 25 is not about adding a new feature yet.

It is about turning the architecture into a real project structure.

---

## 1. Where we are now

After Lesson 24, the full app architecture looked like this:

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

That is the purpose of Lesson 25.

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
 |-- MainActivity.kt
 |
 |-- ui
 |   |-- ResearchApp.kt
 |   |-- PatientListScreen.kt
 |   |-- PatientDetailScreen.kt
 |   |-- MeasurementScreen.kt
 |   |-- ResultScreen.kt
 |   |
 |   `-- model
 |       |-- PatientListItem.kt
 |       `-- SessionListItem.kt
 |
 |-- viewmodel
 |   |-- ResearchViewModel.kt
 |   |-- ResearchUiState.kt
 |   |
 |   `-- mapper
 |       `-- EntityUiMappers.kt
 |
 |-- runtime
 |   |-- ResearchRuntimeState.kt
 |   `-- ResearchRuntimeStateManager.kt
 |
 |-- data
 |   |-- ResearchDatabase.kt
 |   |-- MeasurementRepository.kt
 |   |
 |   |-- entity
 |   |   |-- PatientEntity.kt
 |   |   |-- SessionEntity.kt
 |   |   |-- MeasurementEntity.kt
 |   |   `-- ResultEntity.kt
 |   |
 |   `-- dao
 |       |-- PatientDao.kt
 |       |-- SessionDao.kt
 |       |-- MeasurementDao.kt
 |       `-- ResultDao.kt
 |
 |-- device
 |   |-- DeviceDataSource.kt
 |   `-- FakeDeviceDataSource.kt
 |
 |-- processing
 |   |-- SignalProcessor.kt
 |   `-- SignalFeatures.kt
 |
 |-- ml
 |   |-- ModelRunner.kt
 |   |-- FakeModelRunner.kt
 |   `-- PredictionResult.kt
 |
 `-- export
     `-- ExportFormatter.kt
```

This is not the only possible structure.

But it is a good beginner-friendly structure for our current app.

The new package here is:

```text
runtime
```

This package is for temporary workflow state and the operations that safely change that state.

It is separate from `viewmodel` because runtime state is not just one screen's UI state.

It is separate from `data` because runtime state is not permanent database data.

A useful mental model is:

```text
viewmodel
    -> single screen states and events

runtime
    -> runtime states and operations across screens

data
    -> persistent research data and data operations
```

The UI-specific model files introduced later belong in:

```text
app/src/main/java/com/example/researchapp/ui/model/PatientListItem.kt
app/src/main/java/com/example/researchapp/ui/model/SessionListItem.kt
```

These classes describe the values needed by particular screens. They are not Room entities and do not belong in `data/entity`.

The functions that convert database entities into these UI models can go in:

```text
app/src/main/java/com/example/researchapp/viewmodel/mapper/EntityUiMappers.kt
```

For example, that file can later contain:

```kotlin
package com.example.researchapp.viewmodel.mapper

import com.example.researchapp.data.entity.PatientEntity
import com.example.researchapp.ui.model.PatientListItem

fun PatientEntity.toPatientListItem(): PatientListItem {
    return PatientListItem(
        id = id,
        patientCode = patientCode,
        createdAt = createdAt
    )
}
```

This mapper sits near the ViewModel because the ViewModel uses it to build UI state. Do not place this entity-to-UI mapper in the `data` package: doing so would make the data layer depend on a UI model.

We are not adding a `domain` folder yet. A domain layer is useful when the app develops reusable business rules or needs storage-independent models across several features. It is not required merely to convert a Room entity into one screen's UI model.

Section 4 shows the target structure so that each future file has a clear destination. You do not need to create every file before the lesson that introduces it.

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

`ResearchUiState` is mostly for those states used by only one screen.

But some states belongs to a workflow across multiple screens/viewmodels:

```text
selected patient ID
selected session ID
current device connection state
current acquisition state
latest live raw value
latest processed value
```

If only one screen needs a value, keep it in that screen's ViewModel.

If several screens or ViewModels need the same temporary value, put the real shared value in a runtime state manager.

Suggested files:

```text
runtime
|-- ResearchRuntimeState.kt
`-- ResearchRuntimeStateManager.kt
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

### `ResearchRuntimeStateManager.kt`

This file owns and updates that shared runtime state.

```kotlin
package com.example.researchapp.runtime

import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update

class ResearchRuntimeStateManager {

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

`ResearchRuntimeStateManager` is not just a place to store variables that multiple screens share.

It also supplies functions that operate on runtime state.

It does two jobs:

```text
1. creates and owns ResearchRuntimeState
2. provides operations that change that state safely
```

That is why the name `Manager` is useful.

Therefore, runtime state is not like ViewModel state.

ViewModel state is usually used directly by the ViewModel:

In contrast, runtime state is closer to a repository-style object:

```text
it has a state data class
and it provides operations that safely manipulate that state
```

So the ViewModel does not directly rewrite runtime states by itself.

Instead, the ViewModel calls runtime operations:

```kotlin
runtimeStateManager.selectPatient(patientId)
runtimeStateManager.selectSession(sessionId)
runtimeStateManager.setDeviceConnected(true)
```

This is similar to how a ViewModel calls a repository:

```kotlin
measurementRepository.createPatient(...)
measurementRepository.saveMeasurement(...)
measurementRepository.exportSessionCsv(...)
```

Both are objects that a ViewModel can call.

Both hide internal details.

Both own operations.

But they are not the same kind of object:

```text
ResearchRuntimeStateManager
    -> temporary in-memory workflow state
    -> selected patient/session
    -> current connection/acquisition state
    -> latest live values

MeasurementRepository
    -> persistent/data operations
    -> Room database
    -> DAOs
    -> saved measurements/results
    -> export data
```

So runtime is not merely "shared ViewModel state."

It is a separate runtime data workflow layer.

### What functions belong in `ResearchRuntimeStateManager`?

Now that we understand the manager idea, we can ask a better design question:

```text
Which functions should live inside ResearchRuntimeStateManager?
```

The criterion is not only:

```text
How many screens share this function?
```

The better criterion is:

```text
Is this function an operation on ResearchRuntimeState?
```

其实道理很简单，既然这个状态变量是在ResearchRuntimeStateManager文件中私有且唯一的，而你又需要去修改这个状态变量，那凡是涉及修改这个状态变量的操作函数自然都需要放到这个文件中去，不然怎么看到这个状态变量并进行修改呢。其他的viewmodel则只需要调用这些函数就行了。

If a function directly owns or protects changes to `ResearchRuntimeState`, it can belong in `ResearchRuntimeStateManager`, even if currently only one ViewModel calls it.

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

This belongs in `ResearchRuntimeStateManager` because it modifies the shared `_runtimeState` variable safely.

It also protects a workflow rule:

```text
When a new patient is selected,
the previous selected session should be cleared.
```

That rule belongs to `ResearchRuntimeState`, because it protects the meaning of that state.

In contrast, a screen workflow function should stay in the ViewModel:

```kotlin
fun onCreatePatientClick() {
    // validate text field
    // call repository
    // update screen message
    // ask runtime manager to select the patient if needed
}
```

That function is mainly about a screen event, not just about changing `ResearchRuntimeState`.

So the split is:

```text
ViewModel
    -> handles screen events
    -> reads screen input
    -> calls repository
    -> calls runtime manager when runtime states change
    -> prepares UI messages/state for that screen

ResearchRuntimeStateManager
    -> owns ResearchRuntimeState
    -> updates temporary workflow values
    -> protects runtime workflow rules

MeasurementRepository
    -> owns data operations
    -> reads/writes persistent research data
    -> hides Room/DAO/export details from ViewModel
```

A function can belong in `ResearchRuntimeStateManager` even if only one ViewModel calls it today, as long as it is a runtime-state operation.

The mental model is:

```text
ViewModel
    -> state and actions for one screen

runtime manager
    -> temporary workflow state and runtime operations

repository/data
    -> permanent saved data and data operations
```

For now, we only prepare this folder as an architectural home.

We have placed shared runtime state and its operations inside `ResearchRuntimeStateManager`. But defining this class does not automatically make its state shared. If each ViewModel creates its own manager, each gets a separate state holder.

How do we make sure the ViewModels use the same manager instance?

---

## 11. How ViewModels share the same runtime objects

To share this runtime state, the ViewModels need access to the same `ResearchRuntimeStateManager` object. This raises two questions: who creates that object, and how do the ViewModels receive it?

Consider what happens if each ViewModel creates its own manager:

```kotlin
val runtimeStateManager = ResearchRuntimeStateManager()
```

Each call to `ResearchRuntimeStateManager()` creates a new instance with its own state holder.

That means:

```text
PatientListViewModel changes one ResearchRuntimeStateManager
MeasurementViewModel reads a different ResearchRuntimeStateManager
the selected patient/session is not truly shared
```

To share runtime state, both ViewModels must use the same runtime manager object.

The same idea applies to repositories:

```text
If two ViewModels need the same repository-backed data workflow,
they should use the same repository instance.
```

There are two common ways to make that happen:

```text
Singleton-style shared instance
    -> the class exposes one shared instance itself

Dependency injection
    -> some outside setup creates the object and passes it in
```

This section introduces both approaches; we are not implementing a full dependency-injection setup in this lesson.

### Singleton-style shared instance

Sometimes you may see shared-object code written like this:

```kotlin
class ResearchRuntimeStateManager {

    companion object {
        val instance: ResearchRuntimeStateManager by lazy {
            ResearchRuntimeStateManager()
        }
    }
}
```

This creates one shared `ResearchRuntimeStateManager` object that can be accessed through the class name:

```kotlin
val runtimeStateManager = ResearchRuntimeStateManager.instance
```

The pieces mean:

```text
companion object
    -> attach this value to the class itself
    -> call it with ResearchRuntimeStateManager.instance

val instance: ResearchRuntimeStateManager
    -> instance is a ResearchRuntimeStateManager value

by lazy { ResearchRuntimeStateManager() }
    -> do not create the object immediately
    -> create it the first time someone asks for it
    -> return the same object on later calls
```

So the flow is:

```text
App starts
    -> ResearchRuntimeStateManager.instance has not created anything yet

First call to ResearchRuntimeStateManager.instance
    -> lazy block runs
    -> ResearchRuntimeStateManager() creates the object
    -> that object is stored

Later calls to ResearchRuntimeStateManager.instance
    -> return the same stored object
```

This is a simple singleton-style pattern.

Singleton means:

```text
one shared object used from many places
```

For example:

```kotlin
class PatientListViewModel(
    private val runtimeStateManager: ResearchRuntimeStateManager =
        ResearchRuntimeStateManager.instance
) : ViewModel()
```

and:

```kotlin
class MeasurementViewModel(
    private val runtimeStateManager: ResearchRuntimeStateManager =
        ResearchRuntimeStateManager.instance
) : ViewModel()
```

Both `PatientListViewModel` and `MeasurementViewModel` use `ResearchRuntimeStateManager.instance`, so they receive the same shared object.

This can be acceptable for a small fake workflow or a learning project.

But be careful with this pattern in Android if the shared object needs:

```text
Context
Room database
device connection state
coroutines
lifecycle-aware behavior
```

For those cases, application-level setup or constructor-passed dependencies are usually cleaner.

### Dependency injection

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

For this tutorial, the important idea is the constructor pattern:

```kotlin
class MeasurementRepository(
    private val database: ResearchDatabase,
    private val deviceDataSource: DeviceDataSource,
    private val signalProcessor: SignalProcessor,
    private val modelRunner: ModelRunner
)
```

We are not fully adding dependency injection in Lesson 25.

But this is the architecture idea behind it:

```text
Do not make every class build all of its own tools.
Give classes the tools they need from the outside.
```

### Which style are we using for now?

In this beginner project, either style can work.

The singleton-style `instance` pattern is simple to understand:

```text
Use ResearchRuntimeStateManager.instance whenever you need the shared runtime manager.
```

Dependency injection is the cleaner long-term direction:

```text
Create the shared runtime manager or repository outside the ViewModel.
Pass the same object into every ViewModel that needs it.
```

For Lesson 25, the main goal is not to choose the final tool yet.

The main goal is to understand this rule:

```text
Shared runtime state requires a shared runtime manager instance.
Shared repository behavior requires a shared repository instance.
```

The file structure gives the shared object a home.

The object creation pattern decides how the rest of the app accesses that shared object.

---

## 12. `data` folder

The `data` folder contains the local database and repository.

This is where the lesson moves from:

```text
temporary runtime workflow state
```

to:

```text
persistent research data
```

So the difference is:

```text
runtime
    -> what is happening in the app right now
    -> selected patient/session
    -> connection/acquisition state
    -> latest live values

data
    -> what the app stores and loads
    -> patients
    -> sessions
    -> measurements
    -> results
```

```text
data
 ├── ResearchDatabase.kt
 ├── MeasurementRepository.kt
 ├── entity
 └── dao
```

This is where Room-related code will go.

For now, we only create the files.

We do not need to fully implement Room in Lesson 25.

That will be Lesson 27.

### Database access and model coupling are different concerns

A common question is: **If screens should receive data through a ViewModel, can a screen still use a class from `data/entity`, such as `PatientEntity`?**

To answer this clearly, separate two ideas:

```text
1. Which layer accesses the database?
2. Which model type does the UI depend on?
```

#### 1. Keep database access out of the UI

In this app, a composable should not query Room, call a DAO, or call a repository directly. Persistent data follows this path:

```text
Room database
    -> DAO
    -> Repository
    -> ViewModel
    -> UI state
    -> Composable screen
```

User actions travel back through callbacks:

```text
Composable screen
    -> callback
    -> ViewModel function
    -> Repository
    -> DAO / Room database
```

This is **data-access separation**. The UI displays values and reports events, while the data layer performs database operations.

#### 2. An entity can be passed to the UI, but that creates coupling

Passing a `PatientEntity` to a composable does not cause the composable to access Room. Reading `patient.patientCode` is only reading a Kotlin property; it does not execute a database query.

However, it makes the UI depend on the structure of a database entity. For example:

```kotlin
@Composable
fun PatientRow(patient: PatientEntity) {
    Text(patient.patientCode)
}
```

If `PatientEntity` changes because storage requirements change, this UI may also need to change. Therefore:

```text
Using PatientEntity in the UI does not break data-access separation,
but it does couple the UI to the database representation.
```

For a small application, that coupling may be an acceptable simplification. Separate models become more useful when the database and UI need different fields, types, names, or formats.

#### 3. Use a UI model when the UI has a different purpose

Later in this project, the patient list will use a focused UI model:

```kotlin
data class PatientListItem(
    val id: Long,
    val patientCode: String,
    val createdAt: Long
)
```

The ViewModel can map the database entity to that UI model:

```kotlin
fun PatientEntity.toPatientListItem(): PatientListItem {
    return PatientListItem(
        id = id,
        patientCode = patientCode,
        createdAt = createdAt
    )
}
```

The flow then becomes:

```text
Room database
    -> DAO returns PatientEntity
    -> Repository provides PatientEntity
    -> ViewModel maps it to PatientListItem
    -> UI state contains PatientListItem
    -> PatientListScreen displays PatientListItem
```

Now the screen depends on what the patient list needs to display rather than on the complete Room table representation. If the database structure changes but the UI contract remains the same, the mapper can absorb some or all of that change.

The mapper is a boundary between representations; it is not a guarantee that every database change affects only one function. A schema change may also require a Room migration, DAO changes, repository changes, and tests.

#### 4. Domain models and UI models are not the same thing

In a larger app, we might also introduce a storage-independent domain model such as `Patient`. A domain model represents concepts and rules used across the application. A UI model such as `PatientListItem` represents exactly what a particular screen needs.

For example, a value such as `createdAt` may remain a `Long` or another time type in a domain model, while the UI converts it to formatted text for display. A model containing display strings such as `"2026-09-01 10:15:30"` or `"91%"` is usually a UI model rather than a domain model.

We do not need a separate domain layer merely to follow the rule. For this beginner project, the important choice is:

```text
UI never accesses Room directly.
The repository and ViewModel control the data flow.
Use a separate UI model when it usefully protects the UI from storage details.
```

#### 5. Keep meaningful data types until the UI displays them

A UI model does not need to contain only display-ready strings. It should contain the values the UI needs in forms that preserve their meaning.

A useful general rule is:

```text
Keep IDs, numbers, timestamps, nullability, booleans, and states typed.
Convert them to labels, units, percentages, colors, and formatted text
when the UI displays them.
```

For this project:

| Value | Keep in the UI model | Example display conversion |
|---|---|---|
| Database ID | `Long` | `"Patient ID: 12"` |
| Timestamp | `Long` or `Long?` | `"21 Sep 2026, 14:30"` |
| Measurement | `Double` | `"2.438 V"` |
| Confidence | `Double` | `"91%"` |
| Repetition number | `Int` | `"Repetition 3"` |
| Optional value | A nullable type such as `Long?` | `"In progress"` when the value is `null` |
| Status or state | Prefer an enum | A label, color, or icon |

##### Keep IDs as IDs

Even if an ID appears inside text, keep it as a `Long` in the UI model:

```kotlin
data class PatientListItem(
    val id: Long,
    val patientCode: String,
    val createdAt: Long
)
```

The UI may display it:

```kotlin
Text("Patient ID: ${patient.id}")
```

It may also pass the same typed ID through a callback:

```kotlin
onClick = {
    onPatientClick(patient.id)
}
```

Converting the ID to a display string in the mapper would make it less useful for navigation, callbacks, and repository operations.

##### Keep measurements and confidence numeric

Keep measurement values as `Double`:

```kotlin
data class MeasurementUiModel(
    val rawValue: Double,
    val processedValue: Double
)
```

The UI can add precision and units when displaying them:

```kotlin
fun formatMeasurement(value: Double): String {
    return "%.3f V".format(value)
}
```

Likewise, keep model confidence numeric:

```kotlin
data class ResultUiModel(
    val label: String,
    val confidence: Double
)
```

The UI can format `0.91` as `"91%"`. Keeping the `Double` also lets the UI use the value for a progress indicator, threshold comparison, chart, or sort operation.

##### Preserve null when it has meaning

`SessionListItem` should keep `endedAt` nullable:

```kotlin
data class SessionListItem(
    val id: Long,
    val sessionName: String,
    val startedAt: Long,
    val endedAt: Long?
)
```

Here, `null` means that the session has not ended. The UI decides how to explain that state:

```kotlin
val endText = session.endedAt?.let(::formatTimestamp)
    ?: "In progress"
```

Converting `null` to `"In progress"` inside the mapper would discard the original meaning and make calculations or other presentation choices more difficult.

##### Prefer typed states to unrestricted strings

`MeasurementEntity` currently stores status as a `String`. As the app becomes stricter, a typed status can prevent spelling mistakes and invalid values:

```kotlin
enum class MeasurementStatus {
    OK,
    INVALID,
    NOISY,
    ERROR
}
```

The UI can map each status to an appropriate label, color, or icon. Do not replace the state itself with a color name such as `"Green"`; color is only one possible presentation of the state.

These are guidelines rather than a requirement that every UI model must contain raw values. A display-only model may deliberately contain formatted text. For this project, retaining meaningful types gives the UI more flexibility and keeps formatting decisions close to where values are displayed.

#### 6. Date formatting as a specific example

For this project, keep the creation time as a `Long` in both the Room entity and the UI model:

```kotlin
data class PatientListItem(
    val id: Long,
    val patientCode: String,
    val createdAt: Long
)
```

The entity-to-UI mapper copies the value without formatting it:

```kotlin
fun PatientEntity.toPatientListItem(): PatientListItem {
    return PatientListItem(
        id = id,
        patientCode = patientCode,
        createdAt = createdAt
    )
}
```

When a screen needs to display the date, format it in the UI layer. Put the reusable formatter in:

```text
ui/format/DateFormatters.kt
```

For example:

```kotlin
package com.example.researchapp.ui.format

import java.text.DateFormat
import java.util.Date

fun formatTimestamp(timestamp: Long): String {
    return DateFormat.getDateTimeInstance(
        DateFormat.MEDIUM,
        DateFormat.SHORT
    ).format(Date(timestamp))
}
```

The composable uses that function only when it needs display text:

```kotlin
@Composable
fun PatientListRow(
    patient: PatientListItem,
    onClick: () -> Unit
) {
    Card(onClick = onClick) {
        Text("Patient code: ${patient.patientCode}")
        Text("Created: ${formatTimestamp(patient.createdAt)}")
    }
}
```

The complete conversion is:

```text
PatientEntity.createdAt
    Long timestamp stored by Room
        |
        | mapper copies the Long
        v
PatientListItem.createdAt
    Long timestamp available to the UI
        |
        | UI formatter creates locale-aware display text
        v
Text displays the formatted date
```

This approach keeps the raw timestamp available for sorting and comparisons, while the UI controls how it is presented. It also avoids storing both a raw timestamp and a formatted copy in the UI model.

Do not put the full formatting expression directly inside `Text`. Calling a named function keeps the composable readable and lets several screens use the same formatting rule.

Date formatting could instead happen in an entity-to-UI mapper when a UI model deliberately contains a property such as `createdAtText: String`. That is a valid alternative, but it is not the choice used here. In this project, `PatientListItem` retains `createdAt: Long`, and the UI layer formats it for display.

Lessons 32 and 33 apply this choice by introducing `PatientListItem` and mapping `PatientEntity` values before exposing them to the patient-list screen.

---

## 13. `data/entity` folder

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

We will implement these properly in Lesson 26.

For now, the important idea is:

```text
Entity files describe what data we store.
```

---

## 14. `data/dao` folder

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

## 15. `MeasurementRepository.kt`

The repository is the bridge between the ViewModel and the data/device/processing/ML layers.

It is similar to `ResearchRuntimeStateManager` in one way:

```text
ViewModels call both of them.
```

But their jobs are different:

```text
ResearchRuntimeStateManager
    -> manages temporary workflow state
    -> selected session
    -> connection/acquisition state
    -> latest live values

MeasurementRepository
    -> coordinates data operations
    -> Room writes and queries
    -> device reads
    -> processing/ML/export work
```

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

---

## 16. `device` folder

The `device` folder contains:

```text
device
 ├── DeviceDataSource.kt
 └── FakeDeviceDataSource.kt
```

### `DeviceDataSource.kt`

This is the interface from Lesson 20:

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

## 17. `processing` folder

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

## 18. `ml` folder

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

## 19. `export` folder

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

## 20. What should we implement first?

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

Lesson 25 focuses mostly on step 1 and the file structure.

---

## 21. Why we start with placeholders

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

## 22. Common mistake: too much code in UI

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

## 23. Common mistake: starting with real Bluetooth too early

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

## 24. Common mistake: starting with real ML too early

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

## 25. Current skeleton after Lesson 25

After Lesson 25, your project should have this shape:

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
`-- ResearchRuntimeStateManager.kt
```

It may only show placeholder screens.

That is okay.

The goal of Lesson 25 is not to finish the app.

The goal is to create a clean foundation.

---

## 26. What you learned in Lesson 25

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

# Lesson 26 preview

In Lesson 26, we will start implementing the real project files.

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

This will turn the research data model from Lesson 17 into real Kotlin files inside the new project structure.
