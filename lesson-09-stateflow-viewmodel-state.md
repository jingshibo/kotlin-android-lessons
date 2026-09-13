# Lesson 9 - StateFlow ViewModel State

Before comparing the two ViewModel state styles, remember the state object itself.

In Lesson 8, the screen state was grouped into one data class:

```kotlin
data class ResearchUiState(
    val sampleId: String = "",
    val isConnected: Boolean = false,
    val measurements: List<Measurement> = emptyList(),
    val exportMessage: String = ""
)
```

And each measurement row used this data class:

```kotlin
data class Measurement(
    val sampleId: String,
    val repetition: Int,
    val value: Double,
    val timestamp: Long
)
```

So when you see:

```kotlin
ResearchUiState()
```

read it as:

```text
Create the first/default screen state.
```

In Lesson 8, the app used this ViewModel state style:

```kotlin
var uiState by mutableStateOf(ResearchUiState())
    private set
```

The screen used that state like this:

```kotlin
@Composable
fun ResearchRoute(
    viewModel: ResearchViewModel = viewModel()
) {
    val uiState = viewModel.uiState

    ResearchScreenContent(
        uiState = uiState,
        onSampleIdChange = viewModel::updateSampleId,
        onConnectClick = viewModel::toggleConnection,
        onMeasureClick = viewModel::addMeasurement,
        onClearClick = viewModel::clearMeasurements
    )
}
```

And the ViewModel changed that state like this:

```kotlin
fun updateSampleId(newSampleId: String) {
    uiState = uiState.copy(
        sampleId = newSampleId,
        exportMessage = ""
    )
}
```

So the full old pattern was:

```text
ViewModel owns uiState.
Screen reads viewModel.uiState.
Screen calls ViewModel functions.
ViewModel replaces uiState with uiState.copy(...).
Compose notices and redraws.
```

That is a valid Compose-friendly state pattern.

But in many Android projects, you will see another style:

```kotlin
private val _uiState = MutableStateFlow(ResearchUiState())
val uiState: StateFlow<ResearchUiState> = _uiState.asStateFlow()
```

Then the screen reads it with:

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

This lesson explains that implementation style.

The important idea:

```text
StateFlow does not change the architecture.
It changes how the ViewModel stores and exposes state.
```

The same architecture still exists:

```text
Screen
-> ViewModel
-> data work later
```

---

## 1. Why this lesson exists

You may see code like this in other Android examples:

```kotlin
val state by viewModel.uiState.collectAsState()
```

or:

```kotlin
_uiState.update { currentState ->
    currentState.copy(
        sampleId = "S001"
    )
}
```

This can look like a different architecture.

It is not.

It is a different state holder.

In the earlier tutorial style, the ViewModel state is held by Compose state:

```kotlin
mutableStateOf
```

In the StateFlow style, the ViewModel state is held by Kotlin Flow:

```kotlin
MutableStateFlow
```

Both patterns can support the same app idea:

```text
UI reads state.
UI sends events.
ViewModel updates state.
UI redraws.
```

---

## 2. What Flow means

Before learning `StateFlow`, it helps to understand the word `Flow`.

In Kotlin, a `Flow` is:

```text
a stream of values over time.
```

A normal variable usually gives you one value:

```kotlin
val sampleId = "S001"
```

A `Flow` can give you many values over time:

```text
"S001"
"S002"
"S003"
```

For example, imagine the selected sample changes while the app is running:

```text
first value:  no sample selected
next value:   S001
next value:   S002
next value:   S003
```

That is a flow of values.

The code that listens to a Flow is said to:

```text
collect the Flow.
```

In general coroutine code, collecting can look like this:

```kotlin
viewModel.uiState.collect { latestState ->
    // react to latestState
}
```

means:

```text
Start listening to uiState.
Every time uiState emits a new ResearchUiState,
put that new value into latestState,
then run the code inside the block.
```

But this raw `collect { ... }` example is only here to explain the word `collect`.

In the Compose screen for this lesson, we normally do not write raw `collect { ... }`.

Instead, the screen uses:

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

That is the Compose-friendly way to collect ViewModel state.

For ViewModel state, this is useful because the screen state changes many times:

```text
sample ID changes
connection state changes
new measurement is added
export message changes
loading state changes
```

So a Flow is not a new app layer.

It is a way to represent:

```text
values that can change over time.
```

### Flow versus StateFlow

`StateFlow` is a special kind of Flow.

A regular `Flow` is mainly:

```text
values over time.
```

A `StateFlow` is:

```text
current value + future values over time.
```

