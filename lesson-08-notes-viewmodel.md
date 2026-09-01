# Lesson 8 Notes - How to Think About ViewModel

This note explains how to judge what belongs in a `ViewModel`.

一个基本的认识如下： 原本 lesson 07 中的UI plot函数里同时包含了各类变量的定义，UI的绘制，和状态的更新。
- 其中有一部分变量是需要记忆和多次更新的状态量。这些量可以通过类进行组织，然后在ViewModel中进行实例化，产生一个持续存在的mutable变量。这些变量可以被外部读取，其更新也会被UI自动探测以执行相应的操作，但对其的修改只能调用在ViewModel内部的函数完成，而无法在外部直接对其赋值。
- 另一部分变量是临时设置的UI参量，可能是mutable或普通变量，它们仍然留在UI中。比如，`pendingCsvText` 的存在就是仅为了支持文件选择流程，虽然这也是一个mutable变量，我们不放入ViewModel。再比如 `meanvalue`是基于state的值得到的一个普通变量，直接在UI中定义就行了，也不必放入ViewModel。
- 所有的状态更新过程都放入ViewModel，只有ViewModel内部的函数可以对状态量进行更新。当UI上发生了一个event（比如onClick），这个event会自动调用ViewModel中的函数，这个函数会实现状态量的更新。此时UI会探测到状态量的变化，然后自动重刷UI界面，显示最新的状态值。

The goal is not to memorize a rule like:

```text
Put all functions in ViewModel.
```

That is too broad.

The better goal is:

```text
Understand what job the ViewModel has.
Then decide whether a variable or function belongs to that job.
```

## 1. The core problem

Before using a ViewModel, one composable may do too many jobs:

```text
ResearchScreen()
    -> defines state variables
    -> draws UI
    -> updates state
    -> creates fake measurements
    -> prepares CSV export
    -> handles button clicks
```

That works for a small lesson, but it becomes hard to read as the app grows.

So we separate the app into two main parts:

```text
Composable UI
    -> shows the screen
    -> receives user actions
    -> calls ViewModel functions

ViewModel
    -> holds screen state
    -> decides how state changes
    -> handles screen-level logic
```

The simple mental model:

```text
UI draws.
ViewModel remembers and decides.
```

## 2. What ViewModel looks like

A ViewModel looks similar to a normal Kotlin class:

```kotlin
class ResearchViewModel : ViewModel() {

    var uiState by mutableStateOf(ResearchUiState())
        private set

    fun updateSampleId(newSampleId: String) {
        uiState = uiState.copy(
            sampleId = newSampleId
        )
    }
}
```

It has:

- properties
- functions
- normal Kotlin code

The special part is not the appearance.

The special part is how Android manages it.

```text
Regular class:
You usually create it yourself with SomeClass().

ViewModel:
Android/Compose gives it to you with viewModel().
```

This matters because a composable can re-run many times.

If you create a normal object directly inside a composable, it may be recreated too often.

But a ViewModel is kept by Android as the screen's state holder.

## 3. Regular class versus ViewModel

| Regular class | ViewModel |
|---|---|
| Created directly with `SomeClass()` | Usually received with `viewModel()` |
| Lifetime is controlled by your code | Lifetime is controlled by Android |
| Good for helper logic or data logic | Good for screen state and screen behavior |
| May be recreated whenever you create it again | Can survive recomposition and screen rotation |
| Does not have ViewModel lifecycle behavior | Has ViewModel lifecycle behavior |

So:

```kotlin
class ResearchViewModel : ViewModel()
```

means:

```text
ResearchViewModel is still a Kotlin class.
But it is also an Android-managed screen state holder.
```

## 4. The main ViewModel pattern

In this lesson, we use one data class to describe the screen state:

```kotlin
data class ResearchUiState(
    val sampleId: String = "",
    val isConnected: Boolean = false,
    val measurements: List<Measurement> = emptyList(),
    val exportMessage: String = ""
)
```

Then the ViewModel owns one mutable state value:

```kotlin
var uiState by mutableStateOf(ResearchUiState())
    private set
```

This means:

```text
uiState is the current snapshot of the screen state.
The UI can read it.
Only the ViewModel can replace it.
```

The UI should not directly change it:

```kotlin
viewModel.uiState = ...
```

Instead, the UI calls ViewModel functions:

```kotlin
viewModel.updateSampleId("S001")
viewModel.addMeasurement()
viewModel.clearMeasurements()
```

Then the ViewModel decides how `uiState` should change.

## 5. Why private set is useful

