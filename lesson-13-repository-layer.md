# Lesson 13 — Repository Layer: Separating Data Logic

In Lesson 12, we improved the app state model.

Instead of only using:

```kotlin
val isAcquiring: Boolean = false
```

we introduced clearer states:

```kotlin
enum class DeviceConnectionState {
    DISCONNECTED,
    CONNECTING,
    CONNECTED,
    ERROR
}

enum class AcquisitionState {
    IDLE,
    RECORDING,
    STOPPED,
    ERROR
}
```
Device disconnected
 ↓
Connect device
 ↓
Device connected
Start acquisition
 ↓
Recording
 ↓
Stop acquisition
```

Now we improve the **app structure**.


```text
UI state
device connection simulation
acquisition flow
fake measurement generation
file saving
file loading
error messages
```

This is okay for learning, but as the app grows, the ViewModel can become too large.

So in Lesson 13, we introduce a new idea:

```text
Repository layer
```

The Android architecture guide describes repository classes as part of the data layer, and recommends creating repository classes for the different types of data handled by the app. citeturn141600search1 Android’s data-layer guidance also describes repositories as entry points to app data, helping different architecture layers scale independently. citeturn141600search2
```text
The ViewModel controls the screen state and user flow.
Our current ViewModel is becoming like this:

```text
ResearchViewModel
 ├── update sample ID
 ├── disconnect device
 ├── start acquisition
 ├── stop acquisition
```

This works, but it mixes different responsibilities.
 ↓
MeasurementRepository
 ↓
Storage / fake data / real device later
```

So the ViewModel becomes more focused on:

```text
What should the screen show?
What should happen when the user clicks a button?
What is the current app state?
```

The Repository becomes responsible for:

```text
How do we create a measurement?
How do we save measurements?
How do we load measurements?
Where does the data come from?
```

---

## 2. A simple mental model

Think of the repository as a middle layer.

Without repository:

```text
ViewModel
 ├── UI state
 ├── fake data generation
 ├── file saving
 ├── file loading
 └── later Bluetooth / Wi-Fi / database / ML
```

With repository:

```text
ViewModel
 ├── UI state
 └── calls repository

Repository
 ├── fake data generation
 ├── file saving
 ├── file loading
 └── later Bluetooth / Wi-Fi / database / ML
```

The ViewModel does not need to know the details.

It only asks:

```text
Give me a new measurement.
Save these measurements.
Load saved measurements.
```

---

## 3. What repository should we create?

For our current app, the most useful repository is:

```kotlin
MeasurementRepository
```

Why this name?

Because the app is mainly handling measurement data.

For now, this repository will handle:

```text
creating simulated measurements
saving measurements
loading measurements
```

Later, it can be expanded or connected to:

```text
Room database
Bluetooth data source
Wi-Fi data source
signal processing pipeline
ML inference result storage
```

But not yet.

We keep Lesson 13 simple.

---

## 4. First version of `MeasurementRepository`

Create a new Kotlin file:

```text
MeasurementRepository.kt
```

At this stage, you can place it in the same package as your other files.

Later, when the project becomes larger, you might use folders such as:

```text
data/
domain/
ui/
```

But for now, simple file separation is enough.

The first version:

```kotlin
class MeasurementRepository {

    fun createSimulatedMeasurement(
        sampleId: String,
        repetition: Int
    ): Measurement {
        val value = Random.nextDouble(0.0, 5.0)

        return Measurement(
            sampleId = sampleId,
            repetition = repetition,
            value = value,
            timestamp = System.currentTimeMillis(),
            status = "OK"
        )
    }
}
```

You need:

```kotlin
import kotlin.random.Random
```

This moves fake measurement generation out of the ViewModel.

Previously, the ViewModel did this:

```kotlin
val value = Random.nextDouble(0.0, 5.0)

