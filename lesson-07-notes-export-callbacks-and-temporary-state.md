# Lesson 7 Notes - Export Callbacks, Temporary State, and Recomposition

This note explains a subtle part of Lesson 7:

```text
Why does CSV export need this strange-looking pendingCsvText state?
Can we avoid it?
What happens while the Android file picker is open?
```

The important idea is that Android file export is not one simple line of code.

It is a small conversation between your app and Android:

```text
Your app:
    Please open a file picker.

Android:
    Okay. I will ask the user where to save the file.

Later...

Android:
    Here is the Uri the user chose.

Your app:
    Now I can write the CSV file.
```

Because the result comes back later, we have to think carefully about where temporary data should live.

This note is organized in this order:

1. Start from the first natural question: why not use a local `val` in `onClick`?
2. Understand the old `pendingCsvText` method.
3. Understand `var`, `remember`, and `mutableStateOf` in that old method.
4. Learn what a callback is.
5. Follow the exact timing of `onClick`, `launch(...)`, the file picker, and `onResult`.
6. Understand what may happen while the file picker is open.
7. Understand recomposition during callbacks, including batching.
8. Understand why picker blocking does not make local variables absolutely safe.
9. See the stale-data problem and extra-state burden in the old method.
10. Learn the cleaner `onResult` method.
11. Compare `pendingCsvText` with local `val csvText`.
12. Compare snapshot export with latest export.
13. See a better beginner version.
14. Understand small export versus large export.
15. Use the final checklist.

## 1. The First Natural Question

When you first see export code, it is natural to think:

```text
The user clicks Export.
So I can create the CSV text inside the button's onClick.
Why do I need a separate pendingCsvText state variable?
```

You may want to write:

```kotlin
Button(
    onClick = {
        val pendingCsvText = measurementsToCsv(measurements)
        createCsvLauncher.launch("measurements.csv")
    }
) {
    Text("Export CSV")
}
```

This feels reasonable.

Inside `onClick`, you create the CSV text.

Then you launch the file picker.

But this local `val` has one problem:

```text
onResult cannot see it later.
```

Why?

Because `pendingCsvText` is local to the `onClick` block.

It only exists inside this pair of braces:

```kotlin
onClick = {
    val pendingCsvText = measurementsToCsv(measurements)
    createCsvLauncher.launch("measurements.csv")
}
```

The file is not written inside that same block, but is written later, inside the launcher's `onResult` callback:

```kotlin
onResult = { uri: Uri? ->
    // file writing happens here
}
```

So this would not work:

```kotlin
val createCsvLauncher = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.CreateDocument("text/csv"),
    onResult = { uri: Uri? ->
        if (uri != null) {
            context.contentResolver.openOutputStream(uri)?.use { outputStream ->
                outputStream.write(pendingCsvText.toByteArray()) // Error
            }
        }
    }
)
```

The reason is both scope and time:

```text
Scope:
    pendingCsvText was declared inside onClick.
    onResult is a different block.

Time:
    onClick runs now.
    onResult runs later.
```

That is the first reason Lesson 7 used a variable outside `onClick`.

## 2. The Old `pendingCsvText` Method

The method in lesson-07 moves the variable outside the button (old method):

```kotlin
var pendingCsvText by remember {
    mutableStateOf("")
}
```

Then both pieces of export code can use it.

The button writes into it:

```kotlin
Button(
    onClick = {
        pendingCsvText = measurementsToCsv(measurements)
        createCsvLauncher.launch("measurements.csv")
    }
) {
    Text("Export CSV")
}
```

The result callback reads from it:

```kotlin
val createCsvLauncher = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.CreateDocument("text/csv"),
    onResult = { uri: Uri? ->
        if (uri != null) {
            context.contentResolver.openOutputStream(uri)?.use { outputStream ->
                outputStream.write(pendingCsvText.toByteArray())
            }
        }
    }
)
```

Read the old method as:

```text
onClick:
    create CSV text
    store it in pendingCsvText
    request the file picker

onResult:
    receive the Uri later
    read pendingCsvText
    write the file
```