This code:

```kotlin
var uiState by mutableStateOf(ResearchUiState())
    private set
```

means:

```text
Other code can read uiState.
Only this ViewModel can assign a new uiState.
```

So the UI can do this:

```kotlin
val uiState = viewModel.uiState
```

But the UI cannot do this:

```kotlin
viewModel.uiState = ResearchUiState()
```

That is good because it keeps state changes controlled.

The pattern becomes:

```text
UI reads state.
UI sends events.
ViewModel updates state.
```

## 6. What should go in ViewModel?

A good beginner rule:

```text
Put code in ViewModel if it belongs to the screen's state, behavior, or user flow.
```

Good candidates:

| Code | Why it fits ViewModel |
|---|---|
| `sampleId` | screen state |
| `isConnected` | screen state |
| `measurements` | screen state |
| `exportMessage` | screen state |
| `updateSampleId()` | user typing changes state |
| `toggleConnection()` | user action changes state |
| `addMeasurement()` | screen-level event and state update |
| `clearMeasurements()` | screen-level event and state update |
| `setExportMessage()` | updates message shown by screen |
| `prepareCsvExport()` | prepares app data for an export action |

The ViewModel can also contain helper functions that are not called directly by UI:

```kotlin
private fun nextRepetitionFor(sampleId: String): Int {
    return uiState.measurements.count {
        it.sampleId == sampleId
    } + 1
}
```

This still belongs in the ViewModel if it supports screen state logic.

So the rule is not:

```text
Only onClick functions go in ViewModel.
```

The better rule is:

```text
Screen state and screen behavior go in ViewModel.
```

## 7. What should stay in the Composable?

Composable code should mostly handle UI:

| Code | Why it stays in UI |
|---|---|
| `Text(...)` | drawing UI |
| `Button(...)` | drawing UI |
| `TextField(...)` | drawing UI |
| `LazyColumn(...)` | drawing UI |
| `rememberLauncherForActivityResult(...)` | Android UI/system launcher |
| `LocalContext.current` | Android UI context |
| `createCsvLauncher.launch(filename)` | opens system file picker |
| temporary UI-only values | supports UI interaction only |

For example:

```kotlin
var pendingCsvText by remember {
    mutableStateOf("")
}
```

This can stay in the composable because it is temporary state for the file picker flow.

It is not core research state.

It exists only because the UI starts the file picker first, then receives the result later.

## 8. Temporary UI state versus screen state

Ask:

```text
Does this value describe the meaning of the screen?
Or does it only help one UI tool/API work?
```

Screen state should usually go in the ViewModel:

```text
sampleId
isConnected
measurements
exportMessage
```

Temporary UI state can stay in the composable:

```text
pendingCsvText for the file picker callback
whether a dialog is currently open
temporary text before confirming an action
scroll position
focus state
```

The boundary is not always perfect. That is normal.

When unsure, ask:

```text
If the UI disappeared and came back, would this value still matter as app state?
```

If yes, ViewModel is more likely.

If no, composable is probably fine.

## 9. How UI events reach the ViewModel

Events do not magically find the ViewModel.

You connect them manually.

For example:

```kotlin
ResearchScreenContent(
    uiState = uiState,
    onSampleIdChange = viewModel::updateSampleId,
    onToggleConnection = viewModel::toggleConnection,
    onMeasure = viewModel::addMeasurement,
    onClear = viewModel::clearMeasurements
)
```

This means:

```text
When the UI needs to update the sample ID,
call viewModel.updateSampleId(...).

When the UI needs to measure,
call viewModel.addMeasurement().
```

Then inside the UI:

```kotlin
Button(
    onClick = onMeasure
) {
    Text("Measure")
}
```

The button does not know about the ViewModel directly.

It only knows:

```text
When clicked, call onMeasure.
```

That keeps the UI easier to preview and test.

## 10. Function reference syntax

This:

```kotlin
onMeasure = viewModel::addMeasurement
```

means:

```text
Pass the addMeasurement function itself.
Do not call it yet.
Let the button call it later.
```

This is different:

```kotlin
onMeasure = viewModel.addMeasurement()
```

That would call `addMeasurement()` immediately while building the UI.

Two correct versions are:

```kotlin
onMeasure = viewModel::addMeasurement
```

and:

```kotlin
onMeasure = {
    viewModel.addMeasurement()
}
```

Quick comparison:

| Code | Meaning |
|---|---|
| `viewModel.addMeasurement()` | call now |
| `viewModel::addMeasurement` | pass function for later |
| `{ viewModel.addMeasurement() }` | create a function that calls it later |

