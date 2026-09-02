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
4. Understand how the Android file picker and `onResult` callback work.
5. See what may continue while the file picker is open.
6. Understand the weakness of the old method.
7. Move to the cleaner callback-local method.
8. Compare `pendingCsvText` with a local `val csvText`.
9. Decide when each export style makes sense.
10. Use the final checklist.

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
4. App opens the Android file picker.
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

It only asks Android to open the system file picker.

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
After onClick finishes, the file picker is open and onResult has not returned yet.
During that gap, recomposition or Activity recreation may still happen.
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

## 4. How the File Picker Really Works

Android file export with `ActivityResultContracts.CreateDocument` has two separate stages.

Stage 1 starts from your app:

```kotlin
createCsvLauncher.launch("measurements.csv")
```

Meaning:

```text
Ask Android to open a system UI where the user can create a document.
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

So the important flow is:

```text
onClick:
    starts the request

system file picker:
    user chooses location or cancels

onResult:
    receives the Uri and writes the file
```

This is why export code often feels unusual.

It is not one continuous block of code that immediately saves a file.

It is an event now plus a callback later.

## 5. What May Happen While the File Picker Is Open

The file picker is usually a different Activity.

It may also be part of another app or system process.

When it opens, your own Activity may move to:

```text
onPause
possibly onStop
```

That means:

```text
Your visible app screen may stop drawing.
Your app process may still be alive.
Your ViewModel may still be alive.
Some background work may continue.
```

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

## 6. The Weakness of the Old Method

The old method creates CSV text before the picker opens:

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

## 7. The Cleaner Method

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
    open the file picker

onResult:
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

## 8. Why Local `val csvText` Is Not the Same Problem

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

## 9. Recomposition and the Callback

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

## 10. Snapshot Export Versus Latest Export

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

## 11. A Better Beginner Version

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

## 12. Small Files Versus Large Files

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

## 13. Final Checklist

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