val newMeasurement = Measurement(
    sampleId = uiState.sampleId,
    repetition = uiState.measurements.size + 1,
    value = value,
    timestamp = System.currentTimeMillis(),
    status = "OK"
)
```

Now the ViewModel can simply ask the repository:

```kotlin
val newMeasurement = measurementRepository.createSimulatedMeasurement(
    sampleId = uiState.sampleId,
    repetition = uiState.measurements.size + 1
)
```

This is cleaner.

---

## 5. Add save and load to the repository

In Lesson 9, we saved and loaded measurements from internal storage. The uploaded tutorial specifically introduced internal storage as a beginner-friendly auto-save method. fileciteturn1file0L397-L405

Now we move those calls into the repository.

```kotlin
class MeasurementRepository {

    fun createSimulatedMeasurement(
        sampleId: String,
        repetition: Int
    ): Measurement {
        val value = Random.nextDouble(0.0, 5.0)

        return Measurement(
            sampleId = sampleId,
            repetition = repetition,
            value = value,
            timestamp = System.currentTimeMillis(),
            status = "OK"
        )
    }

    suspend fun saveMeasurements(
        context: Context,
        measurements: List<Measurement>
    ) {
        withContext(Dispatchers.IO) {
            saveMeasurementsToInternalStorage(
                context = context,
                measurements = measurements
            )
        }
    }

    suspend fun loadMeasurements(
        context: Context
    ): List<Measurement> {
        return withContext(Dispatchers.IO) {
            loadMeasurementsFromInternalStorage(context)
        }
    }
}
```

Imports:

```kotlin
import android.content.Context
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import kotlin.random.Random
```

Now the repository handles background file work.

The ViewModel no longer needs to directly call:

```kotlin
saveMeasurementsToInternalStorage(...)
```

or:

```kotlin
loadMeasurementsFromInternalStorage(...)
```

Instead, it calls:

```kotlin
measurementRepository.saveMeasurements(...)
```

and:

```kotlin
measurementRepository.loadMeasurements(...)
```

---

## 6. Why are `saveMeasurements()` and `loadMeasurements()` marked `suspend`?

They are marked as:

```kotlin
suspend fun saveMeasurements(...)
```

and:

```kotlin
suspend fun loadMeasurements(...)
```

because they do background work.

Inside them, we use:

```kotlin
withContext(Dispatchers.IO) {
    ...
}
```

This means:

```text
Run file input/output work on the IO dispatcher.
```

This follows the idea from Lesson 10:

```text
Do not block the UI with file saving or loading.
```

---

## 7. Add repository to the ViewModel

Previously, the ViewModel looked like:

```kotlin
class ResearchViewModel : ViewModel() {
    ...
}
```

Now we give it a repository:

```kotlin
class ResearchViewModel(
    private val measurementRepository: MeasurementRepository = MeasurementRepository()
) : ViewModel() {
    ...
}
```

This means:

```text
ResearchViewModel uses MeasurementRepository.
```

The default value:

```kotlin
MeasurementRepository()
```

keeps things beginner-friendly for now.

Later, when the app becomes more professional, we may use dependency injection. But we do not need that yet.

---

## 8. Update `addSimulatedMeasurement()`

Before Lesson 13, the ViewModel created fake data directly:

```kotlin
private fun addSimulatedMeasurement(context: Context) {
    val value = Random.nextDouble(0.0, 5.0)

    val newMeasurement = Measurement(
        sampleId = uiState.sampleId,
        repetition = uiState.measurements.size + 1,
        value = value,
        timestamp = System.currentTimeMillis(),
        status = "OK"
    )

    val updatedMeasurements = uiState.measurements + newMeasurement

    uiState = uiState.copy(
        measurements = updatedMeasurements,
        latestValue = value
    )

    saveMeasurementsAsync(
        context = context,
        measurements = updatedMeasurements
    )
}
```

Now it becomes:

```kotlin
private fun addSimulatedMeasurement(context: Context) {
    val newMeasurement = measurementRepository.createSimulatedMeasurement(
        sampleId = uiState.sampleId,
        repetition = uiState.measurements.size + 1
    )

    val updatedMeasurements = uiState.measurements + newMeasurement

    uiState = uiState.copy(
        measurements = updatedMeasurements,
        latestValue = newMeasurement.value
    )

    saveMeasurementsAsync(
        context = context,
        measurements = updatedMeasurements
    )
}
```

This is better because the ViewModel no longer cares how a measurement is created.

It only receives a `Measurement`.

The mental model becomes:

```text
ViewModel:
I need a new measurement.