## 11. Getting the ViewModel in Compose

This line looks strange at first:

```kotlin
val viewModel: ResearchViewModel = viewModel()
```

There are three similar-looking names:

| Code | What it is |
|---|---|
| `ViewModel` | Android parent class |
| `ResearchViewModel` | your class |
| `viewModel()` | Compose helper function |

Read the line as:

```text
Create a variable named viewModel.
Its type is ResearchViewModel.
Get the object by calling the Compose viewModel() function.
```

It is like:

```kotlin
val viewModel: ResearchViewModel = getTheResearchViewModelForThisScreen()
```

This is not mainly an inheritance assignment example.

It is mainly a function-call assignment example:

```text
Function returns an object:
val thing: Thing = getThing()
```

The inheritance part only explains why `ResearchViewModel` belongs to Android's ViewModel system:

```kotlin
class ResearchViewModel : ViewModel()
```

That means:

```text
ResearchViewModel is a kind of ViewModel.
```

## 12. Why not ResearchViewModel() directly?

For a normal Kotlin class, this is common:

```kotlin
val helper = MeasurementHelper()
```

So this may look tempting:

```kotlin
val viewModel: ResearchViewModel = ResearchViewModel()
```

But inside Compose, we usually use:

```kotlin
val viewModel: ResearchViewModel = viewModel()
```

because:

```text
ResearchViewModel() creates a new object directly.
viewModel() asks Android/Compose for the correct ViewModel for this screen.
```

The `viewModel()` function can return an existing ViewModel if one already exists for the screen.

That helps the state survive recomposition and configuration changes such as screen rotation.

## 13. Updating lists in ViewModel state

In `ResearchUiState`, measurements are stored as:

```kotlin
val measurements: List<Measurement> = emptyList()
```

That is a read-only `List`, not a `MutableList`.

So we update it like this:

```kotlin
uiState = uiState.copy(
    measurements = uiState.measurements + newMeasurement
)
```

This means:

```text
Create a new list.
Old measurements plus one new measurement.
Put that new list into a new uiState object.
```

We do not write:

```kotlin
measurements = uiState.measurements.add(newMeasurement)
```

because:

```text
List does not have add().
MutableList.add() changes the existing list and returns Boolean, not the list.
```

Similarly:

```kotlin
measurements = emptyList()
```

means:

```text
Replace the old list with a new empty list.
```

It is not the same operation as:

```kotlin
measurementList.clear()
```

because `clear()` mutates an existing mutable list.

The ViewModel state style in this lesson prefers:

```text
old state -> new state
```

instead of:

```text
mutate the old object directly
```

## 14. CSV export: one useful boundary example

The export button has a mixed job.

It may need to:

```text
prepare CSV text
decide filename
open Android file picker
write to selected file
show export message
```

A simple first version can keep much of this in the composable so the whole flow is visible.

A cleaner version is:

```text
ViewModel prepares what to export.
Composable asks Android where to save it.
```

For example, the ViewModel can prepare:

```kotlin
data class CsvExportRequest(
    val filename: String,
    val csvText: String
)
```

and:

```kotlin
fun prepareCsvExport(): CsvExportRequest {
    val filename = if (uiState.sampleId.isNotBlank()) {
        "${safeFilename(uiState.sampleId)}_measurements.csv"
    } else {
        "measurements.csv"
    }

    return CsvExportRequest(
        filename = filename,
        csvText = measurementsToCsv(uiState.measurements)
    )
}
```

But this should stay in the composable:

```kotlin
createCsvLauncher.launch(exportRequest.filename)
```

because it opens the Android file picker.

## 15. Pure helper functions do not have to live in ViewModel

Some functions are not UI drawing, but they still do not need to be inside the ViewModel.

For example:

```kotlin
fun measurementsToCsv(
    measurements: List<Measurement>
): String
```

```kotlin
fun escapeCsv(value: String): String
```

```kotlin
fun safeFilename(text: String): String
```

These are pure helper functions.

They:

```text
take input
calculate or transform something
return output
```

They do not:

```text
read uiState directly
update uiState
draw UI
open Android system UI
need ViewModel lifecycle behavior
```

So they can stay outside the ViewModel.

The ViewModel can still call them:

```kotlin
fun getCsvText(): String {
    return measurementsToCsv(uiState.measurements)
}
```

This split is useful:

```text
getCsvText()
    -> belongs in ViewModel because it reads current screen state

measurementsToCsv(measurements)
    -> can stay outside because it only converts a list into text
```

