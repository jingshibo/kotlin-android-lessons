# Lesson 7 Notes - Export Callbacks, Temporary State, and Recomposition

Lesson 7 teaches CSV export.

At first, export sounds simple:

```text
User clicks Export -> app saves a CSV file
```

But Android does not usually let your app directly choose an ordinary file path.

Instead, your app asks Android to open a system file picker. The user chooses where to save the file, and Android returns a `Uri` later.

That delay is the key idea in this note.

Because the file picker returns later, we need to decide:

```text
Should the app create CSV text before the picker opens?
Or should the app create CSV text after the picker returns?
```

This note is organized in this order:

1. Understand the two-step file-picker workflow.
2. Understand the launcher and `onResult` callback.
3. Start from the old `pendingCsvText` method.
4. Understand what the old method means.
5. Move to the cleaner callback-local method.
6. Compare Compose state with a local `val`.
7. Connect this to recomposition and Activity lifecycle.
8. Choose between snapshot export and latest export.
9. Decide when background work is needed.
10. Use the final checklist.

## 1. The File Picker Workflow

Android CSV export with `ActivityResultContracts.CreateDocument` is a two-step workflow.

```text
Step 1:
Your app launches the file picker.

Step 2:
Later, Android returns a Uri,
and your app writes data to that Uri.
```

The first step usually happens in `onClick`:

```kotlin
Button(
    onClick = {
        createCsvLauncher.launch("measurements.csv")
    }
) {
    Text("Export CSV")
}
```

The second step happens in `onResult`:

```kotlin
val createCsvLauncher = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.CreateDocument("text/csv"),
    onResult = { uri: Uri? ->
        if (uri != null) {
            // Write the CSV file here.
        }
    }
)
```

So the button click does not directly write the file.

It only starts the file-picker request.

The real file writing happens later, after Android gives your app a `Uri`.

This is why the export code needs a callback.

## 2. What the Launcher Remembers

This code creates a launcher:

```kotlin
val createCsvLauncher = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.CreateDocument("text/csv"),
    onResult = { uri: Uri? ->
        ...
    }
)
```

`rememberLauncherForActivityResult` remembers the launcher across recompositions.

It also keeps the `onResult` callback connected so Android can deliver the result later.

But this is important:

```text
Remembering the launcher does not mean local variables inside onResult already exist.
```

Example:

```kotlin
onResult = { uri: Uri? ->
    val csvText = measurementsToCsv(measurements)
    ...
}
```

Here, `csvText` is created only when `onResult` actually runs.

It is not created when the screen first draws.

It is not recreated during every recomposition.

It is normal local Kotlin code inside a callback.

## 3. The Old Method: Store CSV Before Opening the Picker

The first Lesson 7 version used `pendingCsvText`.

```kotlin
var pendingCsvText by remember {
    mutableStateOf("")
}
```

When the user clicked Export, the app created the CSV text immediately:

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

Then, when the file picker returned, the app wrote the already-created text:

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
    create CSV text now
    store it in pendingCsvText
    open the file picker

onResult:
    write the stored CSV text
```

This method works, and it is easy to understand.

But it has a specific meaning.

## 4. What the Old Method Means

The old method creates a click-time snapshot.

```text
Click-time snapshot:
    the exported CSV contains the measurements that existed
    when the user clicked Export
```

For example:

```text
10:00:00
User clicks Export.
pendingCsvText is created with 10 measurements.

10:00:15
User chooses the file location.
The app now has 25 measurements.

Export result:
The file still contains 10 measurements,
because pendingCsvText was created at 10:00:00.
```

This is not automatically wrong.

Sometimes a snapshot is exactly what you want.

But in a live measurement app, it may be surprising because new measurements can arrive while the file picker is open.

The old method also stores CSV text as Compose state.

That raises another question:

```text
Is CSV text really UI state?
```

Usually, no.

CSV text is usually temporary data for writing a file.

It is not something the UI needs to display, edit, or react to.

That leads to the cleaner method.

## 5. The Cleaner Method: Create CSV Inside `onResult`

Instead of preparing CSV before the picker opens, prepare it after the picker returns:

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
```

Then the button only launches the picker:

```kotlin
Button(
    onClick = {
        createCsvLauncher.launch("measurements.csv")
    }
) {
    Text("Export CSV")
}
```

Read the cleaner method as:

```text
onClick:
    open the file picker

onResult:
    create CSV text from the current measurements
    write the file
```

This method is often better because:

```text
CSV text is created only when it is needed.
CSV text is not stored as screen state.
The export uses the data available when the file is written.
```

For a live measurement app, this is usually the better beginner default.

