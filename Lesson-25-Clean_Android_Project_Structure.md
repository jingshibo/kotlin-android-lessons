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
 |   |-- model
 |   |   |-- PatientListItem.kt
 |   |   `-- SessionListItem.kt
 |   |
 |   |-- mapper
 |   |   |-- PatientUiMappers.kt
 |   |   `-- SessionUiMappers.kt
 |   |
 |   `-- format
 |       |-- DateFormatters.kt
 |       `-- ValueFormatters.kt
 |
 |-- viewmodel
 |   |-- ResearchViewModel.kt
 |   `-- ResearchUiState.kt
 |
 |-- domain
 |   `-- model
 |       |-- PatientRecord.kt
 |       |-- SessionRecord.kt
 |       |-- MeasurementRecord.kt
 |       `-- ResultRecord.kt
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
 |   |-- mapper
 |   |   |-- PatientMappers.kt
 |   |   |-- SessionMappers.kt
 |   |   |-- MeasurementMappers.kt
 |   |   `-- ResultMappers.kt
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

It is the target structure for this research app. The separate models and mappers keep Room-specific classes out of the ViewModel and composable layers.

### Distinguishing `runtime` and `domain`

Two packages that need a clear distinction are:

```text
runtime
domain
```

The `runtime` package is for temporary workflow state and the operations that safely change that state.

It is separate from `viewmodel` because runtime state is not just one screen's UI state.

It is separate from `data` because runtime state is not permanent database data.

The `domain` package contains storage-independent application records. It is separate from `data` because domain models do not describe Room tables, and it is separate from `ui` because they do not describe one screen's presentation.

A useful mental model is:

```text
viewmodel
    -> single screen states and events

runtime
    -> runtime states and operations across screens

domain
    -> storage-independent application records

data
    -> persistent research data and data operations
```

### Distinguishing model and mapping folders

A **model** is a class that holds data in the form needed by one layer. A **mapper** is a function that usually converts a whole object from one model class into an object of another model class. For example, `PatientRecord` holds patient data, while `PatientEntity.toDomainModel()` converts one complete `PatientEntity` into a `PatientRecord`.

A **formatter** normally converts one individual value into text for display. It does not create another model class. For example, `formatTimestamp(createdAt)` converts one `Long` value into a date `String`.

```text
mapper
    whole model object -> another model object

formatter
    one typed value -> display text
```

The word `model` therefore appears in more than one folder because each layer may need a different representation of the same patient:

- `data/entity` contains the database representation. `PatientEntity` follows the Room table structure and annotations.
- `domain/model` contains the application representation. `PatientRecord` describes meaningful patient data without depending on Room or a particular screen.
- `ui/model` contains a screen-specific representation. `PatientListItem` may contain only the fields required by the patient list.

Mapper folders contain the conversion functions between those representations:

- `data/mapper` converts between database entities and domain models. It belongs to `data` because these functions know about Room-specific entity classes.
- `ui/mapper` converts domain models into UI models. It belongs to `ui` because these functions prepare data for a particular screen.
- `ui/format` is not another model layer. It normally contains reusable functions such as `formatTimestamp()` that turn individual typed values into display text.

The complete flow can look like this:

```text
PatientEntity
    -> data mapper
PatientRecord
    -> UI mapper
PatientListItem
    -> UI formatter, converts createdAt to text shown by the composable
```

A UI mapper can call a formatter when the UI model deliberately stores display-ready text such as `createdAtText: String`. UI models can also keep meaningful values typed, while formatting happens when the composable displays a value.

Here is the same distinction as a compact folder reference:

```text
data/entity
    -> Room table representations

data/mapper
    -> Room entities <-> domain models

domain/model
    -> storage-independent application records

ui/model
    -> values required by particular screens

ui/mapper
    -> domain models -> UI models

ui/format
    -> typed UI values -> display text
```

The important boundary is that a ViewModel may use `PatientRecord` and `PatientListItem`, but it should not need to import the Room-specific `PatientEntity`. A UI model is useful when a screen needs a smaller or presentation-specific shape, but it is not mandatory for every domain model. Section 13 explains mapper placement in more detail.

