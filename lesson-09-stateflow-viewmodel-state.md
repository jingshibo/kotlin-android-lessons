# Lesson 9 - StateFlow ViewModel State

In Lesson 8, the app used this ViewModel state style:

```kotlin
var uiState by mutableStateOf(ResearchUiState())
    private set
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

## 2. Previous ViewModel style

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

## 3. StateFlow ViewModel style

The StateFlow version looks like this:

```kotlin
import androidx.lifecycle.ViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update

class ResearchViewModel : ViewModel() {

    private val _uiState = MutableStateFlow(ResearchUiState())

    val uiState: StateFlow<ResearchUiState> =
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

| Property | Who uses it | Can it be changed? |
|---|---|---|
| `_uiState` | ViewModel only | Yes |
| `uiState` | UI and other outside code | No, read-only |

This is the StateFlow version of:

```kotlin
private set
```

The ViewModel can update the state.

The UI can observe the state.

The UI cannot directly replace the state.

---

## 4. Why the underscore exists

This line creates the real mutable state holder:

```kotlin
private val _uiState = MutableStateFlow(ResearchUiState())
```

The underscore is a common naming convention.

It means:

```text
This is the private mutable version.
Do not expose this directly to the UI.
```

Then this line exposes a read-only version:

```kotlin
val uiState: StateFlow<ResearchUiState> =
    _uiState.asStateFlow()
```

So the ViewModel says:

```text
Inside this class, I can change _uiState.
Outside this class, you can only observe uiState.
```

That protects the app flow.

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
val sampleId = uiState.sampleId.trim()
```

With StateFlow, `uiState` is now a `StateFlow`.

So inside the ViewModel, read the current value like this:

```kotlin
val sampleId = _uiState.value.sampleId.trim()
```

or:

```kotlin
val currentState = _uiState.value
val sampleId = currentState.sampleId.trim()
```

For example:

```kotlin
import kotlin.random.Random

fun addMeasurement() {
    val currentState = _uiState.value
    val sampleId = currentState.sampleId.trim()

    if (sampleId.isBlank() || !currentState.isConnected) {
        _uiState.update { currentState ->
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

Screen reading, previous style:

```kotlin
val uiState = viewModel.uiState
```

Screen reading, StateFlow style:

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

Same mental model:

```text
old:
ViewModel updates uiState
-> Compose observes mutableStateOf
-> UI redraws

new:
ViewModel updates _uiState
-> StateFlow emits new value
-> Compose collects StateFlow
-> UI redraws
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
Room flows
repository streams
background work
multiple screens observing shared state
more advanced testing
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
ViewModel has coroutine-observable state.
Compose collects it and turns it into Compose state.
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

Same architecture.

More scalable state tool.
