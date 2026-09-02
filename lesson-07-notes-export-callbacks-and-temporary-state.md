# Lesson 7 Notes - Export Callbacks, Temporary State, and Recomposition

This note starts from the export method used in Lesson 7.

That method used a variable named `pendingCsvText`:

```kotlin
var pendingCsvText by remember {
    mutableStateOf("")
}
```

At first, this can feel strange.

Why do we need a remembered mutable state variable just to export a CSV file?

This note explains that question step by step.

This note is organized in this order:

1. Start from the old `pendingCsvText` method.
2. Understand why that method needs a temporary variable.
3. Understand why the variable used `var`, `remember`, and `mutableStateOf`.
4. Understand what a callback is.
5. Understand how the Android file picker and `onResult` callback work.
6. Follow the exact timing of `onClick`, `launch(...)`, the picker, and `onResult`.
7. See what may continue while the file picker is open.
8. Understand the weakness of the old method.
9. Move to the cleaner callback-local method.
10. Compare `pendingCsvText` with a local `val csvText`.
11. Decide when each export style makes sense.
12. Use the final checklist.

## 1. The Old Method

The first Lesson 7 export code used this pattern:

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
        }
    }
)

Button(
    onClick = {
        pendingCsvText = measurementsToCsv(measurements)
        createCsvLauncher.launch("measurements.csv")
    }
) {
    Text("Export CSV")
}
```

Read it as:

```text
1. User clicks Export.
2. App converts measurements to CSV text.
3. App stores that text in pendingCsvText.
4. App requests the Android file picker.
5. Later, Android returns a Uri.
6. App writes pendingCsvText to that Uri.
```

This method works.

But it only makes sense after we understand one important Android idea:

```text
The file picker does not return a file location immediately.
```

## 2. Why the Old Method Needed a Temporary Variable

When the user clicks the button, this line runs:

```kotlin
createCsvLauncher.launch("measurements.csv")
```

This does not save the file immediately.

It only sends a request to Android:

```text
Please open the system file picker for this document.
```

After `launch(...)` sends that request, the `onClick` callback can finish.

The user may then spend time choosing a folder, changing the filename, or cancelling.

Only later does Android call:

```kotlin
onResult = { uri: Uri? ->
    ...
}
```

So there is a time gap:

```text
onClick runs now.
onResult runs later.
```

The old method creates the CSV during `onClick`.

But the file can only be written later inside `onResult`.

So the old method needs somewhere to keep the CSV text during that gap:

```text
onClick:
    create CSV text
    store it somewhere

onResult:
    retrieve that stored CSV text
    write it to the Uri
```

That "somewhere" is `pendingCsvText`.

## 3. Why `pendingCsvText` Was `var`, `remember`, and `mutableStateOf`

The old method used:

```kotlin
var pendingCsvText by remember {
    mutableStateOf("")
}
```

Each part has a reason.

### Why `var`

The value starts as an empty string:

```kotlin
""
```

Later, when the user clicks Export, the code replaces it:

```kotlin
pendingCsvText = measurementsToCsv(measurements)
```

Because the variable is reassigned with `=`, it must be `var`.

A `val` would not allow this reassignment.

### Why `remember`

Composable functions can recompose.

That means Compose can rerun the composable function to describe the latest UI.

If we used a normal local variable:

```kotlin
var pendingCsvText = ""
```

then the variable could be recreated when the composable runs again.

That is risky because the file picker result comes back later.

A common confusion is:

```text
If onClick blocks recomposition while it is running,
why can pendingCsvText still be lost?
```

The answer is:

```text
onClick only blocks recomposition during the short time that onClick is running.
createCsvLauncher.launch(...) is called inside onClick,
but it only starts the file-picker request.
Then onClick finishes.
The picker/result flow continues outside that onClick callback.
During the gap before onResult, recomposition or Activity recreation may still happen.
```

So this is not enough:

```kotlin
var pendingCsvText = ""
```

Even if you assign it in `onClick`, it is still just a normal variable from one run of the composable.

If the composable runs again before `onResult`, this line creates a fresh empty variable again:

```kotlin
var pendingCsvText = ""
```

The old method wants the CSV text to survive between:

```text
button click
file picker result
```

So it uses `remember`.

### Why `mutableStateOf`

`mutableStateOf` creates Compose-observable state.

That is useful when the UI displays or reacts to the value.

For example:

```kotlin
var exportMessage by remember {
    mutableStateOf("")
}
```

`exportMessage` should be state because the UI displays it.

For `pendingCsvText`, the reason is weaker.

The UI usually does not display the raw CSV text.

In the old method, `mutableStateOf` is mainly being used as a remembered mutable holder:

```text
Store the CSV text now.
Read it later in onResult.
```

So the old method is understandable, but not ideal.

It uses screen state for data that is not really part of the screen.

## 4. What Is a Callback?

A callback is code that you give to something else, so that it can call your code later.

Simple idea:

```text
I cannot finish the work right now.
When the event happens later, call this code.
```

In Compose and Android, callbacks are everywhere.

For example, `onClick` is a callback:

```kotlin
Button(
    onClick = {
        exportMessage = "Export clicked"
    }
) {
    Text("Export CSV")
}
```

You give the `Button` some code.

The `Button` does not run that code immediately when the UI is drawn.

It runs that code later when the user clicks the button.

So:

```text
onClick:
    callback for a future click event
