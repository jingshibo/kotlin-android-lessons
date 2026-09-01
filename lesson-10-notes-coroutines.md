# Lesson 10 Notes - Coroutines, Threads, and Dispatchers

This note explains how to think about the coroutine pattern used in Lesson 10.

The main pattern is:

```kotlin
viewModelScope.launch {
    val result = withContext(Dispatchers.IO) {
        doSlowIoWork()
    }

    uiState = uiState.copy(
        result = result
    )
}
```

The note is organized around one question:

```text
How do we run slow work without freezing the UI?
```

Read the note in this order:

```text
First: understand the UI-freezing problem.
Then: learn the main launch + withContext pattern.
Then: understand what waits, what returns, and what continues.
Then: name the pieces: thread, coroutine, dispatcher, suspend.
Finally: decide where helper functions, IO work, and try/catch should live.
```

## 1. The Core Problem

Android has a main/UI thread.

The main thread handles:

```text
drawing UI
responding to taps
responding to text input
scrolling
quick UI callbacks
```

If slow work runs on the main thread, the app can freeze.

Slow work includes:

```text
file reading
file writing
database work
network requests
Bluetooth / USB / device communication
signal processing
ML inference
long loops
```

So the goal of Lesson 10 is:

```text
Keep quick UI state updates on Main.
Move slow IO work away from Main.
Return the result to Main.
Update uiState after the work finishes.
```

## 2. The Main Pattern in Lesson 10

Start with the actual code shape:

```kotlin
viewModelScope.launch {
    uiState = uiState.copy(
        message = "Saving..."
    )

    withContext(Dispatchers.IO) {
        saveMeasurementsToInternalStorage(
            context = context,
            measurements = measurements
        )
    }

    uiState = uiState.copy(
        message = "Saved"
    )
}
```

Read it as:

```text
Main:
    start ViewModel coroutine
    update uiState quickly

IO:
    do slow file-writing work

Main:
    update uiState after saving
```

The important pieces are:

```text
viewModelScope.launch { ... }
    -> start coroutine work tied to the ViewModel

withContext(Dispatchers.IO) { ... }
    -> run this block on the IO dispatcher
```

This is the core pattern.

Everything else in this note explains why this pattern works.

## 3. What Runs Now, What Waits, and What Returns

Inside one coroutine, code still runs in order.

Example:

```kotlin
viewModelScope.launch {
    println("Before IO")

    withContext(Dispatchers.IO) {
        saveToFile()
    }

    println("After IO")
}
```

Inside the coroutine, the code runs in order:

```text
Before IO (runs in main thread)
saveToFile: (runs in IO thread)
After IO: (runs in main thread)
```

So code after `withContext(...)` waits until the IO block finishes.

But code after `launch { ... }` outside the coroutine does not wait:

```kotlin
fun saveSomething() {
    viewModelScope.launch {
        withContext(Dispatchers.IO) {
            saveToFile()
        }

        uiState = uiState.copy(message = "Saved")
    }

    println("After launch")
}
```

This means:

```text
viewModelScope.launch starts the coroutine.
println("After launch") runs almost immediately.
The coroutine continues separately.
```

`withContext(...)` can also return a value.

This is the preferred style when you need the result later:

```kotlin
val loadedMeasurements = withContext(Dispatchers.IO) {
    loadMeasurementsFromInternalStorage(context)
}

uiState = uiState.copy(
    measurements = loadedMeasurements
)
```

`withContext` returns the last expression inside its block.

Do not write this if you need the value after the block:

```kotlin
withContext(Dispatchers.IO) {
    val loadedMeasurements = loadMeasurementsFromInternalStorage(context)
}

uiState = uiState.copy(
    measurements = loadedMeasurements // Error
)
```

In that version, `loadedMeasurements` only exists inside the `{ ... }` block.

Beginner rule:

```text
Put val outside withContext when you need the result after the block.
Put val inside withContext only for temporary values used inside that block.
```

## 4. Thread, Coroutine, Dispatcher

Now we can name the pieces.

