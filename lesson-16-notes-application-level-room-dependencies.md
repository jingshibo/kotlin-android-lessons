# Lesson 16 notes - Application-level Room dependencies with ViewModel

Lesson 16 creates `MeasurementRepository` inside an `AndroidViewModel`. That is a useful beginner version because it introduces Room without adding too many new ideas at once.

This note explains a more reusable setup:

```text
Android creates ResearchApplication
    -> ResearchApplication provides one database
    -> ResearchApplication provides one repository
    -> a ViewModel factory receives the repository
    -> the factory creates ViewModel
    -> Compose receives the ViewModel
```

The setup is called **manual dependency injection**. We create each object in one clear place and pass it to the class that needs it. The core practices are very common in small or medium-sized apps:
   - Dependencies are created in one application-level location.
   - Room has one shared database instance per app process.
   - Repositories receive DAOs through constructors.
   - ViewModels receive repositories through constructors.
   - A factory creates ViewModels that have constructor dependencies.
   - Screens receive state and callbacks instead of accessing repositories.

---

## 1. The problem we are solving

In the beginner version, the repository creates its own database:

```kotlin
class MeasurementRepository(
    context: Context
) {
    private val database = Room.databaseBuilder(
        context.applicationContext,
        ResearchDatabase::class.java,
        "research_database"
    ).build()
}
```

This works, but ownership becomes less clear as the app grows. If several repositories or ViewModels create their own database objects, the app can accidentally create more than one expensive `RoomDatabase` instance.

We want one application-level place to answer these questions:

```text
Who creates the database?
Who creates the repository?
How does the ViewModel receive the repository?
```

For this note, the answer is `ResearchApplication`.

---

## 2. What is an Android `Application`?

`Application` is an Android framework class representing the current app process. Android creates its application object before it creates the app's activities.

It is different from an `Activity`:

| Object | Main responsibility | Typical lifetime |
|---|---|---|
| `ResearchApplication` | Hold application-level dependencies | As long as the app process exists |
| `MainActivity` | Host the app's UI | As long as that activity instance exists |
| `ResearchViewModel` | Hold screen state and handle screen events | As long as its ViewModel scope exists |

The application object is a suitable owner for objects shared across the app, such as a database and repository. It is not permanent: Android can stop the process and create a new application object later. The Room database file remains on storage unless app data is cleared or the app is uninstalled.

---

## 3. The dependency graph

Before writing code, identify what each object needs:

```text
ResearchDatabase
    needs application Context

MeasurementRepository
    needs MeasurementDao

ResearchViewModel
    needs MeasurementRepository

Compose UI
    needs ResearchViewModel state and event functions
```

The creation order therefore becomes:

```text
ResearchApplication
    -> ResearchDatabase
        -> MeasurementDao
            -> MeasurementRepository
                -> ResearchViewModel
                    -> Compose UI
```

An object passed into another object's constructor is called a **dependency**. For example, `MeasurementRepository` is a dependency of `ResearchViewModel`.

---

## 4. Make the repository receive a DAO

The repository should receive the database operation interface it needs:

```kotlin
class MeasurementRepository(
    private val measurementDao: MeasurementDao
) {
    suspend fun insertMeasurement(measurement: Measurement) {
        measurementDao.insertMeasurement(measurement)
    }

    suspend fun getAllMeasurements(): List<Measurement> {
        return measurementDao.getAllMeasurements()
    }

    suspend fun deleteAllMeasurements() {
        measurementDao.deleteAllMeasurements()
    }
}
```

The repository no longer needs `Context` and no longer decides how to construct Room. It only uses the `MeasurementDao` supplied to it.

This makes its responsibility clearer:

```text
ResearchApplication
    -> creates and connects dependencies

MeasurementRepository
    -> performs app-facing measurement data operations
```

It also makes the repository easier to test because a test can supply a test DAO or another implementation instead of constructing the real app database.

---

## 5. Create `ResearchApplication.kt`

Place `ResearchApplication.kt` in the app's main package, beside `MainActivity.kt` for this beginner structure:

```text
com.example.researchapp
 |-- MainActivity.kt
 |-- ResearchApplication.kt
 |-- data
 |-- repository
 |-- ui
 `-- viewmodel
```

Create the class:

```kotlin
package com.example.researchapp

import android.app.Application
import androidx.room3.Room
import com.example.researchapp.data.ResearchDatabase
import com.example.researchapp.repository.MeasurementRepository

class ResearchApplication : Application() {

    val database: ResearchDatabase by lazy {
        Room.databaseBuilder(
            applicationContext,
            ResearchDatabase::class.java,
            "research_database"
        ).build()
    }

    val measurementRepository: MeasurementRepository by lazy {
        MeasurementRepository(
            measurementDao = database.measurementDao()
        )
    }
}
```

Adjust package and import names to match the real project.

The application object now owns two dependency properties:

```text
database
    -> the shared ResearchDatabase instance