```

`onResult` is also a callback:

```kotlin
val createCsvLauncher = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.CreateDocument("text/csv"),
    onResult = { uri: Uri? ->
        // This runs later when Android returns the result.
    }
)
```

You give Android some code.

Android does not run it immediately when the launcher is created.

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

That difference between where code is written and when code runs is the key to understanding this export logic.

## 5. How the File Picker Really Works

Android file export with `ActivityResultContracts.CreateDocument` has two separate stages.

Stage 1 starts from your app:

```kotlin
createCsvLauncher.launch("measurements.csv")
```

Meaning:

```text
Send a request to Android to open a system UI
where the user can create a document.
```

Stage 2 comes back later:

```kotlin
onResult = { uri: Uri? ->
    ...
}
```

Meaning:

```text
Android gives your app the result.
If uri is not null, your app can write to that chosen location.
If uri is null, the user cancelled.
```

Important:

```text
onResult is not the return value of launch(...).
launch(...) returns quickly.
onResult is a separate callback that runs later.
```

So the important flow is:

```text
onClick:
    calls createCsvLauncher.launch(...)
    sends the file-picker request
    finishes quickly

system file picker:
    opens/takes over after the request
    user chooses location or cancels

onResult:
    runs later
    receives the Uri and writes the file
```

This is why export code often feels unusual.

It is not one continuous block of code that immediately saves a file.

It is an event now plus a callback later.

## 6. Exact Timing: Who Runs When?

This is the most important timing picture.

```text
1. User taps Export.
2. Android runs the Button onClick callback on your app's Main thread.
3. onClick may create CSV text, depending on the export style.
4. onClick calls createCsvLauncher.launch(...).
5. launch(...) sends a file-picker request to Android.
6. launch(...) returns; it does not wait for the user to choose a file.
7. onClick finishes.
8. Android shows the system file picker.
9. The user chooses a location/name, or cancels.
10. Android delivers the result back to your app.
11. The launcher's onResult callback runs in your app.
12. In this beginner version, your app writes the CSV from that callback if the Uri is not null.
```

So this line:

```kotlin
createCsvLauncher.launch("measurements.csv")
```

is called inside `onClick`.

But the user's file choice is not returned inside `onClick`.

The result comes later through `onResult`.

Another way to see it:

| Time | Who is active? | What happens? |
| --- | --- | --- |
| Button tap | App Main thread | `onClick` starts |
| Inside `onClick` | App Main thread | app calls `createCsvLauncher.launch(...)` |
| After `launch(...)` | App Main thread | `onClick` continues and finishes quickly |
| Picker visible | Android/system picker UI | user chooses location or cancels |
| Result returns | App callback, usually Main thread | `onResult` receives `Uri?` |
| File write | App callback code | app writes CSV to the selected `Uri` |

In this beginner code, treat `onResult` like other UI callbacks:

```text
It should stay quick.
Small file writing is okay for the lesson.
Large file writing should move to background work later.
```

This timing explains why a `val` created inside `onClick` is not visible later in `onResult`.

It also explains why the old method needed a value that survived from `onClick` to `onResult`.

## 7. What May Happen While the File Picker Is Open

The file picker is usually a different Activity.

It may also be part of another app or system process.

During this period, there are three different pieces to think about:

```text
System picker UI:
    foreground screen where the user chooses a file

