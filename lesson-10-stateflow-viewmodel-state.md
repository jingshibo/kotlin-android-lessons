# Lesson 10 - StateFlow ViewModel State

Before comparing the two ViewModel state styles, remember the state object itself.

In Lesson 9, the screen state was grouped into one data class:

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

In Lesson 9, the app used this ViewModel state style:

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

è¿™é‡Œcollectçš„å«ä¹‰å¾ˆç›´æŽ¥ï¼Œå°±æ˜¯èŽ·å– viewModel.uiStateçš„å€¼ï¼Œè¿™é‡Œçš„latestStateæŒ‡çš„å°±æ˜¯uiStateçš„å½“å‰å€¼ã€‚ç„¶åŽåŸºäºŽèŽ·å–çš„è¿™ä¸ªå€¼è¿›è¡ŒåŽç»­æ“ä½œã€‚

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

    // 1. PRIVATE & MUTABLE (Editable)
    // The '_' prefix means private. ONLY DataViewModel can change this!
    private val _uiState = MutableStateFlow(ResearchUiState()) // visible to ViewModel only

    // 2. PUBLIC & IMMUTABLE (Read-Only)
    // Exposed to the UI screen. The UI can ONLY READ this stream!
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

### What update means

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

## 5. Reading current state inside the ViewModel

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

Why `collectAsState()` is essential:

StateFlow belongs to Kotlin, but Compose is a UI framework. Compose doesn't know how to read Kotlin StateFlow directly.

`collectAsState()` acts as the bridge:
1. It subscribes to `viewModel.uiState`.
2. Every time a new ResearchUiState is produced, `collectAsState()` notifies Compose.
3. Compose detects the change and automatically re-draws (recomposes) the exact UI composables that read state.

For Android screens, `collectAsStateWithLifecycle()` is usually preferred because it is lifecycle-aware.

Differences: `collectAsState()` vs. `collectAsStateWithLifecycle()`

| Feature | `collectAsState()` | `collectAsStateWithLifecycle()` (Best Practice) |
|---|---|---|
| Lifecycle Awareness | âŒ None (Keeps collecting in background) | âœ… Lifecycle-Aware (Pauses when app is minimized) |
| Battery & CPU Usage | Wastes CPU/battery updating UI state when app is hidden | Saves CPU & Battery by pausing background flow collection |
| Behavior on Minimize | Keeps collecting flow emissions when screen is off | Pauses collection when lifecycle falls below `STARTED`, resumes on foreground |
| Library Origin | `androidx.compose.runtime` (Base Compose) | `androidx.lifecycle.compose` (Android Lifecycle) |

So in a normal Android screen, prefer:

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

Use `collectAsState()` only when lifecycle awareness is not needed, or when you are not inside a normal Android lifecycle-aware screen.


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

## 8. Code comparison

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

| Previous Lesson 9 code | StateFlow version |
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

### Pros and cons

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

---

## 9. StateFlow outside Compose

The examples above show `StateFlow` in a Compose screen:

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

That line is a Compose convenience.

It subscribes to the `StateFlow`, converts the latest value into Compose `State`, and lets Compose redraw the UI when the value changes.

But `StateFlow` itself does not belong to Compose.

`StateFlow` is part of Kotlin coroutines Flow APIs:

```kotlin
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
```

So you can use `StateFlow` in normal Kotlin coroutine code too.

The difference is:

```text
Compose screen:
collect StateFlow with collectAsStateWithLifecycle()

Regular coroutine code:
collect StateFlow inside a coroutine
```

For example, later your app may want to observe patients from a repository or Room:

```kotlin
repository.observePatients()
```

If that returns a Flow, then a StateFlow-based ViewModel fits naturally.

The data can flow like this:

```text
Data
-> Repository / Room Flow
-> ViewModel StateFlow
-> Compose collectAsStateWithLifecycle()
-> UI
```

This means the ViewModel may collect a repository Flow and then update its own UI state manually.

