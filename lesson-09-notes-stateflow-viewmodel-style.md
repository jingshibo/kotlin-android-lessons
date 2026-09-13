# Lesson 9 Notes - StateFlow ViewModel Style

This note explains a ViewModel state style you may see in other Android examples:

```kotlin
val state by viewModel.uiState.collectAsState()
```

and:

```kotlin
_uiState.update { currentState ->
    currentState.copy(
        sampleId = "S001"
    )
}
```

This style uses:

```text
MutableStateFlow / StateFlow
```

instead of:

```text
mutableStateOf
```

It is not a different app architecture.

It is a different implementation of ViewModel state.

---

## 1. Why you did not see this first

Lesson 8 used this simpler ViewModel pattern:

```kotlin
var uiState by mutableStateOf(ResearchUiState())
    private set
```

That was chosen because it is easier to read when first learning Compose architecture.

The lesson wanted to teach this idea first:

```text
UI displays state.
UI calls ViewModel functions.
ViewModel changes state.
Compose redraws the UI.
```

If we started with `StateFlow`, there would be extra syntax:

```text
MutableStateFlow
StateFlow
asStateFlow()
update
collectAsState()
collectAsStateWithLifecycle()
```

Those are useful, but they are not the first idea.

The first idea is architecture.

---

## 2. The previous Lesson 8 style

In Lesson 8, the ViewModel owns one state object:

```kotlin
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

The composable reads it like this:

```kotlin
val uiState = viewModel.uiState
```

This works because `mutableStateOf` is observable by Compose.

When the ViewModel assigns:

```kotlin
uiState = uiState.copy(...)
```

Compose knows that the screen may need to redraw.

---

## 3. The StateFlow style

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

The composable reads it like this:

```kotlin
val uiState by viewModel.uiState.collectAsState()
```

or, in most Android Compose screens:

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

That means:

```text
Collect the latest value from the ViewModel.
Turn it into Compose-readable state.
Redraw when it changes.
```

---

## 4. What _uiState and uiState mean

StateFlow examples often use two properties:

```kotlin
private val _uiState = MutableStateFlow(ResearchUiState())
val uiState: StateFlow<ResearchUiState> = _uiState.asStateFlow()
```

The private one:

```text
_uiState
```

is mutable.

Only the ViewModel can use it.

The public one:

```text
uiState
```

is read-only.

The UI can observe it, but cannot directly change it.

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

---

## 5. What update means

This:

```kotlin
_uiState.update { currentState ->
    currentState.copy(
        sampleId = "S001"
    )
}
```

means:

```text
Get the current state.
Create a copied state with one change.
Save that copied state as the new state.
Notify collectors.
```

It is very similar to:

```kotlin
uiState = uiState.copy(
    sampleId = "S001"
)
```

The biggest difference is where the value is stored:

```text
mutableStateOf version:
uiState directly holds Compose state.

StateFlow version:
_uiState holds a flow of state values.
```

---

## 6. Code comparison

### Updating text input

Previous Lesson 8 style:

```kotlin
fun updateSampleId(newSampleId: String) {
    uiState = uiState.copy(
        sampleId = newSampleId
    )
}
```

StateFlow style:

```kotlin
fun updateSampleId(newSampleId: String) {
    _uiState.update { currentState ->
        currentState.copy(
            sampleId = newSampleId
        )
    }
}
```

### Reading state in the screen

Previous Lesson 8 style:

```kotlin
val uiState = viewModel.uiState
```

StateFlow style:

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

### Reading current state inside the ViewModel

Previous Lesson 8 style:

```kotlin
val sampleId = uiState.sampleId
```

StateFlow style:

```kotlin
val sampleId = _uiState.value.sampleId
```

---

## 7. Pros and cons

| Style | Pros | Cons |
|---|---|---|
| `mutableStateOf` ViewModel state | Short, direct, beginner-friendly, very natural with Compose | Tied closely to Compose, less useful when combining with Flow-based data |
| `StateFlow` ViewModel state | Common in modern Android, works well with coroutines, Room flows, repositories, and lifecycle-aware collection | More syntax, more concepts, easier to confuse at first |

So the earlier lesson was not wrong.

It was a simpler first version.

The StateFlow version is a more scalable version.

---

## 8. When StateFlow becomes useful

StateFlow becomes especially useful when the app has:

```text
Room data streams
repository functions returning Flow
background loading
multiple state sources
larger ViewModels
tests that observe state changes
```

For example, later your app may want to observe patients from Room:

```kotlin
repository.observePatients()
```

If that returns a `Flow`, then a StateFlow-based ViewModel fits naturally.

The data can flow like this:

```text
Room Flow
-> Repository Flow
-> ViewModel StateFlow
-> Compose collectAsStateWithLifecycle()
-> UI
```

---

## 9. Important mental model

Do not think:

```text
mutableStateOf = old architecture
StateFlow = new architecture
```

Think:

```text
mutableStateOf = simpler Compose state holder
StateFlow = more general coroutine state holder
```

Both can be used inside the same architecture:

```text
Screen
-> ViewModel
-> Repository
-> data/device/storage layers
```

The ViewModel is still the application/screen state layer.

The repository is still the coordination/data layer.

StateFlow only changes the way the ViewModel exposes changing state to the UI.

---

## 10. Tiny translation guide

| Previous Lesson 8 code | StateFlow version |
|---|---|
| `var uiState by mutableStateOf(...)` | `private val _uiState = MutableStateFlow(...)` |
| `private set` | expose `val uiState = _uiState.asStateFlow()` |
| `uiState = uiState.copy(...)` | `_uiState.update { it.copy(...) }` |
| `val uiState = viewModel.uiState` | `val uiState by viewModel.uiState.collectAsStateWithLifecycle()` |
| `uiState.sampleId` inside ViewModel | `_uiState.value.sampleId` inside ViewModel |

Final mental model:

```text
StateFlow is a more advanced state holder; the state is still ViewModel state.
```