That current value is important for UI state.

The UI needs to know:

```text
What should I show right now?
```

So this:

```kotlin
val uiState: StateFlow<ResearchUiState>
```

means:

```text
uiState has the current ResearchUiState.
uiState can also emit new ResearchUiState values later.
```

For this lesson, the simple mental model is:

```text
Flow = values over time
StateFlow = current state + future state updates
collect = listen to those updates
```

---

## 3. Previous ViewModel style

The previous tutorial style looked like this:

```kotlin
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.setValue
import androidx.lifecycle.ViewModel

class ResearchViewModel : ViewModel() {

    var uiState by mutableStateOf(ResearchUiState())
        private set

    fun updateSampleId(newSampleId: String) {
        uiState = uiState.copy(
            sampleId = newSampleId,
            exportMessage = ""
        )
    }
}
```

The screen could read the state directly:

```kotlin
@Composable
fun ResearchRoute(
    viewModel: ResearchViewModel = viewModel()
) {
    val uiState = viewModel.uiState

    ResearchScreenContent(
        uiState = uiState,
        onSampleIdChange = viewModel::updateSampleId,
        onConnectClick = viewModel::toggleConnection,
        onMeasureClick = viewModel::addMeasurement,
        onClearClick = viewModel::clearMeasurements
    )
}
```

This works because `mutableStateOf` is observable by Compose.

When the ViewModel assigns:

```kotlin
uiState = uiState.copy(...)
```

Compose notices and redraws the composable that read `uiState`.

---

## 4. StateFlow ViewModel style

The StateFlow version looks like this:

```kotlin
import androidx.lifecycle.ViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update

class ResearchViewModel : ViewModel() {

    private val _uiState = MutableStateFlow(ResearchUiState()) // visible to ViewModel only

    val uiState: StateFlow<ResearchUiState> = // exposed outside for reading only
        _uiState.asStateFlow()

    fun updateSampleId(newSampleId: String) {
        _uiState.update { currentState ->
            currentState.copy(
                sampleId = newSampleId,
                exportMessage = ""
            )
        }
    }
}
```

There are now two properties:

```text
_uiState
uiState
```

They are intentionally different.

| Property | Who sees it | Can it be changed? |
|---|---|---|
| `_uiState` | ViewModel only | Yes |
| `uiState` | UI and other outside code | No, read-only |

This is the StateFlow version of:

```kotlin
private set
```

The ViewModel can update the state.

The UI can observe the state but cannot directly replace the state.

Another way to read it:

```text
_uiState is the private mutable version.
uiState is the public read-only version.
```

Only the ViewModel should use `_uiState`.

The UI should only use `uiState`.

So this:

```kotlin
private val _uiState = MutableStateFlow(...)
val uiState = _uiState.asStateFlow()
```

has the same protective purpose as:

```kotlin
var uiState by mutableStateOf(...)
    private set
```

Both mean:

```text
The UI can read state.
Only the ViewModel can change state.
```

The screen should call:

```kotlin
viewModel.updateSampleId("S001")
```

It should not do:

```kotlin
viewModel._uiState.value = ...
```

In fact, it cannot do that because `_uiState` is private.

---

## 5. What update means

This code:

```kotlin
_uiState.update { currentState ->
    currentState.copy(
        sampleId = "S001"
    )
}
```

means:

```text
Take the current state.
Create a new state object from it.
Change only sampleId.
Publish the new state.
```

It is the StateFlow version of:

```kotlin
uiState = uiState.copy(
    sampleId = "S001"
)
```

The word `currentState` is just a parameter name.

You may also see:

```kotlin
_uiState.update { state ->
    state.copy(
        sampleId = "S001"
    )
}
```

or:

```kotlin
_uiState.update {
    it.copy(
        sampleId = "S001"
    )
}
```

For learning, `currentState` is clearer.

---

## 6. Reading StateFlow in Compose

With the previous `mutableStateOf` ViewModel style, the screen could do:

```kotlin
val uiState = viewModel.uiState
```

With StateFlow, the screen must collect the flow:

```kotlin
import androidx.compose.runtime.getValue
import androidx.lifecycle.compose.collectAsStateWithLifecycle

@Composable
fun ResearchRoute(
    viewModel: ResearchViewModel = viewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    ResearchScreenContent(
        uiState = uiState,
        onSampleIdChange = viewModel::updateSampleId,
        onConnectClick = viewModel::toggleConnection,
        onMeasureClick = viewModel::addMeasurement,
        onClearClick = viewModel::clearMeasurements
    )
}
```