This method works because `pendingCsvText` now lives in a place that both `onClick` and `onResult` can access.

So the old method is not nonsense.

It is trying to solve a real timing problem:

```text
The CSV text is created now.
The file location arrives later.
Something must carry the text across that gap.
```

## 3. Why `var`, `remember`, and `mutableStateOf`

The old method used:

```kotlin
var pendingCsvText by remember {
    mutableStateOf("")
}
```

Each word solves a different problem.

### Why `var`

`pendingCsvText` starts empty:

```kotlin
""
```

Later, when the user clicks Export, the code replaces it:

```kotlin
pendingCsvText = measurementsToCsv(measurements)
```

Because the variable is reassigned with `=`, it must be `var`.

This would not work:

```kotlin
val pendingCsvText = ""

pendingCsvText = measurementsToCsv(measurements) // Error
```

So:

```text
var is needed because the old method changes the value later.
```

### Why `remember`

Compose can recompose.

That means the composable function can run again to describe the latest UI.

If we wrote:

```kotlin
var pendingCsvText = ""
```

then `pendingCsvText` is just a normal local variable in one run of the composable.

If the composable runs again for some reasons, that line creates a fresh new variable again:

```kotlin
var pendingCsvText = ""
```

That is risky for the old method because the data is lost before `onResult` saves it, after the Android file picker returns.

For more details on how the file picker works and what happens to the main UI at the time, see section 5.

The old method needs the CSV text to survive this gap:

```text
button click
    -> CSV text is created
    -> file picker request is sent
    -> time passes
file picker result
    -> CSV text is needed
```

So it uses `remember`.

`remember` tells Compose:

```text
Keep this value across recompositions while this composable is remembered.
```

### Why `mutableStateOf`

`mutableStateOf` creates Compose-observable state.

That is very useful when the UI displays or reacts to the value.

For example:

```kotlin
var exportMessage by remember {
    mutableStateOf("")
}
```

`exportMessage` should be state because the UI displays it:

```kotlin
Text(exportMessage)
```

But `pendingCsvText` is different.

The UI usually does not display raw CSV text.

In the old method, `mutableStateOf` is mainly being used as a convenient remembered mutable holder:

```text
store CSV text now
read it later in onResult
```

The old method needs a remembered mutable value. However:

```text
It does not strongly need Compose-observable UI state unless the UI reads/display pendingCsvText, but this is uncommon.
```

That is why the old method is understandable but not ideal.

It uses screen state for data that is really just temporary file-writing data.

## 4. What Is a Callback?

We have already mentioned `onClick` and `onResult` several times.

They play a central role in the export function, and they are both examples of callbacks.

A callback is code that you give to something else, so that it can call your code later.

Simple idea:

```text
I cannot finish this work right now.
When the event happens later, call this code.
```

`onClick` is a callback.

You give code to a `Button`:

```kotlin
Button(
    onClick = {
        exportMessage = "Export clicked"
    }
) {
    Text("Export CSV")
}
```

The `Button` does not run that code when the UI is first drawn.

It runs that code later when the user clicks the button.

So:

```text
onClick:
    callback for a future click event
```

`onResult` is also a callback.

You give code to Android's activity-result system:

```kotlin
val createCsvLauncher = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.CreateDocument("text/csv"),
    onResult = { uri: Uri? ->
        // This runs later when Android returns the result.
    }
)
```

Android does not run that code when the launcher is created.

Android runs it later when the file picker finishes.

So:

```text
onResult:
    callback for a future file-picker result event
```

This is why callback code can be confusing:

```text
You write the code now.
But the code runs later.
```

That difference between where code is written and when code runs is the heart of this note.

## 5. The File Picker Timeline

Now we can describe the whole export process accurately by connecting the code to the timing.

Here is the old export pattern from the earlier sections:

```kotlin
var pendingCsvText by remember {
    mutableStateOf("")
}

val createCsvLauncher = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.CreateDocument("text/csv"),
    onResult = { uri: Uri? ->
        if (uri != null) {
            context.contentResolver.openOutputStream(uri)?.use { outputStream ->
                outputStream.write(pendingCsvText.toByteArray())
            }

            exportMessage = "CSV exported successfully."
        } else {
            exportMessage = "CSV export cancelled."
        }
    }
)

Button(
    onClick = {
        pendingCsvText = measurementsToCsv(measurements)

        val filename = if (sampleId.isNotBlank()) {
            "${safeFilename(sampleId)}_measurements.csv"
        } else {
            "measurements.csv"
        }

        createCsvLauncher.launch(filename)
    }
) {
    Text("Export CSV")
}
```

The confusing part is that the code is written in one place, but it runs in different moments.

### Part A: Launcher Setup

This part runs when Compose builds the screen:

```kotlin
val createCsvLauncher = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.CreateDocument("text/csv"),
    onResult = { uri: Uri? ->
        ...
    }
)
```

What is running?

```text
Your composable function is running.
Compose is creating/remembering the Activity Result launcher.
Android is storing the onResult callback for later.
```

What is not running yet?

```text
The file picker is not open.
onClick is not running.
onResult is not running.
No CSV text is created.
No file is written.
```

The `contract` describes the kind of request:

```kotlin
ActivityResultContracts.CreateDocument("text/csv")
```

Meaning:

```text
Later, when this launcher is launched,
ask Android to create a document with CSV file type.
```

The `onResult` block is callback code:

```kotlin
onResult = { uri: Uri? ->
    ...
}
```

It is like instructions stored for later.

The instructions exist now, but the body of the callback does not run now.

### Part B: Button Click

This part runs when the user taps the Export button:

```kotlin
onClick = {
    pendingCsvText = measurementsToCsv(measurements)

    val filename = if (sampleId.isNotBlank()) {
        "${safeFilename(sampleId)}_measurements.csv"
    } else {
        "measurements.csv"
    }

    createCsvLauncher.launch(filename)
}
```

Where is it running?

```text
In your app.
Usually on the Android Main thread.
```

What is running?

```text
The Button's onClick callback is running.
The app creates CSV text from the current measurements.
The app stores that text in pendingCsvText.
The app reads sampleId.
The app creates a suggested filename String.
The app sends a file-picker request to Android.
```

This line creates the CSV text now:

```kotlin
pendingCsvText = measurementsToCsv(measurements)
```

That means the old method creates a click-time snapshot.

Then this code creates a filename:

```kotlin
val filename = if (sampleId.isNotBlank()) {
    "${safeFilename(sampleId)}_measurements.csv"
} else {
    "measurements.csv"
}
```

At this moment:

```text
No Uri exists yet.
No output stream exists yet.
CSV text has been created and stored in pendingCsvText.
No CSV file has been written.
```

Then this line is called:

```kotlin
createCsvLauncher.launch(filename)
```

This `launch(...)` is easy to misunderstand.

It does not mean:

```text
open picker
wait here inside onClick
return the user's chosen Uri
continue writing the file inside onClick
```

It means:

```text
send a request to Android:
"please open the file picker for this document"
```

Then `launch(...)` returns quickly.

The `onClick` callback finishes.

What is blocked during `onClick`?

```text
The Main thread is busy while onClick is running.
Compose cannot recompose in the middle of this same onClick callback.
```

But this is usually still short for small beginner data.

In this old code, `onClick` creates the CSV text, creates a filename, and sends the picker request.

For a small CSV file, the Main thread is not blocked for a long time.

### Part C: File Picker Is Open

After `createCsvLauncher.launch(filename)`, Android opens the system file picker.

The picker is not more code continuing inside `onClick`.

`onClick` blcok running has already finished.

Think of it like this:

```text
onClick:
    create CSV text from current measurements
    store it in pendingCsvText
    create filename
    call createCsvLauncher.launch(filename)
    finish

Android/system picker:
    user chooses a location/name
    or user cancels

onResult:
    runs later after Android file picker sends the result back
```

The important timing point is:

```text
The app has already created pendingCsvText.
The app is now waiting for Android to return a Uri.
The actual file has not been written yet.
```

The next section explains in detail what may happen to your app while the picker is open.

### Part D: Result Callback

After the user chooses a file or cancels, Android file picker sends the result back. Android calls:

```kotlin
onResult = { uri: Uri? ->
```

This callback runs back in your app, usually on the Main thread.

The `uri` is the result from Android:

```text
uri != null:
    the user chose a file location

uri == null:
    the user cancelled
```

So this line decides whether there is a file location to write to:

```kotlin
if (uri != null) {
```

If the user cancelled, the app only updates the message:

```kotlin
exportMessage = "CSV export cancelled."
```

If the user chose a file, the app uses the CSV text that was stored earlier:

```kotlin
pendingCsvText
```

This is click-time data, not save-result-time data.

```text
The CSV uses the measurements that existed when onClick created pendingCsvText.
```

Then the app opens a writable stream and writes the file:

```kotlin
context.contentResolver.openOutputStream(uri)?.use { outputStream ->
    outputStream.write(pendingCsvText.toByteArray())
}
```

The key point:

```text
openOutputStream(uri):
    asks Android for a stream to the selected Uri

use { ... }:
    runs immediately and closes the stream afterward

outputStream.write(...):
    is the actual file write
```

The file write is:

```text
not inside onClick
not inside createCsvLauncher.launch(filename)
not inside the system picker
```

It is:
```text
inside onResult
inside the use block
at outputStream.write(...)
```

In this beginner version, treat this callback working in Main thread.

For a small CSV file, this is fine.

For a very large CSV file or a slow storage provider, this could briefly freeze the UI. Later lessons move heavy file work to coroutines and `Dispatchers.IO`.

### Full Timeline

The detailed timeline is:

1. Compose builds the screen and remembers `createCsvLauncher`.
2. The `onResult` callback is stored for later, but does not run yet.
3. User taps Export.
4. Android runs the Button `onClick` callback on your app's Main thread.
5. `onClick` creates CSV text from the current measurements.
6. `onClick` stores that text in `pendingCsvText`.
7. `onClick` creates a suggested filename.
8. `onClick` calls `createCsvLauncher.launch(filename)`.
9. `launch(...)` sends a file-picker request to Android.
10. `launch(...)` returns; it does not wait for the user to choose a file.
11. `onClick` finishes, so the Main thread is free again.
12. Android shows the system file picker.
13. Your visible Activity may be paused, stopped, or covered.
14. Background work may continue if its scope is still alive.
15. The user chooses a location/name, or cancels.
16. Android delivers the result back to your app.
17. The launcher's `onResult` callback runs in your app.
18. If `uri` is not null, your app opens an output stream to the selected `Uri`.
19. Your app writes the stored `pendingCsvText` bytes with `outputStream.write(...)`.
20. Your app updates `exportMessage`.
21. After the callback finishes, Compose can recompose if state changed.

The same timeline as a table:

| Time | Who is active? | What is running? | What is blocked or paused? |
| --- | --- | --- | --- |
| Launcher setup | App composition | `rememberLauncherForActivityResult(...)` creates/remembers the launcher | picker not open; `onResult` not running |
| Button tap | App Main thread | `onClick` starts | recomposition cannot run in the middle of this callback |
| Inside `onClick` | App Main thread | `pendingCsvText` and filename are created | no `Uri`; no file write yet |
| Inside `onClick` | App Main thread | `createCsvLauncher.launch(filename)` sends request | it does not wait for user's choice |
| After `launch(...)` | App Main thread | `onClick` finishes quickly | Main thread becomes free again |
| Picker visible | Android/system picker UI, usually in another process | user chooses location/name or cancels | your visible Activity may be paused/covered |
| Picker visible | Your app background work | acquisition may continue if still alive | UI drawing/ recomposition may pause while covered |
| Result returns | App callback, usually in Main thread | `onResult` receives `Uri?` | UI recomposition waits while callback is running |
| File write | App callback, usually in Main thread | `outputStream.write(...)` in `onResult` writes the stored `pendingCsvText` bytes | Main thread may be blocked during the write |
| Message update | App state update | `exportMessage = ...` changes UI state | recomposition happens after callback work finishes |

Important:

```text
onResult is not the return value of launch(...).
launch(...) returns quickly.
onResult is a separate callback that runs later.
```