Section 4 shows the target structure so that each future file has a clear destination. You do not need to create every file before the lesson that introduces it.

---

## 5. Why these folders?

Each folder has a clear meaning.

| Folder | Purpose |
|---|---|
| `ui` | Compose screens, UI models, UI mappers, and display formatting |
| `viewmodel` | UI state and app flow |
| `domain` | Storage-independent application models |
| `runtime` | Shared temporary app/session state used across screens or ViewModels |
| `data` | Room database, DAOs, entities, entity/domain mappers, and repository |
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

The `ui` folder contains visible screens and the supporting models, mappers, and formatters used to present their data.

```text
ui
 ├── ResearchApp.kt
 ├── PatientListScreen.kt
 ├── PatientDetailScreen.kt
 ├── MeasurementScreen.kt
 └── ResultScreen.kt
```

The supporting folders are:

```text
ui
 |-- model
 |   |-- PatientListItem.kt
 |   `-- SessionListItem.kt
 |-- mapper
 |   |-- PatientUiMappers.kt
 |   `-- SessionUiMappers.kt
 `-- format
     |-- DateFormatters.kt
     `-- ValueFormatters.kt
```

The screen files render UI. Files in `ui/model` contain screen-focused data classes. Files in `ui/mapper` contain functions that convert whole domain-model objects into UI-model objects. Files in `ui/format` contain functions that convert individual typed values, such as timestamps or measurements, into display text.

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

This is where Room-related code and the entity/domain mapping boundary will go. The complete data package also includes:

```text
data/mapper
    -> PatientMappers.kt
    -> SessionMappers.kt
    -> MeasurementMappers.kt
    -> ResultMappers.kt
```

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
    -> data mapper
    -> domain model returned by Repository
    -> ViewModel builds UI state
    -> UI state
    -> Composable screen
```

User actions travel back through callbacks:

```text
Composable screen
    -> callback
    -> ViewModel function
    -> Repository
    -> data mapper
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

#### 3. Use a domain model to keep Room out of the ViewModel

For this research app, we choose the stricter boundary: the repository does not expose `PatientEntity` outside the data layer. It maps the entity to a storage-independent model:

```kotlin
data class PatientRecord(
    val patientId: Long,
    val patientCode: String,
    val notes: String,
    val createdAt: Long
)
```

`PatientRecord` belongs in `domain/model`. It contains application data without Room annotations or database-specific behavior.

The repository performs the conversion before returning data:

```text
Room database
    -> DAO returns PatientEntity
    -> data mapper converts it to PatientRecord
    -> Repository returns PatientRecord
    -> ViewModel receives PatientRecord
```

The ViewModel therefore does not import `PatientEntity`, a DAO, or the Room database. If the storage representation changes while the repository contract remains stable, the ViewModel can continue using the same `PatientRecord`.

#### 4. Use a UI model when a screen has a narrower purpose

A domain model represents application data. A UI model represents exactly what a particular screen needs.

For example, the patient-list screen can use:

```kotlin
data class PatientListItem(
    val id: Long,
    val patientCode: String,
    val createdAt: Long
)
```

A UI mapper converts the domain model:

```kotlin
fun PatientRecord.toPatientListItem(): PatientListItem {
    return PatientListItem(
        id = this.patientId,
        patientCode = this.patientCode,
        createdAt = this.createdAt
    )
}
```

The resulting flow is:

```text
PatientEntity
    -> data mapper
PatientRecord
    -> Repository
ViewModel
    -> UI mapper
PatientListItem
    -> PatientListScreen
```

The entity, domain model, and UI model have different responsibilities even when some of their properties currently look similar. A separate model is useful when it creates a meaningful boundary; it is not a requirement to duplicate every class mechanically.

A mapper can absorb representation changes when its input or output contract remains stable. It does not guarantee that every database change affects only one function. A schema change may also require a Room migration, DAO changes, repository changes, and tests.
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

The domain-to-UI mapper copies the value without formatting it:

```kotlin
fun PatientRecord.toPatientListItem(): PatientListItem {
    return PatientListItem(
        id = this.patientId,
        patientCode = this.patientCode,
        createdAt = this.createdAt
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
        | data mapper copies the Long
        v
PatientRecord.createdAt
    Long timestamp exposed by the repository
        |
        | UI mapper copies the Long
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

Date formatting could instead happen in a domain-to-UI mapper when a UI model deliberately contains a property such as `createdAtText: String`. That is a valid alternative, but it is not the choice used here. In this project, `PatientListItem` retains `createdAt: Long`, and the UI layer formats it for display.

Section 13 explains where the data and UI mappers belong and how the repository keeps Room entities away from the ViewModel.

---

## 13. Where mapper functions belong

Mapper placement becomes easier when we first identify which representations the function connects.

For patient data, this app uses three representations:

| Representation | Example type | Owner | Purpose |
|---|---|---|---|
| Database entity | `PatientEntity` | `data/entity` | Represents a Room table row |
| Domain model | `PatientRecord` | `domain/model` | Represents patient data without Room or UI details |
| UI model | `PatientListItem` | `ui/model` | Contains the values required by one screen |

The mapper belongs near the layer-specific representation it is protecting the rest of the app from.

### Entity/domain mappers belong in `data/mapper`

Create:

```text
data/mapper/PatientMappers.kt
```

That file contains both mapping directions:

```kotlin
package com.example.researchapp.data.mapper

import com.example.researchapp.data.entity.PatientEntity
import com.example.researchapp.domain.model.PatientRecord

fun PatientEntity.toDomainModel(): PatientRecord {
    return PatientRecord(
        patientId = this.id,
        patientCode = this.patientCode,
        notes = this.notes,
        createdAt = this.createdAt
    )
}

fun PatientRecord.toEntity(): PatientEntity {
    return PatientEntity(
        id = this.patientId,
        patientCode = this.patientCode,
        notes = this.notes,
        createdAt = this.createdAt
    )
}
```

These functions belong in the data layer because both know about `PatientEntity`, which is a Room storage representation.

This remains true for:

```kotlin
fun PatientRecord.toEntity(): PatientEntity
```

The receiver is `PatientRecord`, but the function still creates and depends on `PatientEntity`. The receiver type controls how an extension function is called; it does not decide which architectural layer owns the function.

[Lesson 3, Section 6](lesson-03-classes-null-safety-and-enums.md#6-extension-functions) explains extension-function syntax and the meaning of `this`. In the first mapper, `this` is the `PatientEntity` being converted. In the second mapper, `this` is the `PatientRecord` being converted.

### The repository performs the data-boundary conversion

The repository calls the DAO and immediately converts entities before returning data to the rest of the app. A simplified example is:

```kotlin
class MeasurementRepository(
    private val patientDao: PatientDao
) {
    suspend fun getPatients(): List<PatientRecord> {
        return patientDao.getAllPatients()
            .map { entity ->
                entity.toDomainModel()
            }
    }

    suspend fun savePatient(patient: PatientRecord) {
        patientDao.insertPatient(
            patient.toEntity()
        )
    }
}
```

The exact DAO method names may differ when Room is implemented, but the boundary remains the same:

```text
DAO speaks in entities.
Repository exposes domain models.
```

After this conversion, a ViewModel can use:

```kotlin
val patients: List<PatientRecord> = repository.getPatients()
```

The ViewModel does not need to import:

```text
PatientEntity
PatientDao
ResearchDatabase
```

### Domain/UI mappers belong in `ui/mapper`

If a screen needs a smaller or differently shaped model, create:

```text
ui/mapper/PatientUiMappers.kt
```

For example:

```kotlin
package com.example.researchapp.ui.mapper

import com.example.researchapp.domain.model.PatientRecord
import com.example.researchapp.ui.model.PatientListItem

fun PatientRecord.toPatientListItem(): PatientListItem {
    return PatientListItem(
        id = this.patientId,
        patientCode = this.patientCode,
        createdAt = this.createdAt
    )
}
```

This mapper belongs to the UI layer because its output, `PatientListItem`, exists for a screen. It knows about the domain model and UI model, but it does not know about Room.

A list-item UI model is a screen-specific projection of the application data. It contains the fields and item-specific UI state required to render that item on screen.

The UI model is normally **prepared by the ViewModel and rendered by the screen**. It is called a UI model because it is designed for the screen, not because only a composable is allowed to use it.

The flow is:

```text
Repository returns PatientRecord
    -> ViewModel calls the UI mapper