Sections 2 and 3 used `launch` and `withContext` first because those are the code shapes you actually write.

This section explains the system behind that code.

These words are related, but they are not the same.

| Word | Meaning |
|---|---|
| Thread | an actual execution path provided by the system/JVM |
| Coroutine | a Kotlin task that can pause and resume |
| Dispatcher | decides which thread or thread pool runs coroutine code |

Short version:

```text
Coroutine = the task
Thread = where the task runs right now
Dispatcher = the scheduler that chooses the thread
```

In Android:

```text
Main dispatcher
    -> usually the Android main/UI thread

Dispatchers.IO
    -> a shared pool of background threads for IO work
```

Important points:

```text
Dispatchers.IO is not one thread.
A coroutine is not a thread.
A coroutine does not create a new thread by itself.
Many coroutines can share a smaller number of threads.
```

The dispatcher assigns a coroutine a thread for it to run a task.

When the coroutine suspends or finishes, that thread is released and can be used for other work.

The same coroutine can move:

```text
Main -> IO -> Main
```

Example:

```kotlin
viewModelScope.launch {
    // current thread: usually Main

    withContext(Dispatchers.IO) {
        // current thread: one IO thread
    }

    // current thread: usually Main again
}
```

Current thread means:

```text
the thread where the coroutine is running at this exact line
```

## 5. Why `suspend` Appears

In Lesson 10, `suspend` appears because we start using coroutine-only functions.

This section comes after dispatcher switching because `suspend` is easier to understand after you have seen where it is needed.

The main one is:

```kotlin
withContext(Dispatchers.IO) {
    ...
}
```

`withContext(...)` is a suspend function.

That means it can be called from:

```text
inside a coroutine
or inside another suspend function
```

This works because `launch { ... }` creates coroutine code:

```kotlin
viewModelScope.launch {
    withContext(Dispatchers.IO) {
        saveMeasurementsToInternalStorage(context, measurements)
    }
}
```

But if we move the IO block into a helper function, that helper must be marked `suspend`:

```kotlin
private suspend fun saveMeasurementsInBackground(
    context: Context,
    measurements: List<Measurement>
) {
    withContext(Dispatchers.IO) {
        saveMeasurementsToInternalStorage(context, measurements)
    }
}
```

Why?

Because the helper directly calls:

```kotlin
withContext(Dispatchers.IO)
```

Basic meaning:

```text
suspend fun = a function that can contain coroutine suspension points
```

Important:

```text
suspend does not mean "run on IO".
suspend does not mean "create a new thread".
suspend does not mean "automatically fast".
suspend does not mean "always non-blocking".
```

It means:

```text
this function belongs to coroutine-aware code
```

Where can you call a suspend function?

Allowed inside a coroutine:

```kotlin
viewModelScope.launch {
    saveMeasurementsInBackground(context, measurements)
}
```

Allowed inside another suspend function:

```kotlin
suspend fun saveAndReload(context: Context) {
    saveMeasurementsInBackground(context, measurements)
    loadMeasurementsInBackground(context)
}
```

Usually not allowed inside a normal function:

```kotlin
fun normalFunction(context: Context) {
    saveMeasurementsInBackground(context, measurements) // Error
}
```

When do you need `suspend fun`?

```text
If the function contains withContext(...), delay(...), or another suspend call,
mark that function suspend.

If the function is only called inside withContext(...),
it does not need to be suspend.

If the function creates a coroutine with launch { ... },
and the suspend calls are inside that coroutine block,
the outer function does not need to be suspend.
```

So this does not need `suspend`:

```kotlin
fun saveMeasurementsToInternalStorage(
    context: Context,
    measurements: List<Measurement>
) {
    // normal blocking file-writing code
}
```

It may be slow, but slow does not automatically mean suspend.

This also does not need `suspend`:

```kotlin
fun saveSomething() {
    viewModelScope.launch {
        withContext(Dispatchers.IO) {
            saveToFile()
        }
    }
}
```

Why?

Because `saveSomething()` itself is not directly pausing.