For example, inside the repository, we can have runtime state defined as:

```kotlin
data class DeviceRuntimeState(
    val selectedDevice: String = "",
    val isMeasuring: Boolean = false,
    val isTransferring: Boolean = false,
    val isAnalyzing: Boolean = false,
    val transferInterrupted: Boolean = false
)

class DeviceRepository {
    private val _deviceState =
        MutableStateFlow(DeviceRuntimeState())

    val deviceState: StateFlow<DeviceRuntimeState> =
        _deviceState.asStateFlow()
}
```

Then inside the `ViewModel`, we can collect the repository state in `init`:

```kotlin
class DeviceViewModel(
    private val deviceRepository: DeviceRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(DeviceUiState())
    val uiState: StateFlow<DeviceUiState> = _uiState.asStateFlow()

    init { // runs automatically when an object is created.
        viewModelScope.launch { // 放在一个coroutine中，一直不停的观察值的变化，直到该 viewmodel object is destroyed。
            deviceRepository.deviceState.collect { deviceState ->
                _uiState.update {  // 这里的 collect 指的就是获取 deviceRepository.deviceState 的值。然后 deviceState -> 指用这个值更新 _uiState。此处的 it 则是用来代指_uiState，非常直接清楚。
                    it.copy(
                        selectedDevice = deviceState.selectedDevice,
                        isMeasuring = deviceState.isMeasuring,
                        isTransferring = deviceState.isTransferring,
                        isAnalyzing = deviceState.isAnalyzing
                    )
                } 
            }
        }
    }
}
```

Read that as:

```text
Start a coroutine when the ViewModel is created.
Listen to deviceRepository.deviceState.
Every time the repository emits a new deviceState,
copy the useful fields into this screen's _uiState.
```

This is different from `collectAsStateWithLifecycle()`.

`collectAsStateWithLifecycle()` is specifically for Compose UI code to easily observe the ViewModel's StateFlow. 

On the other hand, `viewModelScope.launch { flow.collect { ... } }` is the normal  Kotlin coroutines Flow APIs to observe a Flow inside a ViewModel or other Kotlin class.

In fact, `collectAsStateWithLifecycle()` plays a similar collect/listen role to this part in the ViewModel code:

```kotlin
viewModelScope.launch {
    someRepository.someFlow.collect { latestValue ->
        // ViewModel reacts to repository data
    }
}
```

Both mean:

```text
Listen to a Flow over time.
React when the Flow emits a new value.
```

But they are not exactly the same tool.

`collectAsStateWithLifecycle()` also converts the latest Flow value into Compose `State`, so Compose can recompose the UI automatically.

Also notice there are actually two separate operations in the ViewModel code: collect + update (which `collectAsStateWithLifecycle` does not do)

- collect = listen to values from a Flow over time, similar to `collectAsStateWithLifecycle`.
- update = publish a new value into your own MutableStateFlow, which `collectAsStateWithLifecycle` not have

So this pattern is common and includes these two operations:

```kotlin
init {
    viewModelScope.launch {
        someRepository.someFlow.collect { latestValue ->
            _uiState.update { currentState ->
                currentState.copy(
                    sampleId = latestValue.sampleId
                )
            }
        }
    }
}
```

But if you only want to set one fixed value once, you do not need `collect`:

```kotlin
init {
    _uiState.update { currentState ->
        currentState.copy(
            sampleId = "S001"
        )
    }
}
```

Use `collect` when another Flow is producing values over time.

Use `update` when this ViewModel wants to change its own `MutableStateFlow`.

This example shows the full chain:

```text
Repository has StateFlow.
ViewModel collects repository StateFlow.
ViewModel publishes updated UI state.
Compose collects ViewModel StateFlow.
```

---

## 10. Is StateFlow a different layer?

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

## 11. Common mistakes

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

## 12. Which one should you use?

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

## 13. Final mental model

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