ViewModel creates PatientListItem
    -> ViewModel stores it in PatientsUiState
Screen reads PatientsUiState
    -> Composable renders PatientListItem
```

For example, the screen state can contain a list of UI models:

```kotlin
data class PatientsUiState(
    val patients: List<PatientListItem> = emptyList()
)
```

The ViewModel maps the repository results while preparing that state:

```kotlin
val patientItems = repository.getPatients()
    .map { patient ->
        patient.toPatientListItem()
    }

uiState = uiState.copy(
    patients = patientItems
)
```

The screen then consumes the prepared UI models:

```kotlin
@Composable
fun PatientsScreen(uiState: PatientsUiState) {
    LazyColumn {
        items(uiState.patients) { patientItem ->
            PatientRow(patient = patientItem)
        }
    }
}
```

The ViewModel is not displaying `PatientListItem`; it is preparing the state that the screen will display. Calling a plain mapper function does not make the ViewModel depend on Compose UI functions.

In this project structure, `viewmodel` and `ui` are separate packages, but both are on the presentation side of the app. The name `ui/mapper` describes the mapper's destination: it creates a model for the UI. In a feature-based structure, the ViewModel, UI state, UI model, mapper, and screen could instead live together under a package such as `patients`.

### Display formatting belongs in `ui/format`

A mapper usually accepts a whole model object and returns an object of another model class. A formatter usually accepts one value and returns text intended for display.

The `ui/format` folder therefore normally contains functions, not involving another set of data classes:

```kotlin
fun formatTimestamp(timestamp: Long): String

fun formatPercentage(value: Double): String
```

For example:

```text
PatientRecord -> PatientListItem
    UI mapper

Long timestamp -> "22 Sep 2026, 14:30"
    UI formatter
```

Put reusable date, measurement, and percentage formatting functions under `ui/format`. A UI mapper may call one of these functions if it creates a display-ready property such as `createdAtText`. In practice, timestamps and measurements can also remain typed in UI models and are formatted when they are displayed, as explained in Section 12. 

### Simplified and stricter designs are both possible

The application does not automatically need a different model class in every layer. Add a boundary when it prevents unwanted coupling or gives a layer a meaningfully different representation.

The shortest design allows a Room entity to travel beyond the data layer:

```text
PatientEntity
    -> Repository
    -> ViewModel
    -> Composable
```

The ViewModel is not querying Room directly, but it still depends on a Room-specific class. This can be acceptable for a very small prototype, but database changes can then affect the ViewModel and UI.

A cleaner design introduces a domain boundary but lets the UI use the domain model directly:

```text
PatientEntity
    -> data mapper
PatientRecord
    -> Repository
    -> ViewModel
    -> Composable
```

In this version, `PatientRecord` serves both the application logic and the UI. This is a good choice when the screen needs roughly the same fields and structure as the domain model. There is no need to create `PatientListItem` merely to duplicate every property.

For example, the screen state may hold domain models directly:

```kotlin
data class PatientsUiState(
    val patients: List<PatientRecord> = emptyList()
)
```

A separate UI model becomes useful when the screen needs a meaningfully different representation:

- only a small subset of a large domain model
- combined or derived values
- item-specific UI state such as `isSelected`
- a different structure designed for that screen
- stronger isolation from changes to the domain model

The flow then includes a UI mapper:

```text
PatientEntity
    -> data mapper
PatientRecord
    -> Repository
    -> ViewModel
    -> UI mapper
PatientListItem
    -> Composable
```

This research app keeps Room entities behind the data boundary. It can use `PatientRecord` directly on simple screens and introduce a screen-specific model such as `PatientListItem` when the presentation needs justify one.

The important distinction is:

```text
Required: Room entity -> domain model
    part of this project's data boundary

Optional: domain model -> UI model
    add it when the screen needs a different representation
