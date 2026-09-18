# Lesson 5 notes - callbacks in Compose architecture

This note continues Lesson 5 after the callback and lambda fundamentals. The focus here is how callbacks support Compose state ownership, event flow, ViewModels, recomposition, and asynchronous results.

---
## 1. State flows down and events flow up

The reusable `DeviceRow` example from Lesson 5 follows a central Compose pattern:

```text
Parent state
    |
    | selectedDevice and isSelected flow down
    v
DeviceRow
    |
    | onSelect(deviceName) event flows up
    v
Parent callback updates state
    |
    v
Compose redraws the affected UI
```

This is often summarized as:

```text
State flows down.
Events flow up.
```

More precisely:

```text
The parent passes current values down to children.
Children invoke callbacks to report user events upward.
The state owner handles the event and changes state.
The new state flows down during recomposition.
```

The callback itself usually does not "flow upward" as data. The parent passes the callback down, and the child invokes it to send an event upward.

---

## 2. State hoisting

**State hoisting** means moving state to the nearest parent that needs to control or share it.

Here is a component that owns its own text:

```kotlin
@Composable
fun InternalSampleInput() {
    var sampleId by remember {
        mutableStateOf("")
    }

    OutlinedTextField(
        value = sampleId,
        onValueChange = { newValue ->
            sampleId = newValue
        }
    )
}
```

This works, but the parent cannot read or control `sampleId`.

The hoisted version is:

```kotlin
@Composable
fun SampleInput(
    sampleId: String,
    onSampleIdChange: (String) -> Unit
) {
    OutlinedTextField(
        value = sampleId,
        onValueChange = onSampleIdChange
    )
}
```

The parent owns the state:

```kotlin
@Composable
fun ResearchScreen() {
    var sampleId by remember {
        mutableStateOf("")
    }

    SampleInput(
        sampleId = sampleId,
        onSampleIdChange = { newValue ->
            sampleId = newValue
        }
    )
}
```

The common pair is:

```kotlin
value: T
onValueChange: (T) -> Unit
```

This makes the child reusable, previewable, and easy to test.

---

## 3. A complete navigation callback path

Now follow a callback through several levels.

### Level 1: `NavigationItem` reports a click

```kotlin
@Composable
fun NavigationItem(
    label: String,
    onClick: () -> Unit
) {
    Row(
        modifier = Modifier.clickable(onClick = onClick)
    ) {
        Text(label)
    }
}
```

`NavigationItem` does not decide which app state should change. It simply exposes a click event.

### Level 2: `SideNavBar` knows which item was clicked

```kotlin
@Composable
fun SideNavBar(
    activeScreen: TabletScreen,
    onNavigate: (TabletScreen) -> Unit
) {
    val items = listOf(
        TabletScreen.Device,
        TabletScreen.Data,
        TabletScreen.Repository,
        TabletScreen.Settings
    )

    Column {
        items.forEach { item ->
            NavigationItem(
                label = item.navLabel,
                onClick = {
                    onNavigate(item)
                }
            )
        }
    }
}
```

Each `onClick` lambda captures its current `item`.

If the Data row is clicked:

```text
item == TabletScreen.Data
```

so the callback call becomes conceptually:

```kotlin
onNavigate(TabletScreen.Data)
```

### Level 3: `TabletShell` forwards the callback

```kotlin
@Composable
fun TabletShell(
    activeScreen: TabletScreen,
    onNavigate: (TabletScreen) -> Unit,
    content: @Composable () -> Unit
) {
    Row {
        SideNavBar(
            activeScreen = activeScreen,
            onNavigate = onNavigate
        )

        content()
    }
}
```

This does not call `onNavigate`. It passes the same function value to `SideNavBar`.

### Level 4: the app owns navigation state

```kotlin
@Composable
fun ResearchTabletApp() {
    var currentScreenName by rememberSaveable {
        mutableStateOf(TabletScreen.Device.name)
    }

    val currentScreen = TabletScreen.valueOf(currentScreenName)

    TabletShell(
        activeScreen = currentScreen,
        onNavigate = { selectedScreen ->
            currentScreenName = selectedScreen.name
        }
    ) {
        when (currentScreen) {
            TabletScreen.Device -> DeviceScreen()
            TabletScreen.Data -> DataScreen()
            TabletScreen.Repository -> RepositoryScreen()
            TabletScreen.Settings -> SettingsScreen()
        }
    }
}
```