This line:

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

means:

```text
Observe the ViewModel's StateFlow.
Convert its latest value into Compose State.
Redraw this composable when the value changes.
Respect the Android lifecycle while collecting.
```

You may also see:

```kotlin
val uiState by viewModel.uiState.collectAsState()
```

That also converts a `StateFlow` into Compose-readable state.

For Android screens, `collectAsStateWithLifecycle()` is usually preferred because it is lifecycle-aware.

---

## 7. Dependency and imports

For `StateFlow` itself, you need Kotlin coroutines Flow imports:

```kotlin
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update
```

For lifecycle-aware Compose collection, use:

```kotlin
import androidx.lifecycle.compose.collectAsStateWithLifecycle
```

If Android Studio cannot find that import, add the lifecycle runtime Compose dependency:

```kotlin
dependencies {
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.10.0")
}
```

Your project may use a different lifecycle version.

Use the same lifecycle version family already used by your project when possible.

---

## 8. Reading current state inside the ViewModel

With the old style, the ViewModel could read:

```kotlin
val sampleId = uiState.sampleId
```

With StateFlow, `uiState` is now a `StateFlow`.

So inside the ViewModel, read the current value like this:

```kotlin
val sampleId = _uiState.value.sampleId
```

or:

```kotlin
val currentState = _uiState.value
val sampleId = currentState.sampleId
```

For example:

```kotlin
import kotlin.random.Random

fun addMeasurement() {
    val currentState = _uiState.value
    val sampleId = currentState.sampleId

    if (sampleId.isBlank() || !currentState.isConnected) {
        _uiState.update { currentState ->
            // This currentState is a new lambda parameter from update.
            // It is not the same variable as val currentState = _uiState.value and !currentState.isConnected above.
            currentState.copy(
                exportMessage = "Enter a sample ID and connect first"
            )
        }
        return
    }

    val newMeasurement = Measurement(
        sampleId = sampleId,
        repetition = currentState.measurements.count {
            it.sampleId == sampleId
        } + 1,
        value = Random.nextDouble(0.0, 5.0),
        timestamp = System.currentTimeMillis()
    )

    _uiState.update { latestState ->
        latestState.copy(
            measurements = latestState.measurements + newMeasurement,
            exportMessage = ""
        )
    }
}
```

The shape is the same as before.

Only the state syntax changed.

---

## 9. Code comparison

Here is the same operation in both styles.

Previous style:

```kotlin
var uiState by mutableStateOf(ResearchUiState())
    private set

fun updateSampleId(newSampleId: String) {
    uiState = uiState.copy(
        sampleId = newSampleId,
        exportMessage = ""
    )
}
```

StateFlow style:

```kotlin
private val _uiState = MutableStateFlow(ResearchUiState())
val uiState: StateFlow<ResearchUiState> = _uiState.asStateFlow()

fun updateSampleId(newSampleId: String) {
    _uiState.update { currentState ->
        currentState.copy(
            sampleId = newSampleId,
            exportMessage = ""
        )
    }
}
```

Reading current state inside the ViewModel. Previous style:

```kotlin
val sampleId = uiState.sampleId
```

Reading current state inside the ViewModel. StateFlow style:

```kotlin
val sampleId = _uiState.value.sampleId
```

Screen reading, previous style:

```kotlin
val uiState = viewModel.uiState
```

Screen reading, StateFlow style:

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```


### Tiny translation guide

| Previous Lesson 8 code | StateFlow version |
|---|---|
| `var uiState by mutableStateOf(...)` | `private val _uiState = MutableStateFlow(...)` |
| `private set` | expose `val uiState = _uiState.asStateFlow()` |
| `uiState = uiState.copy(...)` | `_uiState.update { it.copy(...) }` |
| `val uiState = viewModel.uiState` | `val uiState by viewModel.uiState.collectAsStateWithLifecycle()` |
| `uiState.sampleId` inside ViewModel | `_uiState.value.sampleId` inside ViewModel |

Same mental model:

```text
old:
ViewModel updates uiState
-> Compose observes mutableStateOf
-> UI redraws