Repository:
Here is a simulated measurement.
```

---

## 9. Update `saveMeasurementsAsync()`

Before:

```kotlin
private fun saveMeasurementsAsync(
    context: Context,
    measurements: List<Measurement>
) {
    viewModelScope.launch {
        try {
            withContext(Dispatchers.IO) {
                saveMeasurementsToInternalStorage(
                    context = context,
                    measurements = measurements
                )
            }

            uiState = uiState.copy(
                message = "Measurements auto-saved"
            )
        } catch (e: Exception) {
            uiState = uiState.copy(
                message = "Auto-save failed"
            )
        }
    }
}
```

Now:

```kotlin
private fun saveMeasurementsAsync(
    context: Context,
    measurements: List<Measurement>
) {
    viewModelScope.launch {
        try {
            measurementRepository.saveMeasurements(
                context = context,
                measurements = measurements
            )

            uiState = uiState.copy(
                message = "Measurements auto-saved"
            )
        } catch (e: Exception) {
            uiState = uiState.copy(
                message = "Auto-save failed"
            )
        }
    }
}
```

The ViewModel no longer knows the internal details of saving.

It simply says:

```text
Repository, please save these measurements.
```

---

## 10. Update `loadSavedMeasurements()`

Before:

```kotlin
fun loadSavedMeasurements(context: Context) {
    viewModelScope.launch {
        uiState = uiState.copy(
            isLoading = true,
            message = "Loading saved measurements..."
        )

        try {
            val loadedMeasurements = withContext(Dispatchers.IO) {
                loadMeasurementsFromInternalStorage(context)
            }

            uiState = uiState.copy(
                measurements = loadedMeasurements,
                isLoading = false,
                message = "Loaded ${loadedMeasurements.size} saved measurements"
            )
        } catch (e: Exception) {
            uiState = uiState.copy(
                isLoading = false,
                message = "Could not load saved measurements"
            )
        }
    }
}
```

Now:

```kotlin
fun loadSavedMeasurements(context: Context) {
    viewModelScope.launch {
        uiState = uiState.copy(
            isLoading = true,
            message = "Loading saved measurements..."
        )

        try {
            val loadedMeasurements = measurementRepository.loadMeasurements(
                context = context
            )

            val latestValue = loadedMeasurements.lastOrNull()?.value

            uiState = uiState.copy(
                measurements = loadedMeasurements,
                latestValue = latestValue,
                isLoading = false,
                message = "Loaded ${loadedMeasurements.size} saved measurements"
            )
        } catch (e: Exception) {
            uiState = uiState.copy(
                isLoading = false,
                message = "Could not load saved measurements"
            )
        }
    }
}
```

Notice this line:

```kotlin
val latestValue = loadedMeasurements.lastOrNull()?.value
```

This means:

```text
If there is at least one saved measurement, use the last measurement value.
If there are no saved measurements, latestValue becomes null.
```

This reuses the null-safety idea from Lesson 3.

---

## 11. Updated ViewModel structure

Now the ViewModel has a cleaner role.

```kotlin
class ResearchViewModel(
    private val measurementRepository: MeasurementRepository = MeasurementRepository()
) : ViewModel() {

    var uiState by mutableStateOf(ResearchUiState())
        private set

    fun updateSampleId(newSampleId: String) {
        uiState = uiState.copy(
            sampleId = newSampleId
        )
    }

    fun connectDevice() {
        uiState = uiState.copy(
            deviceConnectionState = DeviceConnectionState.CONNECTING,
            message = "Connecting to device..."
        )

        viewModelScope.launch {
            delay(1000)

            uiState = uiState.copy(
                deviceConnectionState = DeviceConnectionState.CONNECTED,
                message = "Device connected"
            )
        }
    }

    fun disconnectDevice() {
        if (uiState.acquisitionState == AcquisitionState.RECORDING) {
            stopAcquisition()
        }

        uiState = uiState.copy(
            deviceConnectionState = DeviceConnectionState.DISCONNECTED,
            acquisitionState = AcquisitionState.IDLE,
            message = "Device disconnected"
        )
    }

    fun startAcquisition(context: Context) {
        if (uiState.deviceConnectionState != DeviceConnectionState.CONNECTED) {
            uiState = uiState.copy(
                message = "Connect device before starting acquisition"
            )
            return
        }

        if (uiState.acquisitionState == AcquisitionState.RECORDING) {
            return
        }

        uiState = uiState.copy(
            acquisitionState = AcquisitionState.RECORDING,
            message = "Acquisition started"
        )

        viewModelScope.launch {
            while (uiState.acquisitionState == AcquisitionState.RECORDING) {
                addSimulatedMeasurement(context)
                delay(1000)
            }
        }
    }

    fun stopAcquisition() {
        if (uiState.acquisitionState != AcquisitionState.RECORDING) {
            return
        }

        uiState = uiState.copy(
            acquisitionState = AcquisitionState.STOPPED,
            message = "Acquisition stopped"
        )
    }

    fun loadSavedMeasurements(context: Context) {
        viewModelScope.launch {
            uiState = uiState.copy(
                isLoading = true,
                message = "Loading saved measurements..."
            )

            try {
                val loadedMeasurements = measurementRepository.loadMeasurements(
                    context = context
                )

                val latestValue = loadedMeasurements.lastOrNull()?.value

                uiState = uiState.copy(
                    measurements = loadedMeasurements,
                    latestValue = latestValue,
                    isLoading = false,
                    message = "Loaded ${loadedMeasurements.size} saved measurements"
                )
            } catch (e: Exception) {
                uiState = uiState.copy(
                    isLoading = false,
                    message = "Could not load saved measurements"
                )
            }
        }
    }

    private fun addSimulatedMeasurement(context: Context) {
        val newMeasurement = measurementRepository.createSimulatedMeasurement(
            sampleId = uiState.sampleId,
            repetition = uiState.measurements.size + 1
        )

        val updatedMeasurements = uiState.measurements + newMeasurement

        uiState = uiState.copy(
            measurements = updatedMeasurements,
            latestValue = newMeasurement.value
        )

        saveMeasurementsAsync(
            context = context,
            measurements = updatedMeasurements
        )
    }

    private fun saveMeasurementsAsync(
        context: Context,
        measurements: List<Measurement>
    ) {
        viewModelScope.launch {
            try {
                measurementRepository.saveMeasurements(
                    context = context,
                    measurements = measurements
                )

                uiState = uiState.copy(
                    message = "Measurements auto-saved"
                )
            } catch (e: Exception) {
                uiState = uiState.copy(
                    message = "Auto-save failed"
                )
            }
        }
    }
}
```

This is still not perfect architecture, but it is much better than before.

The ViewModel now mainly handles:

```text
screen state
button actions
acquisition flow
messages
```

The repository handles:

```text
measurement creation
saving
loading
```

---

## 12. Current architecture after Lesson 13

Before Lesson 13:

```text
ResearchScreen
 ↓