measurementRepository
    -> the shared repository using database.measurementDao()
```

For a single-process app, this application-scoped lazy property provides one Room database instance during that process. Room recommends reusing one database instance because each instance is relatively expensive.

---

## 6. What does `by lazy` do?

This declaration:

```kotlin
val database: ResearchDatabase by lazy {
    Room.databaseBuilder(...).build()
}
```

does not immediately run the block. It runs the block the first time code reads `database`, stores the result, and returns the same result on later reads.

The repository is also lazy:

```kotlin
val measurementRepository: MeasurementRepository by lazy {
    MeasurementRepository(
        measurementDao = database.measurementDao()
    )
}
```

When the repository is first requested, its block reads `database`. If the database has not yet been created, that first read creates it.

The timeline is:

```text
App process starts
    -> Android creates ResearchApplication
    -> database has not been requested
    -> repository has not been requested

MainActivity requests measurementRepository
    -> repository lazy block starts
    -> it requests database.measurementDao()
    -> database lazy block creates ResearchDatabase
    -> repository is created with the DAO

Later repository requests
    -> return the same repository object
```

Therefore, registering `ResearchApplication` does not mean Room performs database work immediately at process launch.

---

## 7. Register the class in `AndroidManifest.xml`

Creating the Kotlin class is not enough. Tell Android to use it by adding `android:name` to the existing `<application>` element:

```xml
<application
    android:name=".ResearchApplication"
    android:allowBackup="true"
    android:label="@string/app_name"
    android:theme="@style/Theme.ResearchApp">

    ...

</application>
```

The leading dot means the class is inside the app's base package. If the class is in another package, use its correct path, for example:

```xml
android:name=".app.ResearchApplication"
```

After registration, Android creates `ResearchApplication` automatically. Your code should not call:

```kotlin
ResearchApplication()
```

Doing that would create an ordinary object that Android does not manage as the current application instance.

---

## 8. Let the ViewModel receive the repository

Use a normal `ViewModel` whose constructor declares the dependency:

```kotlin
class ResearchViewModel(
    private val measurementRepository: MeasurementRepository
) : ViewModel() {

    var uiState by mutableStateOf(ResearchUiState())
        private set

    fun loadSavedMeasurements() {
        viewModelScope.launch {
            val measurements =
                measurementRepository.getAllMeasurements()

            uiState = uiState.copy(
                measurements = measurements
            )
        }
    }

    fun clearMeasurements() {
        viewModelScope.launch {
            measurementRepository.deleteAllMeasurements()

            uiState = uiState.copy(
                measurements = emptyList()
            )
        }
    }
}
```

The ViewModel no longer creates its repository:

```kotlin
// Avoid inside ResearchViewModel:
private val repository = MeasurementRepository(...)
```

Instead, the constructor makes the dependency explicit:

```text
To create ResearchViewModel,
you must provide MeasurementRepository.
```

**However, we do not provide a specific repository value here as the input. Instead, we use factory to connect repository to viewmodel.**

---

## 9. The ViewModel need a factory to provide the repository value

The Compose `viewModel()` function can create a ViewModel with an empty constructor:

It does not automatically know what value to provide here:

```kotlin
class ResearchViewModel(
    measurementRepository: MeasurementRepository
) : ViewModel()
```

A `ViewModelProvider.Factory` supplies those construction instructions. Using the current factory DSL, create this function in `ResearchViewModel.kt`:

```kotlin
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewmodel.initializer
import androidx.lifecycle.viewmodel.viewModelFactory

fun researchViewModelFactory(
    repository: MeasurementRepository
): ViewModelProvider.Factory = viewModelFactory {
    initializer {
        ResearchViewModel(
            measurementRepository = repository
        )
    }
}
```

The factory does one job:

```text
Provide MeasurementRepository,
create ResearchViewModel with that repository.
```

The factory does not store screen state and does not replace the ViewModel. It only explains how to construct it.

---

## 10. Connect everything in `MainActivity`

`MainActivity` can access the application object through its inherited `application` property:

```kotlin
val researchApplication =
    application as ResearchApplication