It starts a coroutine, and the coroutine block contains the suspend call.

Compare:

```kotlin
suspend fun saveSomethingInCurrentCoroutine() {
    withContext(Dispatchers.IO) {
        saveToFile()
    }
}
```

This one does need `suspend` because the function itself directly calls `withContext(...)`.

## 6. Waiting Versus Blocking

Several things look like "waiting":

```text
delay
Thread.sleep
file reading/writing
```

But they are different.

**This section answers the confusing question:**

```text
If a coroutine waits, does the thread also wait?
```

| Code | What waits? | Is the thread occupied? | Common use |
|---|---|---|---|
| `delay(1000)` | coroutine waits | no | coroutine timer |
| `Thread.sleep(1000)` | thread waits | yes | avoid on Main |
| `readText()` / `write(...)` | file call waits | yes | run on IO |

This is a suspend function:

```kotlin
delay(1000)
```

Because `delay(...)` is a suspend function, it must be called from coroutine-aware code.

Allowed:

```kotlin
viewModelScope.launch {
    delay(1000)
}
```

Also allowed:

```kotlin
suspend fun waitBeforeSaving() {
    delay(1000)
}
```

Not allowed:

```kotlin
fun normalButtonFunction() {
    delay(1000) // Error
}
```

So be careful with this sentence:

```text
delay can run in a coroutine that is using Main.
delay cannot be called directly from normal Main-thread code.
```

It means:

```text
pause the coroutine for 1000 ms
let the current thread do other work
resume this coroutine later
```

If the coroutine is on Main:

```kotlin
viewModelScope.launch {
    delay(1000)
}
```

then `delay(1000)` is called while the coroutine is on Main. 

But Main is not busy for one second.

Kotlin sets a timer, pauses the coroutine, and gives Main back to Android.

This is different:

```kotlin
Thread.sleep(1000)
```

It means:

```text
make the current thread sit still for 1000 ms
```

If `Thread.sleep(1000)` runs on Main, the UI freezes.

Normal file APIs are also usually blocking:

```kotlin
context.openFileOutput(
    AUTOSAVE_FILENAME,
    Context.MODE_PRIVATE
).use { outputStream ->
    outputStream.write(csvText.toByteArray())
}
```

and:

```kotlin
val csvText = context
    .openFileInput(AUTOSAVE_FILENAME)
    .bufferedReader()
    .use { reader ->
        reader.readText()
    }
```

These operations usually occupy the current thread until the file work finishes.

So:

```text
delay on Main:
    coroutine is suspended
    Main is free

file write on Main:
    Main is occupied
    UI may freeze

file write on IO:
    one IO thread is occupied
    Main is free
```

This is the key distinction:

```text
A coroutine can suspend without blocking a thread.
But blocking code inside a coroutine still blocks whichever thread it runs on.
```

That is why Lesson 10 uses `Dispatchers.IO` for file work.

It does not make normal file writing magically non-blocking.

It moves blocking work away from Main.

## 7. When Do We Use a Coroutine Without Dispatchers.IO?

It is not always necessary to use coroutine together with dispather IO. Use a coroutine without `Dispatchers.IO` when the work is not blocking the Main thread.

For example:

```kotlin
viewModelScope.launch {
    while (uiState.isAcquiring) {
        addSimulatedMeasurement(context)
        delay(1000)
    }
}
```

This is okay because the work is mostly:

```text
create a small object
update uiState
wait using delay(...)
```

`delay(...)` does not block the Main thread.

It suspends the coroutine for 1s and lets Main continue handling the UI.

When the delay time is up, the coroutine resumes execution from where it was suspended, which is somewhat similar to a timer interrupt in an MCU. 

So this coroutine can start from the normal `viewModelScope.launch { ... }`.

We do not need this:

```kotlin
viewModelScope.launch(Dispatchers.IO) {
    while (uiState.isAcquiring) {
        addSimulatedMeasurement(context)
        delay(1000)
    }
}
```

Beginner rule:

```text
Use a coroutine by itself when the code waits by suspension, such as delay.
Use Dispatchers.IO when the code blocks a thread, such as file writing.
```

## 8. Main Work Versus IO Work

Do not put everything inside `Dispatchers.IO`.

Now that we have separated coroutine waiting from thread blocking, we can decide which code belongs on Main and which code belongs on IO.

IO is for slow input/output work:

```text
file reading/writing
database access
network requests
Bluetooth / USB / device communication
```

Main is the right place for quick UI state updates:

```kotlin
uiState = uiState.copy(message = "Saving...")
```

Preferred pattern: 

```kotlin
viewModelScope.launch {
    // Main: quick UI state update
    uiState = uiState.copy(message = "Saving...")

    // IO: slow file work
    val savedSuccessfully = withContext(Dispatchers.IO) {
        saveMeasurementsToInternalStorage(context, measurements)
        true
    }

    // Main: quick UI state update
    uiState = uiState.copy(
        message = if (savedSuccessfully) {
            "Saved"
        } else {
            "Save failed"
        }
    )
}
```

**Avoid this beginner pattern: Do not update/modify shared state variables in IO to avoid race condition.**

```kotlin
viewModelScope.launch {
    withContext(Dispatchers.IO) {
        uiState = uiState.copy(message = "Saving...") // shared state value
        saveMeasurementsToInternalStorage(context, measurements)
        uiState = uiState.copy(message = "Saved") // shared state value
    }
}
```

The problem is:

```text
UI-related state is updated from IO, which may also be modified in Main at the same time.
Shared ViewModel state can become harder to reason about.
```

Beginner rule:

```text
Use Main for quick uiState updates.
Use IO for slow file/database/network/device work.
Let IO return data or status.
Update uiState after returning to Main.
```

Could race conditions happen?

Yes.

For example:

```text
Coroutine A loads saved measurements.
Coroutine B adds a new measurement.
Both update uiState around the same time.
```

The later update may overwrite the earlier one if the state changes are not designed carefully.

For beginner lessons, keep the flow simple.

In larger apps, safer patterns include:

```text
single source of truth
repository layer
database transactions
Mutex
StateFlow
structured state updates
```

## 9. How to Organize Helper Functions

There are three related design questions.

After the mental model is clear, the remaining question is code organization:

```text
Who starts the coroutine?
Who chooses the dispatcher?
Who catches errors?
```

### Question A: Who chooses IO?

Style A: caller chooses IO.

```kotlin
fun saveMeasurementsToInternalStorage(
    context: Context,
    measurements: List<Measurement>
) {
    // normal blocking file-writing code
}

viewModelScope.launch {
    withContext(Dispatchers.IO) {
        saveMeasurementsToInternalStorage(context, measurements)
    }
}
```

Here:

```text
saveMeasurementsToInternalStorage is a normal function.
The caller decides to run it on Dispatchers.IO.
```

Style B: helper chooses IO.

```kotlin
suspend fun saveMeasurementsInBackground(
    context: Context,
    measurements: List<Measurement>
) {
    withContext(Dispatchers.IO) {
        saveMeasurementsToInternalStorage(context, measurements)
    }
}

viewModelScope.launch {
    saveMeasurementsInBackground(context, measurements)
}
```

Here:

```text
saveMeasurementsInBackground is a suspend function.
It hides the Dispatchers.IO detail inside itself.
```

Lesson 10 starts with Style A because it makes the dispatcher visible.

Then it moves toward Style B because the ViewModel code becomes easier to read.

### Question B: Who calls viewModelScope.launch?

Style A: helper starts its own coroutine.

```kotlin
private fun saveMeasurementsAsync(
    context: Context,
    measurements: List<Measurement>
) {
    viewModelScope.launch {
        withContext(Dispatchers.IO) {
            saveMeasurementsToInternalStorage(context, measurements)
        }
    }
}
```

Then the caller simply says:

```kotlin
saveMeasurementsAsync(context, updatedMeasurements)
```

This can be fine for fire-and-forget autosave.

