# Lesson 8 Notes - Compose Observable State

This note explains an important idea that appears from Lesson 4 to Lesson 8:

```text
mutableStateOf creates Compose-observable state.
```

This means:

```text
If a composable reads the state,
and the state later changes,
Compose knows that the UI may need to update.
```

But `mutableStateOf` is only one part of the story.

You also need to understand where the state lives:

- inside a composable with `remember`
- inside a `ViewModel`
- inside a Compose-aware list such as `mutableStateListOf`

This note is organized in this order:

1. Understand the core rule: UI is driven by observable state.
2. Compare ordinary variables with `mutableStateOf`.
3. Learn what `by` does and what `remember` adds.
4. Compare state stored in a composable, a `ViewModel`, and a state list.
5. Follow the lesson progression from Lesson 4 to Lesson 8.
6. Understand `copy()`, `private set`, state down, and events up.
7. Separate real state from calculated `val` values.

## 1. The Core Idea

In Compose, the UI is driven by state.

This means:

```text
State says what is true right now.
The UI displays what is true right now.
When state changes, Compose updates the UI.
```

For example:

```kotlin
var measurementValue by remember {
    mutableStateOf<Double?>(null)
}
```

This creates state for the latest measurement value.

Then the UI reads it:

```kotlin
Text(
    text = measurementValue?.toString() ?: "No measurement yet"
)
```

Then an event changes it:

```kotlin
measurementValue = Random.nextDouble(0.0, 5.0)
```

The flow is:

```text
state changes
    -> Compose notices
    -> UI recomposes
    -> Text shows the new value
```

## 2. Ordinary variables are not Compose state

This is only a normal Kotlin variable:

```kotlin
var measurementValue = 0.0
```

If you change it:

```kotlin
measurementValue = 2.43
```

Compose does not automatically know that the UI should update.

For Compose UI state, use:

```kotlin
var measurementValue by remember {
    mutableStateOf(0.0)
}
```

The important part is:

```kotlin
mutableStateOf(0.0)
```

That creates observable state.

## 3. What mutableStateOf means

This:

```kotlin
mutableStateOf(0.0)
```

means:

```text
Create a state holder.
Store the value 0.0 inside it.
Let Compose observe it.
When the value changes, tell Compose.
```

Without the `by` shortcut, it looks like this:

```kotlin
val measurementValueState = mutableStateOf(0.0)
```

Then you read the value with:

```kotlin
measurementValueState.value
```

And you change the value with:

```kotlin
measurementValueState.value = 2.43
```

With `by`, you can write it in a simpler way:

```kotlin
var measurementValue by mutableStateOf(0.0)
```

Then you read it like a normal variable:

```kotlin
measurementValue
```

And change it like a normal variable:

```kotlin
measurementValue = 2.43
```

## 4. What by means

The `by` keyword means delegated property.

For beginner Compose, the practical meaning is:

```text
by lets you use Compose state like a normal Kotlin variable.
```

Instead of:

```kotlin
measurementValueState.value = 2.43
```

you can write:

```kotlin
measurementValue = 2.43
```

That is why the lessons use this pattern:

```kotlin
var sampleId by remember {
    mutableStateOf("")
}
```

and later:

```kotlin
var uiState by mutableStateOf(ResearchUiState())
    private set
```

In both cases, `by` lets you read and assign the state variable directly.

## 5. What remember adds

Inside a composable, code can rerun during recomposition.

So if you wrote:

```kotlin
@Composable
fun ResearchScreen() {
    var sampleId by mutableStateOf("")
}
```

the state object could be recreated when the composable reruns.

That is why Lesson 4 uses:

```kotlin
@Composable
fun ResearchScreen() {
    var sampleId by remember {
        mutableStateOf("")
    }
}
```

This means:

```text
mutableStateOf
    -> make this observable state

remember
    -> keep this same state object across recompositions
```

So the full meaning is:

```text
Create Compose-observable state.
Remember it while this composable stays in the composition.
Use it like a normal variable because of by.
```

## 6. Why ViewModel state does not need remember

In Lesson 8, the state moves into the `ViewModel`:

```kotlin
class ResearchViewModel : ViewModel() {
    var uiState by mutableStateOf(ResearchUiState())
        private set
}
```

There is no `remember` here.

Why?

Because this code is not inside a composable function.

The `ViewModel` itself is the stable state holder.

So:

```text
Inside a composable:
    use remember { mutableStateOf(...) }

Inside a ViewModel:
    use mutableStateOf(...)
```

The ViewModel survives recomposition, so it does not need `remember` to survive recomposition.

## 7. Same observable state, different storage place

These two examples both create Compose-observable state:

```kotlin
var sampleId by remember {
    mutableStateOf("")
}
```

and:

```kotlin
var uiState by mutableStateOf(ResearchUiState())
    private set
```

The difference is where the state lives.