This timing explains why a `val` created inside `onClick` cannot be used later in `onResult`.

It also explains why the old method needed `pendingCsvText`.

## 6. What Happens While the File Picker Is Open

The timeline above mentions this briefly.

Now slow down and look at the picker-open period by itself.

When the file picker opens, your app is no longer the visible foreground screen.

The picker is usually a different Activity.

It may also belong to another app or system process.

During this period, think about three pieces separately:

```text
System picker UI:
    foreground screen where the user chooses a file

Your app's visible Activity:
    may be paused, stopped, or covered

Your app's background work:
    may continue if its scope is still alive
```

Your Activity may move to:

```text
onPause
possibly onStop
```

That means:

```text
Your visible app screen may stop drawing.
Your visible app screen may not be recomposing normally while covered/stopped.
Your app process may still be alive.
Your ViewModel may still be alive.
Some background work may continue.
```

Drawing:

```text
Your app may stop drawing to the screen while the picker covers it.
Visible recomposition is effectively paused or delayed because your Activity is not the screen the user is looking at. There is no screen for it to draw on!
```

Main thread:

```text
Your app's Main thread may be idle.
It is not drawing the covered screen, but it is still waiting for Android to report when the user is done with the file picker.
```

More precise version:

```text
The visible Activity can be paused.
The Main thread is not "dead"; after onClick finishes, it is free for other work.
The app process can still exist.
ViewModel state can still exist.
Coroutines can still run if their scope is still active.
```

If a coroutine is launched like this:

```kotlin
viewModelScope.launch {
    // usually starts on Main
}
```

then code in that coroutine usually runs on Main unless it switches dispatcher.

If it does heavy file/device work directly there, it can block Main.

If it switches to IO:

```kotlin
viewModelScope.launch {
    val newMeasurement = withContext(Dispatchers.IO) {
        readMeasurementFromDevice()
    }

    measurements = measurements + newMeasurement
}
```

then:

```text
readMeasurementFromDevice():
    runs on an IO background thread

measurements = measurements + newMeasurement:
    runs after IO finishes, usually back on Main
```

So while the picker is open:

```text
Measurement data may keep changing in memory.
The covered UI may not visibly recompose/draw immediately.
When the app becomes visible again, Compose can show the latest state.
```

So this is wrong:

```text
The file picker is open,
therefore the whole app process is blocked.
```

The better idea is:

```text
The visible Activity may pause,
but other app work may still exist.
```

For this research app, the important example is measurement acquisition.

If measurement is running in a coroutine that is still alive, opening the file picker does not automatically stop that measurement work.

Example:

```text
file picker is open
visible app screen is paused or covered
measurement coroutine may still add new measurements
onResult runs later when the user chooses a file
```

But do not overstate this:

```text
This does not mean background work always continues.
Android can still recreate the Activity.
Android can still kill the app process if needed.
Lifecycle-aware UI collection may pause while the screen is stopped.
```

The safe beginner idea is:

```text
The file picker creates a time gap.
During that gap, app state may or may not change.
So we should be clear about when the CSV text is created.
```

## 7. Recomposition During `onClick`

Another question naturally appears:

```text
If the Main thread is running onClick,
can Compose recompose the UI in the middle of that same onClick?
```

Usually, no.

An `onClick` callback usually runs on the Android Main thread.

Recomposition also needs the Main thread.

Because there is only one Main thread, these jobs take turns.

Example:

```kotlin
Button(
    onClick = {
        exportMessage = "Preparing..."
        createCsvLauncher.launch("measurements.csv")
    }
) {
    Text("Export CSV")
}
```

When `exportMessage` changes, Compose can notice that the UI needs to update.

But the actual recomposition waits until the current callback finishes:

```text
onClick starts
state changes
Compose schedules recomposition
onClick finishes
Main thread becomes free
Compose can recompose
```

Important distinction:

```text
Compose is not waiting for "all UI code everywhere" to finish.
It is waiting for the current Main-thread task to finish.
```

In this example, the current Main-thread task is the `onClick` callback.

