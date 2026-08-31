# Lesson 10 — Coroutines and Background Work

In Lesson 9, our app became more useful because it could:

```text
App starts
 ↓
Load previous measurements

User adds a measurement
 ↓
Measurement is added to the app state
 ↓
Measurements are automatically saved
```

That is good.

But there is one important problem:

```text
Saving/loading files directly on the UI path can block the app.
```

For a tiny CSV file, you may not notice the problem. But for a real research app, you may later have:

```text
many measurements
large files
Bluetooth or Wi-Fi communication
signal processing
ML inference
long-running data acquisition
```

If these tasks run directly on the main UI thread, the app may feel frozen.

So in this lesson, we introduce:

```text
coroutines
viewModelScope
Dispatchers.IO
suspend functions
```

Do not worry if these words look abstract. The practical meaning is simple:

```text
Do slow work in the background.
Keep the UI responsive.
```

Android’s documentation describes Kotlin coroutines as a way to write asynchronous code, and `viewModelScope` is a coroutine scope tied to a `ViewModel`. Coroutines launched in `viewModelScope` are automatically cancelled when the ViewModel is cleared. citeturn276776search0

---

## 1. The main/UI thread

An Android app has a main thread.

You can think of it as the thread responsible for:

```text
drawing the UI
responding to button clicks
updating text
handling user interaction
```

For example:

```kotlin
Button(
    onClick = {
        viewModel.addMeasurement(context)
    }
) {
    Text("Add Measurement")
}
```

When the user taps the button, Android expects the app to respond quickly.

If `addMeasurement()` does something slow, such as:

```text
save a large file
read from storage
wait for a device
run signal processing
run ML inference
```

then the UI may pause.

In a research tablet app, this is bad because the user may think:

```text
Did the button work?
Is the app frozen?
Was the measurement saved?
Should I tap again?
```

So the rule is:

```text
The UI should trigger work.
The slow work should happen in the background.
The result should update the UI state.
```

---

## 2. What is a coroutine?

A coroutine is a way to run work asynchronously.

A simple mental model:

```text
normal function
 ↓
runs immediately on the current path

coroutine
 ↓
can run work without blocking the UI
```

This is not exactly the same as a thread, but while learning, you can think of it as:

```text
a lightweight way to do background work
```

For example, instead of this:

```kotlin
fun addMeasurement(context: Context) {
    // Add measurement
    // Save file directly
}
```

we will move toward this:

```kotlin
fun addMeasurement(context: Context) {
    viewModelScope.launch {
        // Add measurement
        // Save file in background
    }
}
```

The important new part is:

```kotlin
viewModelScope.launch {
    ...
}
```

This means:

```text
Start a coroutine connected to this ViewModel.
```

Android’s current coroutine guidance shows `viewModelScope.launch(...)` as the common way to start coroutine work from a ViewModel, and it also explains that `withContext(Dispatchers.IO)` can move blocking I/O work away from the main thread. citeturn276776search17

---

## 3. The basic pattern

The pattern we need is:

```kotlin
viewModelScope.launch {
    // work starts here
}
```

For file saving, we usually want:

```kotlin
viewModelScope.launch {
    // Start a coroutine tied to this ViewModel.
    // If the ViewModel is cleared, this coroutine is cancelled automatically.
    withContext(Dispatchers.IO) {
        // Switch this block to the IO dispatcher.
        // Put slow file/database/network/device work here.
        // file read/write work here
    }
}
```

Read this as:

```text
viewModelScope.launch
 ↓
start background-capable work from the ViewModel

withContext(Dispatchers.IO)
 ↓
run file/network/database-style work on an IO dispatcher
```

`Dispatchers.IO` is intended for input/output work such as file operations, database access, or network communication.

You can treat this as a common Android ViewModel pattern:

```kotlin
viewModelScope.launch {
    withContext(Dispatchers.IO) {
        // slow IO work
    }
}
```

The outer block:

```kotlin
viewModelScope.launch {
    ...
}
```

starts coroutine work that belongs to the ViewModel.

The inner block:

```kotlin
withContext(Dispatchers.IO) {
    ...
}
```

temporarily switches to an IO thread pool for slow input/output work.

Important: `withContext(Dispatchers.IO)` is not complete by itself. It needs a block:

```kotlin
withContext(Dispatchers.IO) {
    // work here
}
```

After the `withContext` block finishes, the coroutine continues after it.

---

## 4. New imports

In your `MainActivity.kt` or wherever your `ResearchViewModel` is defined, you need these imports:

```kotlin
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
```

You probably already have:

```kotlin
import androidx.lifecycle.ViewModel
```