Your app's visible Activity:
    may be paused, stopped, or covered

Your app's background work:
    may continue if its scope is still alive
```

When it opens, your own Activity may move to:

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

So "the UI is not visible" does not mean:

```text
the whole app process is blocked
```

It means the visible Activity is no longer the foreground screen.

For this research app, the important example is measurement acquisition.

If measurement is running in a coroutine that is still alive, opening the file picker does not automatically stop that measurement work.

For example:

```text
file picker is open
visible app screen is paused or covered
measurement coroutine may still add new measurements
onResult runs later when the user chooses a file
```

But be careful:

```text
This does not mean background work always continues.
Android can still recreate the Activity.
Android can still kill the app process if needed.
Lifecycle-aware UI collection may pause while the screen is stopped.
```

The safe beginner idea is:

```text
The file picker can create a time gap.
During that gap, app state may or may not change.
So we should be clear about when the CSV text is created.
```

## 8. The Weakness of the Old Method

The old method creates CSV text before requesting the picker:

```kotlin
onClick = {
    pendingCsvText = measurementsToCsv(measurements)
    createCsvLauncher.launch("measurements.csv")
}
```

This creates a click-time snapshot.

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

The old method also keeps a CSV string in remembered state while the picker is open.

For small data this is not dangerous.

But conceptually, the CSV text is not really UI state.

It is temporary file-writing data.

## 9. The Cleaner Method: Create CSV in `onResult`

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

## 10. Why Local `val csvText` Is Not the Same Problem

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

It exists only during that callback execution.

After the callback finishes, your code cannot use that local variable anymore.

More precisely, the value becomes unreachable by your code, and the memory becomes available for garbage collection later.

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

So the problem is not:

```text
Do we use val or not?
```

The better question is:

```text
Does this value need to live as screen state,
or is it only temporary data for this callback?
```

## 11. Recomposition and the Callback

Recomposition means Compose reruns composable code to describe the latest UI.

But recomposition does not interrupt an `onClick` callback halfway through.

An `onClick` callback usually runs on the Android Main thread.

Recomposition also needs the Main thread.

Because there is only one Main thread, these jobs take turns.

If state changes inside `onClick`, Compose can schedule recomposition:

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

But the actual recomposition waits until the callback finishes:

```text
onClick starts
state changes
Compose schedules recomposition
onClick finishes
Main thread becomes free
Compose can recompose
```

This also means a local `val` inside `onResult` is safe while `onResult` is running.

Recomposition does not reach into the middle of that callback and delete the local variable.

The real caution is:

```text
Keep Main-thread callbacks quick.
```

If a callback builds a huge CSV file or writes a huge file on Main, the UI may feel frozen.

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
It does not magically recover data if the app process was killed
and the data was not saved anywhere.
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

## 14. Small Files Versus Large Files

In Lesson 7, direct callback export is acceptable for small CSV files:

```kotlin
val csvText = measurementsToCsv(measurements)

context.contentResolver.openOutputStream(uri)?.use { outputStream ->
    outputStream.write(csvText.toByteArray())
}
```

For small beginner examples, this is fine.

For very large exports, this can become slow:

```text
building a huge String
converting it to bytes
writing many bytes to storage
```

Later lessons introduce coroutines and `Dispatchers.IO`.

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
Why do we need a callback?
    -> because the file picker returns a Uri later

Does this value need to survive from onClick to onResult?
    -> the old method used pendingCsvText for that

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