new:
ViewModel updates private _uiState 
-> StateFlow exposes this _uiState value via read-only uiState to Compose
-> Compose collects viewModel.uiState
-> UI redraws based on updated uiState
```

---

## 10. Pros and cons

| Style | Pros | Cons |
|---|---|---|
| `mutableStateOf` in ViewModel | Shorter code, beginner-friendly, works naturally with Compose | More Compose-specific, less common in non-Compose layers, less natural when combining multiple flows |
| `MutableStateFlow` in ViewModel | Common in modern Android apps, works well with coroutines, easier to combine with repository flows, lifecycle-aware collection available | More syntax, needs `collectAsState...` in UI, beginner confusion around `_uiState`, `.value`, and `update` |

For this tutorial, the earlier style was chosen because:

```text
it teaches the architecture with less syntax.
```

StateFlow is useful once the app starts to have:

```text
Room data streams
repository functions returning Flow
background loading
multiple screens observing shared state
larger ViewModels
tests that observe state changes
```

For example, later your app may want to observe patients from Room:

```kotlin
repository.observePatients()
```

If that returns a Flow, then a StateFlow-based ViewModel fits naturally.

The data can flow like this:

```text
Room Flow
-> Repository Flow
-> ViewModel StateFlow
-> Compose collectAsStateWithLifecycle()
-> UI
```

---

## 11. Is StateFlow a different layer?

No.

StateFlow is not:

```text
a repository
a database
a ViewModel replacement
a new application layer
```

StateFlow is:

```text
a state stream.
```

In this app, it lives inside the ViewModel:

```text
ResearchViewModel
    -> owns MutableStateFlow
    -> exposes StateFlow
    -> updates state through functions
```

The repository layer does not become the ViewModel.

The ViewModel does not become the repository.

Only the state holder changes.

---

## 12. Common mistakes

### Mistake 1: Exposing MutableStateFlow directly

Avoid:

```kotlin
val uiState = MutableStateFlow(ResearchUiState())
```

This lets outside code change the state directly.

Prefer:

```kotlin
private val _uiState = MutableStateFlow(ResearchUiState())
val uiState: StateFlow<ResearchUiState> = _uiState.asStateFlow()
```

### Mistake 2: Forgetting to collect in Compose

This is not enough:

```kotlin
val uiState = viewModel.uiState
```

That gives you the `StateFlow` object, not the `ResearchUiState` value.

Use:

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

### Mistake 3: Mutating a list inside state

Avoid keeping a mutable list and changing it in place.

Prefer creating a new list:

```kotlin
_uiState.update { currentState ->
    currentState.copy(
        measurements = currentState.measurements + newMeasurement
    )
}
```

This keeps state updates predictable.

### Mistake 4: Thinking update changes one field in place

This:

```kotlin
currentState.copy(...)
```

does not mutate the old object.

It creates a new `ResearchUiState` object with selected fields changed.

That is the same idea used in the earlier lessons.

---

## 13. Which one should you use?

For early learning:

```text
mutableStateOf in ViewModel is easier.
```

For modern production-style Android:

```text
StateFlow is more common and more flexible.
```

The important part is not memorizing one syntax.

The important part is keeping the architecture clean:

```text
Screen displays state.
Screen sends events.
ViewModel owns screen state.
ViewModel handles screen actions.
Later, the ViewModel can call a repository for lower-level work.
```

If you understand that, both implementations make sense.

---

## 14. Final mental model

Previous lesson style:

```text
ViewModel has Compose-observable state.
```

StateFlow style:

```text
ViewModel has Flow-observable state.
Compose collects it and turns it into Compose state.
```

We can also think of it as:

```text
ViewModel has coroutine-observable state.
```

That wording may be a little unclear.

What it means is:

```text
The ViewModel state is stored in a Kotlin Flow object, so coroutine/Flow code can observe changes to it.
```

More specifically, `StateFlow` belongs to Kotlin coroutines/Flow APIs:

```kotlin
private val _uiState = MutableStateFlow(ResearchUiState())
val uiState: StateFlow<ResearchUiState> = _uiState.asStateFlow()
```

To collect the value, in a non-Compose coroutine code, it is like this:

```kotlin
viewModel.uiState.collect { latestState ->
    // react to new state
}
```

But this is not the main screen pattern in this lesson.

In Compose, we usually collect it like this:

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

So the meaning is:

```text
mutableStateOf style:
ViewModel has Compose-observable state.

StateFlow style:
ViewModel has Flow-observable state.
Compose collects that Flow and turns it into Compose state.
```

So when you see:

```kotlin
val state by viewModel.uiState.collectAsState()
```

read it as:

```text
The UI is subscribing to ViewModel state.
```

And when you see:

```kotlin
_uiState.update { currentState ->
    currentState.copy(...)
}
```

read it as:

```text
The ViewModel is publishing a new screen state.
```

Final mental model:

```text
StateFlow is a more advanced state holder; the state is still ViewModel state.
```

Same architecture.

More scalable state tool.