So together, the ViewModel-related imports may look like:

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
```

---

## 5. The problem in the Lesson 9 style

In Lesson 9, the logic was conceptually like this:

```kotlin
fun addMeasurement(context: Context) {
    val newMeasurement = Measurement(
        sampleId = uiState.sampleId,
        repetition = uiState.measurements.size + 1,
        value = Random.nextDouble(0.0, 5.0),
        timestamp = System.currentTimeMillis(),
        status = "OK"
    )

    val updatedMeasurements = uiState.measurements + newMeasurement

    uiState = uiState.copy(
        measurements = updatedMeasurements
    )

    saveMeasurementsToInternalStorage(
        context = context,
        measurements = updatedMeasurements
    )
}
```

This works for learning.

But this line may become slow:

```kotlin
saveMeasurementsToInternalStorage(
    context = context,
    measurements = updatedMeasurements
)
```

So we want:

```text
Update UI state quickly.
Save data in the background.
```

---

## 6. First coroutine version

We can change `addMeasurement()` to this:

```kotlin
fun addMeasurement(context: Context) {
    val newMeasurement = Measurement(
        sampleId = uiState.sampleId,
        repetition = uiState.measurements.size + 1,
        value = Random.nextDouble(0.0, 5.0),
        timestamp = System.currentTimeMillis(),
        status = "OK"
    )

    val updatedMeasurements = uiState.measurements + newMeasurement

    uiState = uiState.copy(
        measurements = updatedMeasurements
    )

    viewModelScope.launch {
        withContext(Dispatchers.IO) {
            saveMeasurementsToInternalStorage(
                context = context,
                measurements = updatedMeasurements
            )
        }
    }
}
```

The important change is:

```kotlin
viewModelScope.launch {
    withContext(Dispatchers.IO) {
        saveMeasurementsToInternalStorage(...)
    }
}
```

Now the UI state updates immediately:

```kotlin
uiState = uiState.copy(
    measurements = updatedMeasurements
)
```

Then the file saving happens through a coroutine.

For the user, the app feels more responsive.

This first version is useful because you can see the whole coroutine pattern directly inside `addMeasurement()`.

But the function is starting to mix two ideas:

```text
addMeasurement()
    -> create and add a measurement
    -> know exactly how background saving works
```

As the app grows, we often want to move the background-saving detail into a helper function.

That is why the next concept appears:

```kotlin
suspend fun
```

---

## 7. What does `suspend` mean?

The first coroutine version works, but this part is still written directly inside `addMeasurement()`:

```kotlin
viewModelScope.launch {
    withContext(Dispatchers.IO) {
        saveMeasurementsToInternalStorage(...)
    }
}
```

To make `addMeasurement()` easier to read, we can move the background-saving detail into a helper function.

That helper will contain:

```kotlin
withContext(Dispatchers.IO) {
    ...
}
```

`withContext(...)` is a suspend function.

So a function that directly calls `withContext(...)` must also be marked with `suspend`:

```kotlin
suspend fun saveMeasurementsInBackground(...)
```

Basic meaning:

```text
suspend fun = a function that can be called from coroutine code and may pause/resume inside that coroutine
```

For this lesson, remember:

```text
viewModelScope.launch { ... }
    -> starts coroutine work

withContext(Dispatchers.IO) { ... }
    -> runs one block on the IO dispatcher

suspend fun
    -> lets us put coroutine-only code such as withContext(...) inside a reusable function