After `onClick` reaches its final `}`, the Main thread can take the next task.

That next task might be recomposition, another UI event, an Android lifecycle callback, or something else Android has queued.

So the better rule is:

```text
Recomposition can happen between Main-thread tasks.
Recomposition cannot happen in the middle of the currently running onClick callback.
```

### Composable Functions Are Not Callbacks

A composable function is different from an `onClick` callback.

The composable function runs during recomposition.

When Compose is already recomposing, it runs the needed composable code to describe the UI.

It does not stop halfway through that same recomposition and start another recomposition inside itself.

This is why a local variable is safe while its callback is currently running.

Recomposition does not jump into the middle of the callback and delete the local variable.

But this does not solve the initial `var pendingCsvText` problem.

Why?

Because `onClick` only blocks recomposition during the short time that `onClick` is running.

After `onClick` finishes, the file-picker/result flow is still not finished.

During that later gap, recomposition or Activity recreation may still happen.

So a plain local variable like this is still not enough for the old method:

```kotlin
var pendingCsvText = ""
```

It belongs to one run of the composable.

If the composable runs again before `onResult`, it can become a fresh empty variable again.

### The Useful Side: Batching

The fact that Compose waits until the callback finishes can be useful.

If you update several pieces of state inside one `onClick`, Compose does not need to redraw the UI after every single assignment.

Example:

```kotlin
onClick = {
    state1 = "A"
    state2 = "B"
    state3 = "C"
}
```

Compose can wait until the callback finishes, then update the UI once using the latest state values.

This is efficient.

So this behavior has two sides:

```text
Good:
    multiple state updates can be batched into one UI update

Caution:
    long-running callback code can block the UI from updating smoothly
```

For this reason, Main-thread callbacks should stay quick.

Small export work is okay for the beginner lesson.

Large export work should move to background work later.

## 8. Why Picker Blocking Does Not Make a Local Variable Safe

Previously, we said the file picker may delay or pause visible recomposition/drawing while it covers your Activity.

But this does not mean `pendingCsvText` is absolutely safe.

Suppose the old method tries to store the CSV text in a plain composable-local variable:

```kotlin
var pendingCsvText = ""
```

Why is this still risky?

Because the old method creates the CSV text in `onClick`, but uses it later in `onResult`.

There is a gap between those two moments:

```text
onClick:
    create pendingCsvText
    request the file picker
    finish

time gap:
    picker is open
    app may be paused, covered, resumed, recomposed, or recreated

onResult:
    try to use pendingCsvText
```

During that gap, the picker may cover the UI and may even delay/pause visible recomposition.

But the picker does not guarantee that the old composable-local variable will still be the same one when `onResult` runs.

In Android development, we have to worry about the "Unstable" situations. There can still be many reasons that the composable runs again or the Activity is recreated before the result is delivered.

For example:

```text
The app goes through lifecycle changes while the picker is open.
The device is rotated, changing the Activity configuration.
Android recreates the Activity because the process was under pressure.
```

If that happens, the next composable run creates a fresh `pendingCsvText`:

```kotlin
var pendingCsvText = ""
```

Then `onResult` may run with no CSV text left to write.

That would create a mistake like this:

```text
Expected:
    write the CSV text created in onClick

Actual risk:
    onResult sees a fresh empty pendingCsvText
    the exported file is empty or wrong
```

So although the file picker blocking or covering the UI may reduce visible UI updates for a while, it does not remove the risk.

If we lived in a perfect world where phones never rotated, apps were never killed, and data never updated in the background, your original way would be fine!

But because Android is "messy," we use the onResult approach because it is "Self-Healing." It doesn't care what happened while the user was away—it just looks at the data right now and saves it.

The beginner-safe rule is:

```text
If data must be used later by another callback,
do not keep it only in a plain local variable from one composable run.
```

## 9. The Weakness of the Old Method

The old method creates CSV text before requesting the picker:

```kotlin
onClick = {
    pendingCsvText = measurementsToCsv(measurements)
    createCsvLauncher.launch("measurements.csv")
}
```

The old method has two main weaknesses:

```text
Weakness 1:
    It creates the CSV too early.

Weakness 2:
    It needs extra remembered mutable state just to move data from onClick to onResult.
```

### Weakness 1: The CSV Is Created Too Early

Because `measurementsToCsv(measurements)` runs in `onClick`, the old method creates a click-time snapshot.

Example:

```text
10:00:00
User clicks Export.
pendingCsvText is created with 10 measurements.

10:00:15
User chooses the file location.
The app now has 25 measurements.

Export result:
The file still contains 10 measurements.
```

This is not always wrong.

It is correct if Export means:

```text
Export exactly what existed when the user clicked the button.
```

But for a live measurement app, this may be surprising.

The user may expect the file to include the measurements available when they finally choose the save location.

### Weakness 2: It Needs a State Bridge Between Callbacks

The old method has to carry CSV text from `onClick` to `onResult`.

Because those callbacks run at different times, a plain local variable is not enough.

So the old code creates a remembered mutable value:

```kotlin
var pendingCsvText by remember {
    mutableStateOf("")
}
```

This works, but it makes the export code heavier than it needs to be.

Every `mutableStateOf` you add makes the code slightly more complex to track.

With the old method, `pendingCsvText` adds this extra burden:

```text
One more variable to manage:
    pendingCsvText starts empty, changes in onClick, and is read later in onResult.

One more state update:
    assigning pendingCsvText can schedule recomposition, even though the UI does not show it.

More work inside onClick:
    onClick creates the CSV text before it opens the file picker.
```

It also keeps the CSV text alive longer than necessary:

```text
Old method:
    The app stores the entire CSV string in memory while the file picker is open.

Cleaner method:
    The app creates the string only when it is needed in onResult.
    After the file is written, that temporary string is no longer needed.
```

The awkward part is that `pendingCsvText` is not really screen state.

The UI does not display it and does not need to react to it.

It is only temporary data for writing a file.

So the old method uses remembered Compose state as a bridge between two callbacks, not because the UI truly needs that state.

## 10. The Cleaner Method: Create CSV in `onResult`

The cleaner method waits until `onResult` to create the CSV text.

```kotlin
val createCsvLauncher = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.CreateDocument("text/csv"),
    onResult = { uri: Uri? ->
        if (uri != null) {
            val csvText = measurementsToCsv(measurements)

            context.contentResolver.openOutputStream(uri)?.use { outputStream ->
                outputStream.write(csvText.toByteArray())
            }
        }
    }
)

Button(
    onClick = {
        createCsvLauncher.launch("measurements.csv")
    }
) {
    Text("Export CSV")
}
```

Read it as:

```text
onClick:
    call createCsvLauncher.launch(...)
    request the file picker
    finish quickly

onResult:
    run later after the picker returns
    create CSV text from the current measurements
    write the file
```

This solves the main issues of the old method:

```text
No pendingCsvText state is needed.
No CSV string is stored while the picker is open.
The CSV is created at the moment the file is written.
```

For a live measurement app, this is often the better default.

## 11. Why Local `val csvText` Is Not the Same Problem

The cleaner method still has this line:

```kotlin
val csvText = measurementsToCsv(measurements)
```

So it is natural to ask:

```text
Are we still storing the CSV text?
```

Yes, but only briefly.

This `csvText` is a local variable inside the `onResult` callback.

It is created and used during the same `onResult` callback, and even UI recomposition does not interrupt this callback while it is running.

After the callback finishes, your code cannot use that local variable anymore, and the memory becomes available for garbage collection later.

Compare:

```text
pendingCsvText:
    remembered Compose state
    exists across recompositions while remembered
    used to carry data from onClick to onResult

local csvText:
    normal local variable
    exists only while onResult is running
    used immediately to write the file
```

The problem is not:

```text
Do we use val or not?
```

The better question is:

```text
Does this value need to live as screen state,
or is it only temporary data for this callback?
```

## 12. Snapshot Export Versus Latest Export

Now the design choice has a name.

### Snapshot Export

Create CSV text in `onClick`:

```kotlin
onClick = {
    pendingCsvText = measurementsToCsv(measurements)
    createCsvLauncher.launch("measurements.csv")
}
```