```

The cast means:

```text
Android exposes this property using the general Application type.
We know the manifest registered ResearchApplication,
so treat the object as ResearchApplication.
```

Now use its repository to create the factory:

```kotlin
class MainActivity : ComponentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val researchApplication =
            application as ResearchApplication

        val factory = researchViewModelFactory(
            repository = researchApplication.measurementRepository
        )

        setContent {
            val viewModel: ResearchViewModel = viewModel(
                factory = factory
            )

            ResearchApp(
                viewModel = viewModel
            )
        }
    }
}
```

Relevant imports include:

```kotlin
import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.lifecycle.viewmodel.compose.viewModel
```

Calling `viewModel(factory = factory)` does not create a new ViewModel on every recomposition. It asks the current `ViewModelStoreOwner` for the scoped instance and uses the factory when an instance must first be created.

---

## 11. What should `ResearchApp` and screens receive?

At the root of the UI, `ResearchApp` can use the ViewModel:

```kotlin
@Composable
fun ResearchApp(
    viewModel: ResearchViewModel
) {
    val uiState = viewModel.uiState

    ResearchScreen(
        uiState = uiState,
        onLoadSaved = viewModel::loadSavedMeasurements,
        onClear = viewModel::clearMeasurements
    )
}
```

The reusable screen receives state and callbacks:

```kotlin
@Composable
fun ResearchScreen(
    uiState: ResearchUiState,
    onLoadSaved: () -> Unit,
    onClear: () -> Unit
) {
    // Display uiState and connect callbacks to UI events.
}
```

Avoid retrieving `ResearchApplication` from deep inside reusable composables:

```kotlin
// Avoid inside an ordinary reusable screen:
val app = LocalContext.current.applicationContext as ResearchApplication
val repository = app.measurementRepository
```

That code hides the screen's dependencies and makes previews and tests harder. Resolve application-level dependencies near the app or screen entry point, then pass state and event functions downward.

---

## 12. The complete path for a button event

Suppose the user taps Clear:

```text
ResearchScreen invokes onClear
    -> ResearchApp supplied viewModel::clearMeasurements
    -> ResearchViewModel calls measurementRepository
    -> MeasurementRepository calls MeasurementDao
    -> Room changes the database
    -> ViewModel updates uiState
    -> Compose displays the updated state
```

`ResearchApplication` is not involved in every operation. Its role was to create and connect the long-lived dependencies. After construction, the ViewModel calls the repository normally.

---

## 13. `AndroidViewModel` versus constructor injection

Lesson 16 uses this beginner approach:

```kotlin
class ResearchViewModel(
    application: Application
) : AndroidViewModel(application) {

    private val repository =
        MeasurementRepository(application)
}
```

This note uses:

```kotlin
class ResearchViewModel(
    private val repository: MeasurementRepository
) : ViewModel()
```

Both can work. The second version has clearer separation because the ViewModel receives the exact dependency it needs instead of receiving Android `Application` and constructing the repository itself.

| Beginner `AndroidViewModel` version | Application-level dependency version |
|---|---|
| ViewModel receives `Application` | ViewModel receives `MeasurementRepository` |
| ViewModel constructs repository | `ResearchApplication` constructs repository |
| No custom factory shown | Factory explains ViewModel construction |
| Fewer concepts initially | Clearer dependencies as the app grows |

This note is a development of Lesson 16, not a claim that the beginner version was invalid.

---

## 14. Common mistakes

### Forgetting the manifest entry

If `ResearchApplication` is not registered, Android creates the default `Application`. This cast then fails:

```kotlin
application as ResearchApplication
```

Check that the `<application>` element contains the correct `android:name`.

### Creating `ResearchApplication` manually

Do not write:

```kotlin
val app = ResearchApplication()
```

Use the instance Android created and attached to the current process.

### Creating Room in a composable

Do not put `Room.databaseBuilder()` inside a composable. Recomposition is for describing UI, not constructing application-level infrastructure.

### Running heavy work in `Application.onCreate()`

Keep startup light. Creating dependency properties with `by lazy` delays their construction until needed. Do not perform long database reads, network requests, or other blocking work directly in `Application.onCreate()`.

### Passing the repository through every screen

Usually the ViewModel uses the repository. Screens receive observable state and callbacks rather than database-layer objects.

---

## 15. Manual dependency injection compared with Hilt

The setup in this note is **manual dependency injection**. We create the database and repository ourselves, write the ViewModel factory, and pass each dependency to the class that needs it.

**Hilt** is Android's recommended dependency-injection library. It is built on Dagger and generates much of this object-creation and connection code during compilation.

Hilt does not replace the architecture described in this note. Both approaches use the same dependency direction:

```text
ResearchDatabase
    -> provides MeasurementDao
    -> used by MeasurementRepository
    -> used by ResearchViewModel
    -> supplies state and events to Compose
```

The difference is who performs the wiring.

### Manual dependency injection

In the manual version, our code constructs and connects the objects:

```kotlin
class ResearchApplication : Application() {

    val database by lazy {
        Room.databaseBuilder(
            applicationContext,
            ResearchDatabase::class.java,
            "research_database"
        ).build()
    }

    val measurementRepository by lazy {
        MeasurementRepository(
            measurementDao = database.measurementDao()
        )
    }
}
```

We also create a factory for the ViewModel:

```kotlin
val factory = researchViewModelFactory(
    repository = researchApplication.measurementRepository
)