```

A `suspend` function is usually called from inside a coroutine:

```kotlin
viewModelScope.launch {
    saveMeasurementsInBackground(...)
}
```

So section 8 will use this idea to create a cleaner save helper.

For the deeper mental model, see:

```text
lesson-10-notes-coroutines.md
```

---

## 8. Cleaner save function

Let us create this helper function inside the ViewModel:

```kotlin
// This helper contains withContext(...), so the helper itself must be suspend.
// The caller no longer needs to see the Dispatchers.IO detail.
private suspend fun saveMeasurementsInBackground(
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
```

Then `addMeasurement()` becomes:

```kotlin
fun addMeasurement(context: Context) {
    val newMeasurement = Measurement(
        sampleId = uiState.sampleId,
        repetition = uiState.measurements.size + 1,
        value = Random.nextDouble(0.0, 5.0),
        timestamp = System.currentTimeMillis(),
        status = "OK"
    )

    val updatedMeasurements = uiState.measurements + newMeasurement

    uiState = uiState.copy(
        measurements = updatedMeasurements
    )

    viewModelScope.launch {
        saveMeasurementsInBackground(
            context = context,
            measurements = updatedMeasurements
        )
    }
}
```

This is easier to read:

```text
add measurement
update UI state
launch background save
```

---

## 9. Background loading

In Lesson 9, we also loaded saved measurements when the app opened.

The old idea was:

```kotlin
fun loadSavedMeasurements(context: Context) {
    val loadedMeasurements = loadMeasurementsFromInternalStorage(context)

    uiState = uiState.copy(
        measurements = loadedMeasurements
    )
}
```

Again, this may be okay for a small file, but loading should also happen in the background.

So we can write:

```kotlin
fun loadSavedMeasurements(context: Context) {
    viewModelScope.launch {
        // Put val outside withContext so loadedMeasurements is still visible after the IO block.
        // withContext returns the last expression from its block.
        val loadedMeasurements = withContext(Dispatchers.IO) {
            loadMeasurementsFromInternalStorage(context)
        }

        uiState = uiState.copy(
            measurements = loadedMeasurements
        )
    }
}
```

Notice the sequence:

```text
viewModelScope.launch
 ↓
start coroutine

withContext(Dispatchers.IO)
 ↓
load file in background

uiState = uiState.copy(...)
 ↓
update UI after loading
```

This line is important:

```kotlin
val loadedMeasurements = withContext(Dispatchers.IO) {
    loadMeasurementsFromInternalStorage(context)
}
```

The variable is declared outside the `withContext` block, so we can use it after the block finishes.

Do not write this if you need the value later:

```kotlin
withContext(Dispatchers.IO) {
    val loadedMeasurements = loadMeasurementsFromInternalStorage(context)
}
```

In that version, `loadedMeasurements` only exists inside the `{ ... }` block.

This is a very common Android pattern.

---

## 10. Add loading status to the UI state

Now that loading is asynchronous, the UI may briefly be in a loading state.

So we can improve our `ResearchUiState`.

Previously, it may have looked like this:

```kotlin
data class ResearchUiState(
    val sampleId: String = "",
    val isConnected: Boolean = false,
    val measurements: List<Measurement> = emptyList()
)
```

Now we can add:

```kotlin
val isLoading: Boolean = false
```

and maybe:

```kotlin
val message: String = ""
```

So:

```kotlin
data class ResearchUiState(
    val sampleId: String = "",
    val isConnected: Boolean = false,
    val measurements: List<Measurement> = emptyList(),
    val isLoading: Boolean = false,
    val message: String = ""
)
```

This allows the ViewModel to tell the UI:

```text
I am loading previous measurements.
I have restored previous measurements.
I saved the latest measurement.
Something went wrong.
```

---

## 11. Improved loading function

Now we can write:

```kotlin
fun loadSavedMeasurements(context: Context) {
    viewModelScope.launch {
        uiState = uiState.copy(
            isLoading = true,
            message = "Loading saved measurements..."
        )

        val loadedMeasurements = withContext(Dispatchers.IO) {
            loadMeasurementsFromInternalStorage(context)
        }

        uiState = uiState.copy(
            measurements = loadedMeasurements,
            isLoading = false,
            message = "Loaded ${loadedMeasurements.size} saved measurements"
        )
    }
}
```

This is much better for the user.

The UI can show:

```text
Loading saved measurements...
```

Then:

```text
Loaded 12 saved measurements
```

---

## 12. Handling errors

File loading or saving can fail.

For example:

```text
file not found
file format problem
storage issue
unexpected app state
```

So we should use:

```kotlin
try {
    ...
} catch (e: Exception) {
    ...
}
```

Example:

```kotlin
fun loadSavedMeasurements(context: Context) {
    viewModelScope.launch {
        uiState = uiState.copy(
            isLoading = true,
            message = "Loading saved measurements..."
        )

        try {
            // Put val outside withContext so loadedMeasurements is visible after the IO block.
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

This is important for a research app.

You do not want the app to simply crash during data collection.

A safer pattern is:

```text
try the operation
if it works, update state normally
if it fails, show a clear message
```

---

## 13. Improved save with error handling

We can also improve saving:

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

Then `addMeasurement()` can call:

```kotlin
saveMeasurementsAsync(
    context = context,
    measurements = updatedMeasurements
)
```

Now `addMeasurement()` is easier to understand:

```kotlin
fun addMeasurement(context: Context) {
    val newMeasurement = Measurement(
        sampleId = uiState.sampleId,
        repetition = uiState.measurements.size + 1,
        value = Random.nextDouble(0.0, 5.0),
        timestamp = System.currentTimeMillis(),
        status = "OK"
    )

    val updatedMeasurements = uiState.measurements + newMeasurement

    uiState = uiState.copy(
        measurements = updatedMeasurements
    )

    saveMeasurementsAsync(
        context = context,
        measurements = updatedMeasurements
    )
}
```

The mental model is:

```text
addMeasurement()
 ↓
create new measurement
 ↓
update app state
 ↓
request background save
```

---

## 14. Showing loading/message in Compose

In your screen, you can show the message:

```kotlin
if (uiState.message.isNotEmpty()) {
    Text(uiState.message)
}
```

You can also show a loading message:

```kotlin
if (uiState.isLoading) {
    Text("Loading...")
}
```

For now, simple text is enough.

Later, you could use a progress indicator, but we do not need that yet.

A simple version inside your `ResearchScreen()` might look like:

```kotlin
if (uiState.isLoading) {
    Text("Loading saved data...")
}

if (uiState.message.isNotEmpty()) {
    Text(uiState.message)
}
```

This gives user feedback.

That is important in research apps because users need confidence that data is being saved.

---

## 15. Full ViewModel pattern for Lesson 10

Here is the important ViewModel pattern from this lesson.

This is not necessarily your whole app file, but it shows the main idea clearly:

```kotlin
class ResearchViewModel : ViewModel() {

    var uiState by mutableStateOf(ResearchUiState())
        private set

    fun updateSampleId(newSampleId: String) {
        uiState = uiState.copy(
            sampleId = newSampleId
        )
    }

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

    fun addMeasurement(context: Context) {
        val newMeasurement = Measurement(
            sampleId = uiState.sampleId,
            repetition = uiState.measurements.size + 1,
            value = Random.nextDouble(0.0, 5.0),
            timestamp = System.currentTimeMillis(),
            status = "OK"
        )

        val updatedMeasurements = uiState.measurements + newMeasurement

        uiState = uiState.copy(
            measurements = updatedMeasurements
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
}
```

The important thing is not to memorise every line.

The important thing is to understand the shape:

```text
ViewModel function
 ↓
update UI state
 ↓
launch coroutine
 ↓
do file work using Dispatchers.IO
 ↓
update UI state again
```

---

## 16. Why this matters for the future

This lesson is not only about saving files.

The same pattern will be used later for many research-app tasks.

### File saving

```kotlin
viewModelScope.launch {
    withContext(Dispatchers.IO) {
        saveMeasurements(...)
    }
}
```

### Loading previous sessions

```kotlin
viewModelScope.launch {
    val sessions = withContext(Dispatchers.IO) {
        loadSessions()
    }
}
```

### Bluetooth or Wi-Fi device communication

Conceptually:

```kotlin
viewModelScope.launch {
    withContext(Dispatchers.IO) {
        readFromDevice()
    }
}
```

### Signal processing

For heavier CPU work, we may later use a different dispatcher:

```kotlin
withContext(Dispatchers.Default) {
    processSignal()
}
```

### ML inference

Conceptually:

```kotlin
viewModelScope.launch {
    val result = withContext(Dispatchers.Default) {
        runModelInference(input)
    }

    uiState = uiState.copy(
        latestResult = result
    )
}
```

So Lesson 10 prepares us for:

```text
live acquisition
device communication
signal processing
ML inference
database operations
```

---

## 17. Important beginner warning

Do not use a normal blocking loop like this in your UI:

```kotlin
while (isMeasuring) {
    readMeasurement()
}
```

This can freeze the app if it runs on the main thread.

Instead, we will later use a coroutine-based loop:

```kotlin
viewModelScope.launch {
    while (isMeasuring) {
        val value = getMeasurement()
        delay(1000)
    }
}
```

We will cover this properly in Lesson 11.

For now, remember:

```text
Long-running acquisition should not directly block the UI.
```

---

## 18. What you learned in Lesson 10

The key concepts are:

```kotlin
viewModelScope.launch {
    ...
}
```

starts coroutine work from the ViewModel.

```kotlin
withContext(Dispatchers.IO) {
    ...
}
```

moves file/database/network-style work away from the main UI path.

```kotlin
suspend fun ...
```

marks a function that can pause inside a coroutine.

```kotlin
try {
    ...
} catch (e: Exception) {
    ...
}
```

helps prevent the app from crashing if saving/loading fails.

The most important mental model is:

```text
UI action
 ↓
ViewModel function
 ↓
quickly update UI state
 ↓
run slow work in background
 ↓
update UI again with result/message
```

For a research app, this is essential because later we will deal with:

- continuous sensor acquisition
- real device communication
- larger data files
- signal processing
- machine-learning inference

## Lesson 11 Preview

In Lesson 11, we should move from:

```text
one random value per button click
                ↓
a simulated live data stream
```

That means the app will be able to:

```text
Start Acquisition
 ↓
generate repeated fake measurements
 ↓
update the latest value
 ↓
append to measurement history
 ↓
Stop Acquisition
```

This will prepare us for replacing fake data with real Bluetooth, Wi-Fi, USB, or sensor data later.