## 6. Compose State Versus Local `val`

Now the variable difference matters.

The old method used Compose state:

```kotlin
var pendingCsvText by remember {
    mutableStateOf("")
}
```

The cleaner method uses a local `val`:

```kotlin
onResult = { uri: Uri? ->
    val csvText = measurementsToCsv(measurements)
    ...
}
```

They are not the same kind of storage.

### Compose State

Use Compose state for values the UI needs to remember or react to.

Examples:

```text
sample ID typed by the user
selected option
current screen message
export success or cancelled message
```

This is state:

```kotlin
var exportMessage by remember {
    mutableStateOf("")
}
```

That makes sense because the UI displays `exportMessage`.

### Local `val`

Use a local `val` for temporary work inside one function or callback.

Example:

```kotlin
val csvText = measurementsToCsv(measurements)
```

This value exists only while that callback is running.

After the callback finishes, your code cannot use that local variable anymore.

More precisely, the value becomes unreachable by your code, and the memory becomes available for garbage collection later.

So:

```text
pendingCsvText:
    remembered screen state

csvText inside onResult:
    temporary local value
```

The problem is not `val`.

The important question is:

```text
Is this value part of the screen state,
or is it only temporary work for this callback?
```

## 7. Why a Local `val` Is Still Useful

You could write the export in one line:

```kotlin
outputStream.write(measurementsToCsv(measurements).toByteArray())
```

But developers often use a local `val` because it is easier to read:

```kotlin
val csvText = measurementsToCsv(measurements)
val bytes = csvText.toByteArray()

outputStream.write(bytes)
```

Both styles create the CSV when `onResult` runs.

The local `val` does not make it Compose state.

It is just a temporary name for a temporary value.

## 8. Recomposition and Callbacks

Recomposition means Compose reruns composable code to describe the latest UI on the Main thread.

An `onClick` callback also usually runs on the Android Main thread.

Because there is only one Main thread, these two jobs must take turns.

So if an `onClick` callback is currently running, Compose cannot recompose in the middle of that same callback.

Compose can notice that state changed and then schedule recomposition, but the actual recomposition waits until the `onClick` callback finishes.

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

If `exportMessage` changes, Compose schedules a UI update.

But recomposition waits until the current callback finishes.

```text
onClick starts
state changes
Compose schedules recomposition
onClick finishes
Main thread becomes free
Compose can recompose
```

So recomposition does not delete a local `val` while its callback is running.

The local variable is safe during that callback execution.

The real caution is different:

```text
Keep Main-thread callbacks quick.
```

If a callback does heavy work for several seconds, the UI cannot update smoothly during that time.

## 9. What Happens While the File Picker Is Open

When you call:

```kotlin
createCsvLauncher.launch("measurements.csv")
```

your app asks Android to open a system UI for creating a document.

That picker is usually a different Activity. It may be run on another system process.

Your own Activity may move to:

```text
onPause
possibly onStop
```

That means:

```text
Your app's visible UI may stop drawing.
Your app process may still be alive.
Your ViewModel may still be alive.
Some background work may continue, depending on how it was started.
```

For this research app, the important example is measurement acquisition.

If measurement is running in a coroutine that is still alive, then opening the file picker does not automatically stop that measurement work because it runs on a different process.

For example, a `viewModelScope` acquisition loop may continue while the `ViewModel` is alive:

```text
file picker is open
visible app screen is paused or covered
measurement coroutine may still add new measurements
onResult runs later when the user chooses a file
```

This is why export timing matters.

If CSV text was created before the picker opened, the file may miss measurements collected while the picker was open.

If CSV text is created inside `onResult`, it can include the measurements available at the moment the file is written.

But do not overstate this:

```text
Android can still destroy and recreate an Activity.
Android can still kill a background process if resources are needed.
Some lifecycle-aware UI collection may pause while the screen is stopped.
```

The safe beginner mental model is:

```text
The visible screen may pause.
The app may still have state or background work alive.
When the picker returns, onResult runs if Android can deliver the result.
```

## 10. Snapshot Export Versus Latest Export

Now we can name the design choice.

### Snapshot Export

Create the CSV before opening the file picker:

```kotlin
onClick = {
    pendingCsvText = measurementsToCsv(measurements)
    createCsvLauncher.launch("measurements.csv")
}
```

Meaning:

```text
Export the data as it looked when the user clicked Export.
```

Use this if the button should freeze a deliberate snapshot.

### Latest Export

Create the CSV after the file picker returns:

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

But this work still runs in the callback.

For small beginner examples, that is fine.

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
Does this variable need to be displayed by the UI?
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