```

Combining the domain and UI representations therefore means allowing the UI to use `PatientRecord`. It does not mean allowing the UI to use the Room-specific `PatientEntity`.

### Mapper placement reference

| Conversion | File location | Reason |
|---|---|---|
| `PatientEntity -> PatientRecord` | `data/mapper/PatientMappers.kt` | Knows about a Room entity |
| `PatientRecord -> PatientEntity` | `data/mapper/PatientMappers.kt` | Creates a Room entity |
| `PatientRecord -> PatientListItem` | `ui/mapper/PatientUiMappers.kt` | Creates a screen-specific model when one is needed |
| `Long -> formatted date String` | `ui/format/DateFormatters.kt` | Creates display text |
| `Double -> value with units` | `ui/format/ValueFormatters.kt` | Creates display text |

The practical rule is:

```text
If a mapper mentions a Room entity, keep it in the data layer.
If a mapper creates a screen-specific model, keep it in the UI layer.
Keep Room entities out of the ViewModel when using the domain boundary.
A separate UI model is optional when the domain model already fits the screen.
```

---

## 14. `data/entity` folder

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

## 15. `data/dao` folder

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

## 16. `MeasurementRepository.kt`

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
    -> maps Room entities to and from domain models
    -> device reads
    -> processing/ML/export work
```

For persistent records, the repository is also the boundary that prevents Room entities from leaking into the ViewModel:

```text
DAO returns entity
    -> repository uses data mapper
    -> repository returns domain model
```

### Repository data type

The repository implementation communicates with the DAO using entities, but its **public functions should normally communicate with the ViewModel using domain models**.

```kotlin
class MeasurementRepository(
    private val patientDao: PatientDao
) {
    suspend fun getPatients(): List<PatientRecord> {
        return patientDao.getAll()
            .map { entity ->
                entity.toDomainModel()
            }
    }

    suspend fun savePatient(patient: PatientRecord) {
        patientDao.insert(
            patient.toEntity()
        )
    }
}
```

Therefore, in the usual design:

```text
Repository input:
    domain model

Repository output:
    domain model

DAO input:
    entity

DAO output:
    entity
```

In comparison, room DAO functions generally work with entity classes because entities represent database tables:

```kotlin
@Dao
interface PatientDao {

    @Insert
    suspend fun insert(patient: PatientEntity)

    @Query("SELECT * FROM patients")
    suspend fun getAll(): List<PatientEntity>
}
```

Room DAOs can also return selected columns, scalar values, or special database projections, but those are still data-layer representations.


### Saving and loading flow

The complete flow is:

```text
Saving:

ViewModel
    -> PatientRecord
Repository
    -> converts PatientRecord to PatientEntity
DAO
    -> saves PatientEntity
Database
```

```text
Loading:

Database
    -> PatientEntity
DAO
    -> returns PatientEntity
Repository
    -> converts PatientEntity to PatientRecord
ViewModel
    -> receives PatientRecord
```

The repository itself knows both representations because it performs the boundary conversion:

```text
ViewModel knows:
    PatientRecord
    MeasurementRepository

Repository knows:
    PatientRecord
    PatientEntity
    PatientDao
    mapper functions

DAO knows:
    PatientEntity

Composable knows:
    PatientRecord or PatientListItem
```

### Repository inputs do not always need to be complete models

Some operations naturally accept a domain model:

```kotlin
suspend fun savePatient(patient: PatientRecord)
```

Others only need a particular value:

```kotlin
suspend fun getPatient(patientId: Long): PatientRecord?

suspend fun voidPatient(
    patientId: Long,
    reason: String
)

suspend fun deletePatient(patientId: Long)
```

You should not construct an entire `PatientRecord` when an operation only requires an ID.

A repository query normally returns a domain model or a collection or stream of domain models. Other operations may instead return `Unit`, a newly created ID, or an operation result.

A more precise rule is:

> A repository's public API should use domain-level models and values. Room entities should remain internal to the data layer.

UI models such as `PatientListItem` should not be passed into the repository.

### The repository is a data boundary

A repository is more than merely a bridge between a database and ViewModel. It represents the application's data boundary and could later coordinate Room, remote APIs, files, or caches without forcing the ViewModel to change.

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