val viewModel: ResearchViewModel = viewModel(
    factory = factory
)
```

Every connection is visible, but we must maintain all of it ourselves.

### The same idea with Hilt

With Hilt, `ResearchApplication` marks the start of Hilt's application-level dependency container:

```kotlin
@HiltAndroidApp
class ResearchApplication : Application()
```

A Hilt module explains how to create objects that Hilt cannot construct directly, such as a Room database and its DAO:

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideResearchDatabase(
        @ApplicationContext context: Context
    ): ResearchDatabase {
        return Room.databaseBuilder(
            context,
            ResearchDatabase::class.java,
            "research_database"
        ).build()
    }

    @Provides
    fun provideMeasurementDao(
        database: ResearchDatabase
    ): MeasurementDao {
        return database.measurementDao()
    }
}
```

Because we own `MeasurementRepository`, Hilt can use constructor injection to create it:

```kotlin
class MeasurementRepository @Inject constructor(
    private val measurementDao: MeasurementDao
) {
    ...
}
```

Hilt can also generate the ViewModel factory:

```kotlin
@HiltViewModel
class ResearchViewModel @Inject constructor(
    private val measurementRepository: MeasurementRepository
) : ViewModel() {
    ...
}
```

After the activity or navigation destination is connected to Hilt, Compose can obtain the ViewModel using Hilt's integration instead of our manually written factory:

```kotlin
val viewModel: ResearchViewModel = hiltViewModel()
```

The annotations are instructions used by Hilt's generated code:

```text
@HiltAndroidApp
    -> create the application-level Hilt container

@Module and @Provides
    -> explain how to provide objects Hilt cannot construct itself

@Singleton
    -> reuse one provided instance in the application-level scope

@Inject constructor
    -> Hilt may create this class by supplying its constructor arguments

@HiltViewModel
    -> let Hilt create this ViewModel and its factory
```

This is only a comparison; using Hilt also requires its Gradle plugin, dependencies, annotation processing, and Android entry-point setup. Those details belong in a later dedicated lesson.

### Side-by-side comparison

| Responsibility | Manual version in this note | Hilt version |
|---|---|---|
| Application-level container | Properties in `ResearchApplication` | Generated from `@HiltAndroidApp` and modules |
| Create Room database | `by lazy { Room.databaseBuilder(...) }` | `@Provides @Singleton` function |
| Create repository | `MeasurementRepository(database.measurementDao())` | `@Inject constructor` |
| Create ViewModel | Our `researchViewModelFactory(...)` | Hilt-generated factory |
| Obtain ViewModel in Compose | `viewModel(factory = factory)` | `hiltViewModel()` |
| Object lifetimes | We manage them | Hilt manages predefined scopes |
| Setup cost | Little framework setup, more handwritten wiring | More initial configuration, less repeated wiring |
| Best fit | Small apps, learning, or a small dependency graph | Growing apps with many dependencies and scopes |

### Which one should this app use?

Manual dependency injection is a reasonable choice while the dependency graph is small. It keeps every connection visible and reinforces the architecture:

```text
classes declare dependencies in constructors
the app setup creates those dependencies
the UI does not search for repositories or databases
```

Consider moving to Hilt when the app has many repositories and ViewModels, repeated factories, different implementations for tests, feature-specific object lifetimes, or enough wiring that it becomes difficult to maintain manually.

The transferable skill is constructor injection. Whether the objects are connected by handwritten code or generated Hilt code, `ResearchViewModel` should still declare that it needs `MeasurementRepository`, and `MeasurementRepository` should still declare that it needs `MeasurementDao`.

See [Android dependency-injection guidance](https://developer.android.com/training/dependency-injection), [manual dependency injection](https://developer.android.com/training/dependency-injection/manual), and [dependency injection with Hilt](https://developer.android.com/training/dependency-injection/hilt-android).

---

## 16. Final mental model

```text
ResearchApplication
    application-level owner created by Android

by lazy
    creates a dependency on first access and reuses it

ResearchDatabase
    shared Room database instance for the app process

MeasurementRepository
    receives DAO and exposes app-facing data operations

ViewModel factory
    knows how to create ResearchViewModel with its repository

ResearchViewModel
    owns screen state and handles events

Compose screen
    receives state and callbacks
```

The most important path is:

```text
ResearchApplication creates dependencies
    -> factory supplies dependency to ViewModel
    -> ViewModel supplies state and callbacks to UI
```

References: [Manual dependency injection](https://developer.android.com/training/dependency-injection/manual), [Room database setup](https://developer.android.com/training/data-storage/room), [ViewModel factories](https://developer.android.com/topic/libraries/architecture/views/viewmodel/viewmodel-factories-views), and [Android startup performance](https://developer.android.com/topic/performance/issues/launch-time).