Style B: caller starts the coroutine, helper is suspend.

```kotlin
private suspend fun saveMeasurementsInBackground(
    context: Context,
    measurements: List<Measurement>
) {
    withContext(Dispatchers.IO) {
        saveMeasurementsToInternalStorage(context, measurements)
    }
}
```

Caller:

```kotlin
viewModelScope.launch {
    saveMeasurementsInBackground(context, updatedMeasurements)

    uiState = uiState.copy(
        message = "Measurements auto-saved"
    )
}
```

This is often cleaner when the caller needs to control:

```text
order
loading state
error handling
result handling
what happens after the work finishes
```

Beginner rule:

```text
UI should not launch ViewModel coroutines.
ViewModel should launch them.

Inside the ViewModel:
Use a normal private function with launch for simple fire-and-forget work.
Use a private suspend helper when the caller should control the flow.
```

### Question C: Where should try/catch go?

If the error handling updates `uiState`, catch in the caller:

```kotlin
viewModelScope.launch {
    try {
        saveMeasurementsInBackground(context, updatedMeasurements)

        uiState = uiState.copy(message = "Saved")
    } catch (e: Exception) {
        uiState = uiState.copy(message = "Save failed")
    }
}
```

This keeps `uiState` updates outside `Dispatchers.IO`.

If a reusable helper catches errors, it should return a result and avoid touching `uiState`:

```kotlin
private suspend fun saveMeasurementsInBackground(
    context: Context,
    measurements: List<Measurement>
): Boolean {
    return try {
        withContext(Dispatchers.IO) {
            saveMeasurementsToInternalStorage(context, measurements)
        }

        true
    } catch (e: Exception) {
        false
    }
}
```

Then the caller updates `uiState`:

```kotlin
viewModelScope.launch {
    val savedSuccessfully = saveMeasurementsInBackground(
        context = context,
        measurements = updatedMeasurements
    )

    uiState = if (savedSuccessfully) {
        uiState.copy(message = "Measurements auto-saved")
    } else {
        uiState.copy(message = "Auto-save failed")
    }
}
```

Beginner rule:

```text
Do not update uiState inside Dispatchers.IO.

Put try/catch inside a reusable helper if it only returns a success/failure result.

Put try/catch in the caller if the catch needs to update uiState directly,
or if different callers need different reactions.
```

## 10. Common Misunderstandings

| Misunderstanding | Better understanding |
|---|---|
| Coroutine = thread | A coroutine is a task that can run on a thread |
| Dispatchers.IO = one IO thread | Dispatchers.IO is backed by a pool of IO threads |
| suspend means run on background thread | suspend means coroutine-aware pause/resume |
| any code inside a coroutine cannot block | blocking code inside a coroutine still blocks its thread |
| withContext runs at the same time as code after it | inside one coroutine, code after withContext waits |
| file IO in a coroutine is automatically non-blocking | normal file IO still blocks an IO thread |

## 11. Checklist and Summary

When writing coroutine code in a ViewModel, ask:

```text
Is this quick UI state work?
    -> keep it on Main

Is this slow file/database/network/device work?
    -> use Dispatchers.IO

Do I need the result after withContext?
    -> val result = withContext(...) { ... }

Does this helper directly call withContext, delay, or another suspend function?
    -> mark the helper suspend

Does the helper update uiState?
    -> avoid doing that inside IO

Does the caller need to control order, errors, or result handling?
    -> caller should launch and call a suspend helper
```

Final pattern to remember:

```kotlin
viewModelScope.launch {
    uiState = uiState.copy(
        message = "Working..."
    )

    val result = withContext(Dispatchers.IO) {
        doSlowIoWork()
    }

    uiState = uiState.copy(
        message = "Done",
        result = result
    )
}
```

Read it as:

```text
Main:
    update UI state before work

IO:
    do slow work and return result

Main:
    update UI state after work
```

Most important rule:

```text
Keep slow blocking work off Main.
Use IO for file/database/network/device work.
Return results to Main and update uiState there.
```