### One repository or several?

At this stage, the project contains only:

```text
data/MeasurementRepository.kt
```

Using one repository is acceptable as a temporary, beginner-friendly starting point. It reduces the number of classes while the project skeleton is being built.

One repository may remain sufficient when:

- the application is small
- its records belong to one closely related data area
- the repository's public API remains focused
- splitting it would only create several tiny classes that always change together

The responsibility list above is intentionally broad for this early learning stage. As the application grows, permanently putting all those operations in one class would give that class several unrelated reasons to change. Its name would also become misleading because it would manage much more than measurements.

Do not automatically create one repository for every Room table. Instead, split repositories around a **coherent area of data and operations**. A likely future structure is:

```text
PatientRepository
    -> create, update, find, and void patients

SessionRepository
    -> create and load research sessions

MeasurementRepository
    -> save and retrieve measurements

ResultRepository
    -> save and retrieve analysis or prediction results
```

Closely related records can still share a repository. For example, if results always belong to measurements and are always loaded with them, `MeasurementRepository` could manage both. The split should follow meaningful responsibility, not mechanically copy the database table list.

Some operations in the earlier broad project structure list are not primarily repository responsibilities:

```text
DeviceDataSource
    -> communicates with the device

SignalProcessor
    -> processes measurements

ModelRunner
    -> runs ML inference

ExportFormatter
    -> creates export text
```

**When one user action must coordinate several of these components, that workflow can belong in a use-case or coordinator class rather than making one repository own every operation:**

```kotlin
class RunMeasurementUseCase(
    private val deviceDataSource: DeviceDataSource,
    private val measurementRepository: MeasurementRepository,
    private val signalProcessor: SignalProcessor,
    private val modelRunner: ModelRunner,
    private val resultRepository: ResultRepository
)
```

The practical direction for this project is:

```text
Lesson 25 skeleton
    -> one MeasurementRepository is acceptable temporarily

As persistent-data responsibilities grow
    -> split repositories by coherent data responsibility

For workflows spanning several components
    -> use a use-case or coordinator
```

If the application deliberately keeps one repository for all persistent research records, `ResearchRepository` would be a clearer name than `MeasurementRepository`.

---

## 17. `device` folder

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

## 18. `processing` folder

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

## 19. `ml` folder

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

## 20. `export` folder

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

## 21. What should we implement first?

Do not implement everything at once.

A good Direction A order is:

```text
1. Create project folders
2. Create placeholder screen files
3. Create domain model files
4. Create entity files
5. Create entity/domain mapper files
6. Create DAO files
7. Create Room database
8. Create repository boundaries that expose domain models
9. Create UI models, UI mappers, UiState, and ViewModel
10. Connect fake device source
11. Add processing
12. Add fake ML
13. Add export
14. Test the complete fake workflow
```

Lesson 25 focuses mostly on step 1 and the file structure.

---

## 22. Why we start with placeholders

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

## 23. Common mistake: too much code in UI

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

## 24. Common mistake: starting with real Bluetooth too early

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

## 25. Common mistake: starting with real ML too early

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

## 26. Current skeleton after Lesson 25

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

The complete target also includes a domain-model folder and explicit mapping folders:

```text
domain/model
    -> PatientRecord and other storage-independent records

data/mapper
    -> entity/domain conversion

ui/mapper
    -> domain/UI conversion

ui/format
    -> display formatting
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

## 27. What you learned in Lesson 25

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

Room, entity/domain mappers, and repository
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

The model boundaries are:

```text
data/mapper
    -> Room entities to and from domain models

domain/model
    -> storage-independent application records

ui/mapper
    -> domain models to screen-specific UI models

ui/format
    -> typed values to display text
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
Is this a UI model, UI mapper, or display formatter?
Is this state-management code?
Is this a storage-independent domain model?
Is this shared runtime state used across screens/ViewModels?
Is this database code?
Does this mapper mention a Room entity and therefore belong in data?
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

Those entity files are the storage-side starting point. Before they are exposed to a ViewModel, the project should also add the corresponding domain models and `data/mapper` functions described in Section 13.