### Full event sequence

```text
1. User taps the Data navigation item.
2. Compose invokes NavigationItem's onClick callback.
3. The callback invokes onNavigate(item).
4. item is TabletScreen.Data.
5. The event passes through TabletShell to ResearchTabletApp's callback.
6. selectedScreen receives TabletScreen.Data.
7. currentScreenName becomes "Data".
8. Compose observes the state change.
9. ResearchTabletApp recomposes.
10. when (currentScreen) now displays DataScreen.
```

Notice the separation of responsibilities:

```text
NavigationItem knows that it was clicked.
SideNavBar knows which item was clicked.
ResearchTabletApp owns navigation state and decides what to display.
```

---

## 4. Nullable callbacks

You may encounter:

```kotlin
onDeviceSelect: ((String) -> Unit)? = null
```

Break it down:

```text
onDeviceSelect       parameter name
(String) -> Unit     function type
?                    the function value may be null
= null               default value when the caller omits it
```

The extra parentheses group the whole function type before `?` makes it nullable:

```kotlin
((String) -> Unit)?
```

A nullable function cannot be called directly:

```kotlin
onDeviceSelect(deviceName) // Error: it may be null
```

Use a safe call with `invoke`:

```kotlin
onDeviceSelect?.invoke(deviceName)
```

Read it as:

```text
If onDeviceSelect is not null, call it with deviceName.
If it is null, do nothing.
```

### Three callback designs

Required callback:

```kotlin
onDeviceSelect: (String) -> Unit
```

Use this when handling the event is required.

Default no-operation callback:

```kotlin
onDeviceSelect: (String) -> Unit = {}
```

Use this when doing nothing is a valid default. Be careful: it can also hide wiring that was accidentally forgotten.

Nullable callback:

```kotlin
onDeviceSelect: ((String) -> Unit)? = null
```

Use this when the presence or absence of a listener has meaning, or when the UI behaves differently if no listener exists.

For important user actions, a required callback is often clearest. A preview can pass an explicit empty lambda:

```kotlin
DeviceScreen(
    onDeviceSelect = {}
)
```

---

## 5. Callbacks help keep screens independent

Imagine that selecting a device must clear another screen's temporary data.

One approach is to pass `DataViewModel` into `DeviceScreen`:

```kotlin
DeviceScreen(
    deviceViewModel = deviceViewModel,
    dataViewModel = dataViewModel
)
```

Then `DeviceScreen` directly knows about another screen's ViewModel:

```kotlin
dataViewModel.clearDataScreen()
```

This creates unnecessary coupling. The device screen now knows a navigation-level coordination decision.

A callback keeps that decision in the parent:

```kotlin
DeviceScreen(
    viewModel = deviceViewModel,
    onDeviceSelect = {
        dataViewModel.clearDataScreen()
    }
)
```

Inside `DeviceScreen`:

```kotlin
onSelectDevice = { deviceName ->
    viewModel.selectDevice(deviceName)
    onDeviceSelect(deviceName)
}
```

Now the responsibilities are:

```text
DeviceScreen:
    displays device UI
    updates its own DeviceViewModel
    reports that a device was selected

Parent app composable:
    coordinates effects involving another screen or ViewModel

DataViewModel:
    owns and changes data-screen state
```

This improves:

- screen reuse
- previews
- testing
- separation of responsibilities
- future navigation changes

Callbacks are not a reason to put all coordination in `MainActivity`. As an app grows, a navigation host, shared state manager, use case, or another appropriate owner may coordinate the operation. The important point is that a reusable child screen should not receive unrelated dependencies merely to trigger one action.

---

## 6. Callbacks and ViewModels

In a ViewModel-based screen, state usually flows down from the ViewModel and events call ViewModel operations.

```kotlin
@Composable
fun ResearchRoute(
    viewModel: ResearchViewModel
) {
    val uiState = viewModel.uiState

    ResearchScreen(
        sampleId = uiState.sampleId,
        measurements = uiState.measurements,
        onSampleIdChange = viewModel::updateSampleId,
        onAddMeasurement = viewModel::addMeasurement,
        onClear = viewModel::clearMeasurements
    )
}
```

The screen remains mostly stateless:

```kotlin
@Composable
fun ResearchScreen(
    sampleId: String,
    measurements: List<Measurement>,
    onSampleIdChange: (String) -> Unit,
    onAddMeasurement: () -> Unit,
    onClear: () -> Unit
) {
    OutlinedTextField(
        value = sampleId,
        onValueChange = onSampleIdChange
    )

    Button(onClick = onAddMeasurement) {
        Text("Measure")
    }

    Button(onClick = onClear) {
        Text("Clear")
    }
}
```

The event path is:

```text
User action
-> screen callback
-> ViewModel operation
-> ViewModel state update
-> new state observed by Compose
-> recomposition
-> updated screen
```

Lesson 9 introduces ViewModels in detail. Lesson 10 then shows the same state pattern with StateFlow.

---

## 7. Callbacks and recomposition

Suppose a callback updates Compose state:

```kotlin
Button(
    onClick = {
        measurementCount++
        statusMessage = "Measurement added"
    }
) {
    Text("Measure")
}
```

The click callback normally runs on Android's main thread.

The sequence is approximately:

```text
onClick starts
measurementCount changes
statusMessage changes
Compose records that UI needs updating
onClick finishes
Compose can recompose using the latest values
```

Compose does not normally interrupt the middle of the running `onClick` callback to recompose. Several state changes in one short callback can therefore be applied together.

This is helpful, but it also means long blocking work inside `onClick` prevents the main thread from updating the UI smoothly.

Avoid this pattern for slow work:

```kotlin
Button(
    onClick = {
        Thread.sleep(5000)
        readLargeDeviceFile()
    }
) {
    Text("Load")
}
```

Later coroutine lessons show how to move slow work away from the main thread.

---

## 8. Callback versus StateFlow versus coroutine

These concepts solve different problems.

| Concept | Main purpose | Typical example |
|---|---|---|
| Callback | Report an event or let another component supply behavior | `onClick`, `onNavigate`, `onResult` |
| StateFlow | Expose changing state as a stream of values | ViewModel `uiState` |
| Coroutine | Run suspendable or asynchronous work without blocking a thread | save file, read device, query Room |

They often work together:

```text
1. User taps Start.
2. Compose invokes the onStart callback.
3. The callback calls a ViewModel function.
4. The ViewModel launches a coroutine.
5. The coroutine reads data from a device.
6. The ViewModel updates StateFlow.
7. Compose collects the new state and redraws the UI.
```

So:

```text
Callback communicates the event.
Coroutine performs ongoing or suspendable work.
StateFlow communicates changing state.
```

One does not replace the others.

---

## 9. A callback result that arrives later

Some APIs use one callback to start work and another callback to report the eventual result.

The Android file picker is an example:

```kotlin
val createCsvLauncher = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.CreateDocument("text/csv"),
    onResult = { uri ->
        // Runs after the picker finishes.
    }
)
```

The button callback starts the request:

```kotlin
Button(
    onClick = {
        createCsvLauncher.launch("measurements.csv")
    }
) {
    Text("Export")
}
```

The two callbacks run at different moments:

```text
onClick:
    user requests export
    app opens the picker

onResult:
    Android returns the selected Uri later
    app handles the result
```

This is an important use of callbacks: code can be registered now and invoked after an external event occurs.

Lesson 8 applies this idea to CSV export and explains the file-picker timeline in more detail.

---

## 10. Common architecture mistakes

### Mistake 1: passing an unrelated ViewModel into a child screen

If the child only needs to report one event, pass a callback rather than an entire unrelated ViewModel.

### Mistake 2: making every callback nullable

Nullability adds another state to reason about. Use a required callback when the action must be handled.

### Mistake 3: doing heavy work directly in a click callback

Callbacks on the main thread should stay short. Let a ViewModel and coroutine coordinate slow work.

---

## 11. Practice exercises

### Exercise 1: hoist the state

Rewrite a child composable that owns `sampleId` so it instead receives:

```kotlin
sampleId: String
onSampleIdChange: (String) -> Unit
```

### Exercise 2: choose the mechanism

Choose callback, StateFlow, or coroutine for each job:

```text
Report that a button was clicked.
Expose the latest ViewModel UI state.
Write a large export file without blocking the UI.
```

---

## 12. Exercise answers

### Answer 1

```kotlin
@Composable
fun SampleInput(
    sampleId: String,
    onSampleIdChange: (String) -> Unit
) {
    OutlinedTextField(
        value = sampleId,
        onValueChange = onSampleIdChange
    )
}
```

### Answer 2

```text
Button click event                -> callback
Latest ViewModel UI state         -> StateFlow
Large non-blocking file operation -> coroutine
```