ResearchViewModel
 ├── UI state
 ├── acquisition logic
 ├── fake data generation
 ├── file saving
 └── file loading
```

After Lesson 13:

```text
ResearchScreen
 ↓
ResearchViewModel
 ├── UI state
 ├── acquisition logic
 └── calls repository
      ↓
      MeasurementRepository
      ├── create simulated measurement
      ├── save measurements
      └── load measurements
```

This structure is closer to a real Android architecture.

Android’s current UI-layer guidance says the UI layer depends on state holders such as ViewModels, and those state holders can depend on classes from the data layer or optional domain layer. citeturn141600search0 That is exactly the direction we are moving in:

```text
UI
 ↓
ViewModel
 ↓
Repository / data layer
```

---

## 13. Why this prepares us for real device data

Right now, the repository creates fake data:

```kotlin
val value = Random.nextDouble(0.0, 5.0)
```

Later, the repository may call a real data source:

```kotlin
val value = bluetoothDataSource.readValue()
```

or:

```kotlin
val value = wifiDataSource.readValue()
```

or:

```kotlin
val value = sensorParser.parse(rawMessage)
```

The ViewModel does not need to care too much.

It can still call:

```kotlin
val newMeasurement = measurementRepository.createSimulatedMeasurement(
    sampleId = uiState.sampleId,
    repetition = uiState.measurements.size + 1
)
```

Later, we may rename this to something more general:

```kotlin
val newMeasurement = measurementRepository.getNextMeasurement(
    sampleId = uiState.sampleId,
    repetition = uiState.measurements.size + 1
)
```

Then the repository decides where the measurement comes from.

That is the advantage.

---

## 14. Repository versus ViewModel

A beginner-friendly difference:

```text
ViewModel asks:
What should happen now?