A pure helper is easier to reuse and test because it does not need a ViewModel object.

Later, these helpers could move into a separate file:

```text
CsvExportUtils.kt
```

or into an object:

```kotlin
object CsvExporter {
    fun measurementsToCsv(measurements: List<Measurement>): String {
        ...
    }
}
```

The judging rule:

```text
If a function needs ViewModel state, put it in ViewModel.
If a function only transforms input into output, keep it as a helper.
```

## 16. What should not grow inside ViewModel forever?

The ViewModel should not become the whole app.  

If code becomes long because it is screen behavior, ViewModel is okay.

If code becomes long because it does specialized work, move that work to another class.

**A basic principle is: ViewModel only includes those functions that directly operates on UI state update. If it is a general function, not involving state operation, this function can be moved out of the ViewModel and ViewModel can call it when needed.**  
| Long code is about | Better place |
|---|---|
| changing screen state | ViewModel |
| validating this screen's user input | ViewModel |
| deciding screen messages | ViewModel |
| drawing UI | Composable |
| saving/loading files | repository or helper |
| database operations | repository |
| Bluetooth/USB/device protocol | device/source class |
| signal processing | processing class |
| ML inference | inference class/service |

Later, the shape becomes:

```text
Composable
    -> ViewModel
        -> Repository
            -> database / files / device / processing / ML
```

The ViewModel coordinates the screen.

It should not personally do every low-level job.

## 17. A practical checklist to decide

When deciding where code belongs, ask these questions.

Question 1:

```text
Is this drawing UI?
```

If yes, put it in the composable.

Question 2:

```text
Is this reacting to a user action and changing screen state?
```

If yes, put it in the ViewModel.

Question 3:

```text
Is this state meaningful for the screen?
```

If yes, put it in `ResearchUiState` and let the ViewModel own it.

Question 4:

```text
Is this temporary state only needed for a UI API?
```

If yes, it can stay in the composable.

Question 5:

```text
Is this database, file, hardware, signal-processing, or ML detail?
```

If yes, it probably belongs in another class, and the ViewModel should call that class.

## 18. ViewModel and the application layer

ViewModel is close to what people sometimes call the application layer.

But it is important to say this carefully:

```text
ViewModel is not the whole application layer.
ViewModel is the screen-level application logic/state holder.
```

It answers screen workflow questions such as:

```text
What should happen when the user clicks Measure?
Can measurement start right now?
What state should the screen show next?
Should we ask another layer for data?
Should we show an error message?
```

These are not drawing questions.

They are application behavior questions.

That is why ViewModel feels more like app logic than UI drawing.

For the current lessons, this model is enough:

```text
Composable
    -> draws UI

ViewModel
    -> screen state
    -> screen behavior
    -> screen-level application logic

Repository/helper/device/ML classes
    -> data, storage, hardware, processing, model details
```

Later, a larger app may split the middle more:

```text
Composable
    -> ViewModel
        -> UseCase / Interactor
            -> Repository
                -> Database / Device / ML
```

In that larger structure, the ViewModel becomes thinner.

It mostly connects the screen to use cases.

So the beginner mental model is:

```text
ViewModel = screen-level application logic
```

not:

```text
ViewModel = all application logic
```

## 19. The main flow to remember

This is the core ViewModel architecture:

```text
ViewModel owns uiState.

UI reads uiState.

UI event happens.

UI calls a ViewModel function.

ViewModel updates uiState.

Compose observes the state change.

UI redraws with the new state.
```

In code:

```kotlin
val uiState = viewModel.uiState
```

```kotlin
Button(
    onClick = viewModel::addMeasurement
) {
    Text("Measure")
}
```

```kotlin
fun addMeasurement() {
    uiState = uiState.copy(
        measurements = uiState.measurements + newMeasurement
    )
}
```

The most important sentence:

```text
The UI does not directly change the screen state.
The UI tells the ViewModel what happened.
The ViewModel decides how the state changes.
```

## 20. Short summary

If you remember only this, you are doing well:

```text
Composable:
shows UI
keeps temporary UI-only state
calls ViewModel functions

ViewModel:
holds screen state
updates screen state
handles screen-level logic
calls other layers when needed

Repository/helper/device/ML classes:
handle specialized data, storage, hardware, processing, or model work
```

ViewModel is not magic syntax.

It is a regular-looking Kotlin class with an Android-managed lifetime and a clear job:

```text
be the state holder and decision layer for a screen.
```