| Code | Where it lives | Why |
|---|---|---|
| `remember { mutableStateOf("") }` | Inside a composable | `remember` keeps it across recompositions |
| `mutableStateOf(ResearchUiState())` | Inside a ViewModel | The ViewModel keeps it across recompositions |

So the corrected mental model is:

```text
mutableStateOf = observable by Compose
remember = survive recomposition inside a composable
ViewModel = survive outside the composable screen
```

## 8. Compose only updates UI that reads the state

`mutableStateOf` does not always update every composable.

It matters only at which UI the state is read.

For example:

```kotlin
Text(
    text = uiState.sampleId
)
```

This `Text` reads `uiState.sampleId`.

If `uiState.sampleId` changes, Compose knows this part of the UI may need to update.

Another example:

```kotlin
Button(
    enabled = uiState.sampleId.isNotBlank() && uiState.isConnected,
    onClick = onMeasure
) {
    Text("Measure")
}
```

This button reads:

```text
uiState.sampleId
uiState.isConnected
```

So if either value changes, Compose may need to recompose the button.

## 9. The Lesson 4 pattern: state inside the screen

In Lesson 4, the app used state directly inside `ResearchScreen`:

```kotlin
@Composable
fun ResearchScreen() {
    var sampleId by remember {
        mutableStateOf("")
    }

    var measurementValue by remember {
        mutableStateOf<Double?>(null)
    }

    var measurementCount by remember {
        mutableStateOf(0)
    }
}
```

This is good for a beginner screen.

The screen owns the state.

The screen also displays the state.

The screen also changes the state.

The flow is:

```text
ResearchScreen owns state
    -> UI displays state
    -> button/text field changes state
    -> ResearchScreen recomposes
```

## 10. The Lesson 6 pattern: Compose state list

In Lesson 6, the app needed a list of measurements.

A normal Kotlin list like this:

```kotlin
val measurements = mutableListOf<Measurement>()
```

can store data, but Compose may not notice when you add an item.

So Lesson 6 uses:

```kotlin
val measurements = remember {
    mutableStateListOf<Measurement>()
}
```

This means:

```text
Create a Compose-aware mutable list.
Remember the list across recompositions.
When items are added or removed, Compose can update the UI.
```

That is why this updates the UI:

```kotlin
measurements.add(newMeasurement)
```

and this also updates the UI:

```kotlin
measurements.clear()
```

## 11. Why the list uses val

This can feel confusing:

```kotlin
val measurements = remember {
    mutableStateListOf<Measurement>()
}
```

Why `val` if the list changes?

Because you are not replacing the list variable.

You are changing the contents inside the same list.

This is allowed:

```kotlin
measurements.add(newMeasurement)
```

This would not be allowed:

```kotlin
measurements = mutableStateListOf()
```

So:

```text
val measurements
    -> the list object itself stays the same
    -> the contents of the list can change
```

## 12. The Lesson 7 pattern: temporary export state

In Lesson 7, the app added export-related state:

```kotlin
var pendingCsvText by remember {
    mutableStateOf("")
}

var exportMessage by remember {
    mutableStateOf("")
}
```

These are still Compose-observable state values.

`pendingCsvText` stores CSV text temporarily while the Android file picker is open.

`exportMessage` stores a message to show the user:

```kotlin
exportMessage = "CSV exported successfully."
```

Then the UI can read it:

```kotlin
if (exportMessage.isNotBlank()) {
    Text(exportMessage)
}
```

So the pattern is the same:

```text
state changes
    -> UI that reads it updates
```

## 13. The Lesson 8 pattern: one UI state object

By Lesson 8, there are many separate state variables:

```text
sampleId
isConnected
measurements
exportMessage
```

So Lesson 8 creates one data class:

```kotlin
data class ResearchUiState(
    val sampleId: String = "",
    val isConnected: Boolean = false,
    val measurements: List<Measurement> = emptyList(),
    val exportMessage: String = ""
)
```

Then the ViewModel stores one observable state object:

```kotlin
class ResearchViewModel : ViewModel() {
    var uiState by mutableStateOf(ResearchUiState())
        private set
}
```

This means:

```text
uiState is Compose-observable.
The UI can read uiState.
When uiState changes, Compose can update the UI.
```

## 14. Why Lesson 8 uses copy()

`ResearchUiState` is a data class whose properties are mostly `val`.

So we do not usually mutate one property directly.

This is the important two-level idea:

```kotlin
var uiState by mutableStateOf(ResearchUiState())
```

means:

```text
uiState is var
    -> the whole ResearchUiState object can be replaced
```

But inside the data class, the fields are declared with `val`:

```kotlin
data class ResearchUiState(
    val sampleId: String = "",
    val isConnected: Boolean = false,
    val measurements: List<Measurement> = emptyList(),
    val exportMessage: String = ""
)
```

That means:

```text
sampleId is val
isConnected is val
measurements is val
exportMessage is val
    -> these fields cannot be changed directly on the existing object
```

So this is not allowed:

```kotlin
uiState.sampleId = "S001"
```

You may wonder why we do not just make the properties `var`:

```kotlin
data class ResearchUiState(
    var sampleId: String = "",
    var isConnected: Boolean = false,
    var measurements: List<Measurement> = emptyList(),
    var exportMessage: String = ""
)
```

Then this would look natural:

```kotlin
uiState.sampleId = "S001"
```

But for Compose UI state, the preferred pattern is to treat the whole state object as an immutable snapshot.

Snapshot means:

```text
This ResearchUiState object describes the whole screen at one moment.
```

When something changes, create the next snapshot instead of editing the old one:

```text
old uiState
    -> copy with one changed field
    -> new uiState
```

This makes state updates easier to reason about and easier for Compose to observe clearly.

Instead, we create a new state object:

```kotlin
uiState = uiState.copy(
    sampleId = newSampleId
)
```

This means:

```text
Take the old uiState.
Copy all the old values.
Replace only sampleId.
Assign the new object back to uiState.
```

Because `uiState` is created with `mutableStateOf`, assigning a new value tells Compose that state changed.

Short version:

```text
uiState is var, so replace the whole state object.
ResearchUiState fields are val, so use copy() to change one field.
```

## 15. Why private set is used

Lesson 8 uses:

```kotlin
var uiState by mutableStateOf(ResearchUiState())
    private set
```

This means:

```text
Other code can read uiState.
Only the ViewModel can replace uiState.
```

So the UI can do this:

```kotlin
val uiState = viewModel.uiState
```

But the UI should not do this:

```kotlin
viewModel.uiState = ResearchUiState()
```

Instead, the UI sends events:

```kotlin
onValueChange = viewModel::updateSampleId
onClick = viewModel::addMeasurement
```

Then the ViewModel updates the state:

```kotlin
fun updateSampleId(newSampleId: String) {
    uiState = uiState.copy(
        sampleId = newSampleId
    )
}
```

## 16. State down, events up

Lesson 8 introduces a cleaner architecture:

```text
State goes down to the UI.
Events go up to the ViewModel.
```

The screen receives state:

```kotlin
ResearchScreenContent(
    uiState = uiState,
    onSampleIdChange = viewModel::updateSampleId,
    onMeasure = viewModel::addMeasurement
)
```

The UI reads the state:

```kotlin
OutlinedTextField(
    value = uiState.sampleId,
    onValueChange = onSampleIdChange
)
```

The UI sends an event:

```kotlin
onValueChange = onSampleIdChange
```

The ViewModel handles the event:

```kotlin
fun updateSampleId(newSampleId: String) {
    uiState = uiState.copy(sampleId = newSampleId)
}
```

Then Compose updates the UI that reads `uiState`.

## 17. Calculated val values are different from state

This is not state:

```kotlin
val latestMeasurement = uiState.measurements.lastOrNull()
```

This is a calculated value.

It is calculated from state.

Same with:

```kotlin
val values = uiState.measurements.map {
    it.value
}

val meanText = if (values.isNotEmpty()) {
    "%.3f".format(values.average())
} else {
    "--"
}
```

These values do not need `mutableStateOf`.

They are recalculated when the composable recomposes.

So:

```text
uiState.measurements
    -> state

latestMeasurement, values, meanText
    -> calculated from state
```

## 18. Quick decision rule

Ask:

```text
Does changing this value need to update the UI?
```

If yes, it probably needs to be Compose-observable state.

Examples:

```kotlin
mutableStateOf("")
mutableStateOf(false)
mutableStateOf<Double?>(null)
mutableStateListOf<Measurement>()
```

Ask:

```text
Is this state stored inside a composable?
```

If yes, use `remember`:

```kotlin
var sampleId by remember {
    mutableStateOf("")
}
```

Ask:

```text
Is this state stored inside a ViewModel?
```

If yes, `remember` is not needed:

```kotlin
var uiState by mutableStateOf(ResearchUiState())
    private set
```

Ask:

```text
Is this just calculated from existing state?
```

If yes, use a normal `val`:

```kotlin
val latestMeasurement = uiState.measurements.lastOrNull()
```

## 19. What to remember

The most important ideas are:

- `mutableStateOf` creates Compose-observable state.
- `by` lets you use that state like a normal variable instead of writing `.value`.
- `remember` keeps state from being recreated during recomposition inside a composable.
- A `ViewModel` can hold Compose state without `remember`.
- `mutableStateListOf` is a Compose-aware mutable list.
- Calculated values such as `meanText` usually do not need to be state.
- `private set` lets the UI read state but prevents the UI from replacing it directly.
- The clean architecture pattern is state down, events up.

Final mental model:

```text
mutableStateOf
    -> Compose can observe changes

remember
    -> keep local composable state across recomposition

ViewModel
    -> keep screen state outside the composable

UI reads state
    -> events call ViewModel functions
    -> ViewModel changes state
    -> Compose updates UI
```