Repository answers:
Here is the data.
I saved the data.
I loaded the data.
```

More specifically:

| Part | Main responsibility |
|---|---|
| `ResearchScreen` | Show UI and send user actions |
| `ResearchViewModel` | Manage UI state and app flow |
| `MeasurementRepository` | Create, load, and save measurement data |
| Storage functions | Actually read/write files |
| Real device code later | Receive data from hardware |

So we should avoid putting everything into one giant ViewModel.

## 15. Do We Need a Repository Interface Now?

You may see code like this in more advanced apps:

```kotlin
interface MeasurementRepository {
    suspend fun saveMeasurements(measurements: List<Measurement>)
    suspend fun loadMeasurements(): List<Measurement>
}
```

and then:

```kotlin
class FileMeasurementRepository : MeasurementRepository {
    ...
}
```
This is useful, but for now it may add too much complexity.
For this tutorial, we can start with a simple class:

```kotlin
class MeasurementRepository {
    ...
}
```

Later, when we introduce:

- fake data source
- real Bluetooth data source
- Room database
- testing

then an interface may become useful.
So for now:

- Use a simple repository class first.
- Understand the purpose before adding more abstraction.

## 16. Important Note About Context

In the current beginner version, we pass context into repository functions:

```kotlin
measurementRepository.saveMeasurements(
    context = context,
    measurements = measurements
)
```
This is acceptable for learning because it keeps the code simple.
However, we should avoid storing an Activity context permanently inside a ViewModel or repository. Android’s ViewModel guidance warns that ViewModels live longer than UI objects, so holding lifecycle-related APIs in a ViewModel can cause memory leaks. Android Developers
So in this tutorial, we pass context only when needed.
Later, a more professional version may use:

- application context
- dependency injection
- repository created at app level
- Room database instance

But not yet.

## 17. What You Learned in Lesson 13

The key idea is:

> Do not let the ViewModel do everything.

You created:

```kotlin
class MeasurementRepository {
    ...
}
```

The repository now handles:

- creating simulated measurements
- saving measurements
- loading measurements

The ViewModel now calls:

```kotlin
measurementRepository.createSimulatedMeasurement(...)
measurementRepository.saveMeasurements(...)
measurementRepository.loadMeasurements(...)
```
The new architecture is:

```text
ResearchScreen
 ↓
ResearchViewModel
 ↓
MeasurementRepository
 ↓
Storage / fake data / real data later
```

The most important mental model is:

- UI shows state.
- ViewModel manages screen behaviour.
- Repository handles data.

For a research app, this is important because the app will later become more complex:

- patients
- sessions
- measurements
- Room database
- Bluetooth/Wi-Fi input
- signal processing
- ML inference
- CSV/JSON export

A repository gives us a place to manage that complexity without making the UI or ViewModel messy.

## Lesson 14 Preview

In Lesson 14, we should introduce:

**Room database**

So far, we used simple internal file storage and CSV-style saving.

That is useful for learning, but a serious research app may need structured data such as:

- Patient
- Session
- Measurement
- Result

Another drawback of the Lesson 13 file-based approach is that saving one new measurement means saving the whole list again.

In Lesson 13, after creating a measurement, we saved the whole list:

```text
create new measurement
 ↓
append to list
 ↓
save entire list to file
```

This is undesired for large dataset. 

With Room, we can insert only the new measurement.

Previously:

```text
create new measurement
 ↓
append to list
 ↓
save entire list to file
```

Room will help us store this kind of structured data locally on the Android tablet.
In Lesson 14, we will cover:

- why Room is useful
- Entity
- DAO
- Database
- insert measurement
- read measurements
- how Room differs from CSV export

That will move the app from simple file-based storage toward a more realistic research-app data layer.