Meaning:

```text
Export the data as it existed when the user clicked Export.
```

Use this if clicking Export should freeze a deliberate snapshot.

### Latest Export

Create CSV text in `onResult`:

```kotlin
onResult = { uri: Uri? ->
    if (uri != null) {
        val csvText = measurementsToCsv(measurements)
        ...
    }
}
```

Meaning:

```text
Export the data available when the app writes the file.
```

Use this if the file should include measurements that may arrive while the picker is open.

Important limit:

```text
"Latest" means latest data available to the code at that moment.
It does not magically recover data if the app process was suddenly killed
and the data has not been saved anywhere.
```

In later architecture, the current source of truth may be:

```text
ViewModel state
repository
database
```

For important research data, saving to a reliable source before export is better than relying only on temporary screen memory.

## 13. A Better Beginner Version

For the simple Lesson 7 app, a clean export version is:

```kotlin
val createCsvLauncher = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.CreateDocument("text/csv"),
    onResult = { uri: Uri? ->
        if (uri != null) {
            val csvText = measurementsToCsv(measurements)

            context.contentResolver.openOutputStream(uri)?.use { outputStream ->
                outputStream.write(csvText.toByteArray())
            }

            exportMessage = "CSV exported successfully."
        } else {
            exportMessage = "CSV export cancelled."
        }
    }
)

Button(
    onClick = {
        val filename = if (sampleId.isNotBlank()) {
            "${safeFilename(sampleId)}_measurements.csv"
        } else {
            "measurements.csv"
        }

        createCsvLauncher.launch(filename)
    },
    enabled = measurements.isNotEmpty()
) {
    Text("Export CSV")
}
```

This keeps:

```text
filename choice in onClick
CSV generation in onResult
file writing in onResult
status message as UI state
```

Why is `exportMessage` state, but `csvText` is not?

```text
exportMessage is displayed by the UI.
csvText is temporary data used only to write a file.
```

## 14. Remaining Issue: Small Files Versus Large Files

In this lesson, the CSV is first built as one complete `String`:

```kotlin
val csvText = measurementsToCsv(measurements)

context.contentResolver.openOutputStream(uri)?.use { outputStream ->
    outputStream.write(csvText.toByteArray())
}
```

This is acceptable for small CSV files.

For very large exports, however, this approach can become inefficient:

```text
build one huge String in memory
convert the whole String to bytes at once
write all those bytes in one operation
```

Usually, the app does not need to keep the whole CSV text as a single `String` unless it wants to display that text in the UI, which is uncommon.

Later lessons introduce coroutines and `Dispatchers.IO` for heavier export work.

In a larger app, heavy export should move off the Main thread:

```kotlin
viewModelScope.launch {
    val csvText = withContext(Dispatchers.Default) {
        measurementsToCsv(measurements)
    }

    withContext(Dispatchers.IO) {
        context.contentResolver.openOutputStream(uri)?.use { outputStream ->
            outputStream.write(csvText.toByteArray())
        }
    }
}
```

Beginner idea:

```text
Small export:
    direct callback code is okay.

Large export:
    prepare/write in background work.
```

## 15. Final Checklist

When you see export code, ask:

```text
Can onResult see a val created inside onClick?
    -> no, because it is local to onClick

Why does the old method need pendingCsvText?
    -> to carry CSV text from onClick to onResult

Why does the old method need var?
    -> because pendingCsvText is reassigned

Why does the old method need remember?
    -> because the value must survive until onResult

Does the UI display or react to this value?
    -> use Compose state if yes

Is this value only needed while writing the file?
    -> use a local val inside onResult

Should Export mean "data at click time"?
    -> create CSV in onClick

Should Export mean "data at save time"?
    -> create CSV in onResult

Could the export be large or slow?
    -> use background work in a later lesson
```

The main rule:

```text
State is for values the UI displays, edits, remembers, or reacts to.
Local variables are for temporary work inside one function or callback.
```

The main file-picker rule:

```text
onClick starts the request.
onResult handles the returned Uri.
Create the CSV where it best matches the meaning of export.
```
