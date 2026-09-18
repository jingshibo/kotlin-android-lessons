# Lesson 5 notes - callbacks in Compose architecture

This note builds on Lesson 5's callback and lambda fundamentals. Its first three sections explain how a parent and child work together: who owns state, what is passed between them, when a callback runs, and how a state change updates the UI.

We start with one parent and one child, explain why state may belong in the parent, and then follow a navigation event through several components. Later sections apply these ideas to ViewModels, optional callbacks, and asynchronous results.

---

## 1. State flows down and events flow up

### 1.1 The parent owns state; the child receives values and a callback

A **parent** composable calls a **child** composable. 
Here, `DeviceList` is the parent and it calls the child `DeviceRow`.

State is information that can change and affect what the UI displays. In this example, it is the selected device name:

```kotlin
@Composable
fun DeviceList() {
    var selectedDevice by remember {
        mutableStateOf("")
    }

    DeviceRow(
        deviceName = "BT-Sensor-01",
        isSelected = selectedDevice == "BT-Sensor-01",
        onSelect = { deviceName ->
            selectedDevice = deviceName
        }
    )
}
```

`mutableStateOf("")` creates an observable state holder, initially containing an empty string. `remember` keeps that holder across recompositions while this part of the UI remains in the composition. The `by` syntax lets us read and update its value using `selectedDevice`.

We call this **parent-owned state** because `DeviceList` declares it and supplies the code that updates it. Compose retains the holder; the `DeviceList` function does not need to keep running to keep it alive.

The child accepts three parameters:

```kotlin
@Composable
fun DeviceRow(
    deviceName: String,
    isSelected: Boolean,
    onSelect: (String) -> Unit
) {
    Row(
        modifier = Modifier.selectable(
            selected = isSelected,
            onClick = { onSelect(deviceName) }
        )
    ) {
        Text(if (isSelected) "$deviceName (selected)" else deviceName)
    }
}
```

| Parameter received by the child | What the parent supplies | Purpose |
|---|---|---|
| `deviceName` | `"BT-Sensor-01"` | The device this row represents |
| `isSelected` | The result of `selectedDevice == "BT-Sensor-01"` | Whether to display the row as selected |
| `onSelect` | `{ deviceName -> selectedDevice = deviceName }` | Behavior to invoke when this device is selected |

`DeviceRow` declares the callback **parameter**. `DeviceList` supplies the actual **function body**. Passing that function into `onSelect` makes it available to the child; it does not execute the body.

### 1.2 A click invokes the callback and changes state

The phrase **state flows down** means the parent supplies values for the child to display. The callback function also travels down as an argument:

```text
DeviceList
    |
    | deviceName, isSelected
    | onSelect callback function
    v
DeviceRow
```

The phrase **events flow up** describes the child reporting a user action by invoking that supplied callback. Read this diagram from the bottom upward:

```text
Parent-created callback updates selectedDevice
    ^
    | deviceName argument is received by the callback
    |
DeviceRow calls onSelect(deviceName)
    ^
    | click handler runs
    |
User clicks DeviceRow
```

The **event** is the selection action; the **event data** is the device name passed as the argument. The callback body decides how to respond. Here, it assigns that name to `selectedDevice`.

This is an ordinary function call. Calling the callback runs its body as part of handling the click; it does not first rerun the parent composable. The function can update parent-owned state because it retains access to that state, as Section 3.4 explains.

When the selection changes, Compose schedules the affected UI to recompose. **Recomposition** means running the relevant composable code again using the updated state. The parent now supplies `isSelected = true`, so the row displays its selected appearance. Compose can skip components that do not need updating.

The child has received a Boolean value and a function, rather than direct access to the parent's state holder. It reports the selection through that function. A callback could instead reject the selection or only log it; invoking a callback does not itself guarantee a state change.

---

## 2. State hoisting: choosing who controls the value

Section 1 started with state already in the parent. **State hoisting** is how we arrive at that arrangement: move state out of a child when another component needs to read or control it. For state shared by several children, choose a common parent that can serve them.

Consider a text input that owns its value:

```kotlin
@Composable
fun InternalSampleInput() {
    var sampleId by remember { mutableStateOf("") }

    OutlinedTextField(
        value = sampleId,
        onValueChange = { newValue -> sampleId = newValue }
    )
}
```

This works for a self-contained input. But a parent that needs the sample ID for a Save button cannot read this local state through the component's parameters.

To let the parent control the value, give the child a value parameter and a callback parameter:

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

@Composable
fun ResearchScreen() {
    var sampleId by remember { mutableStateOf("") }

    SampleInput(
        sampleId = sampleId,
        onSampleIdChange = { newValue ->
            sampleId = newValue
        }
    )
}
```

There is now one state holder, owned by `ResearchScreen`. `SampleInput` receives its current string value. The parameter also being named `sampleId` does not create another state holder.

`onValueChange = onSampleIdChange` forwards the supplied function to `OutlinedTextField`. When the user edits the text, the field invokes it with the proposed new string. The parent-provided body stores that string, and recomposition supplies the updated value to the field.

This value-and-callback pair lets the parent control the input while the child concentrates on displaying it. It also makes the child easier to reuse, preview, and test.

---

## 3. A complete navigation callback path

Now apply the same pattern across several levels:

```text
ResearchTabletApp    owns navigation state and creates onNavigate
    |
    v
TabletShell          forwards the current screen and onNavigate
    |
    v
SideNavBar           connects each item to an onClick handler
    |
    v
NavigationItem       displays a clickable navigation entry
```

Parent and child are relative terms: `TabletShell` is a child of `ResearchTabletApp` and a parent of `SideNavBar`.

The example uses this screen type:

```kotlin
enum class TabletScreen(val navLabel: String) {
    Device("Device"),
    Data("Data"),
    Repository("Repository"),
    Settings("Settings")
}
```

The screen composables such as `DeviceScreen()` and `DataScreen()` below stand for the app's existing screens. Compose imports are omitted.

### 3.1 The parent owns navigation state and supplies the callback

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

`currentScreenName` is the remembered state value. `currentScreen` is the enum value derived from it during composition. The parent passes that enum value as `activeScreen`, and supplies the lambda that will update the state as `onNavigate`.

`rememberSaveable` retains state across recompositions and also supports saving and restoring this string through supported activity or process recreation. It saves the state value, not the callback function.

At this point, the navigation callback has been supplied, but its assignment has not run. The final trailing lambda supplies the UI content that `TabletShell` will display.

### 3.2 The intermediate component forwards the callback

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

In `onNavigate = onNavigate`, the left side names `SideNavBar`'s parameter; the right side is the function value received by `TabletShell`. This forwards the same function without invoking it.

`content()` is different: it invokes the composable content lambda now to describe the screen UI. A lambda's execution time depends on where it is called. Here, the content lambda runs during composition; the navigation lambda is invoked by a later click handler.

### 3.3 The child connects a click to the selected item

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
                isSelected = item == activeScreen,
                onClick = {
                    onNavigate(item)
                }
            )
        }
    }
}

@Composable
fun NavigationItem(
    label: String,
    isSelected: Boolean,
    onClick: () -> Unit
) {
    Row(
        modifier = Modifier.selectable(
            selected = isSelected,
            onClick = onClick
        )
    ) {
        Text(if (isSelected) "$label (selected)" else label)
    }
}
```

There are two event functions here. `NavigationItem` receives an `onClick: () -> Unit`, which needs no argument. `SideNavBar` supplies its body, `{ onNavigate(item) }`, which already has access to that row's `item` and the supplied `onNavigate` function.

When the Data entry is clicked, that body calls `onNavigate(TabletScreen.Data)`. The parent-created function receives `TabletScreen.Data` as `selectedScreen` and assigns `selectedScreen.name` to `currentScreenName`.

`NavigationItem` handles the click, `SideNavBar` supplies the event data, and the parent-created callback defines the state update.

### 3.4 How the callback keeps access to parent state

This is the connection to [Lesson 5, Section 16: Lambdas can capture surrounding values](lesson-05-callback-functions-lambdas-and-event-flow.md#16-lambdas-can-capture-surrounding-values):

> A lambda can carry access to variables from the place where it was created.

This retained access is called **capture**; a function together with its captured context is a **closure**.

In our example, the parent's lambda can access the remembered state holder behind `currentScreenName`. Passing the lambda to a child preserves that access. The child does not need a separate state-holder parameter for the callback to update it.

To make the holder visible, the parent could express the same state and callback without `by`:

```kotlin
val screenNameState = rememberSaveable {
    mutableStateOf(TabletScreen.Device.name)
}

val navigate: (TabletScreen) -> Unit = { selectedScreen ->
    screenNameState.value = selectedScreen.name
}
```

Here, `navigate` retains access to `screenNameState`. Calling it later writes to that same holder. The earlier assignment `currentScreenName = selectedScreen.name` performs the corresponding update using delegated-property syntax.

**The callback does not contain a frozen copy of the original screen name.** It retains access to the holder whose value can change. By comparison, the `activeScreen` parameter receives the enum value calculated for that composition. Updated display values are supplied during recomposition.

When the child invokes the callback, its body executes as a normal function call using that captured access. There is no separate execution location called "the parent position," and no need to restart `ResearchTabletApp` before the assignment can happen. "Parent-defined" describes where the behavior was supplied and whose state it updates.

A composable function finishes after describing its UI. While this UI remains active, Compose retains the remembered state and the UI retains its click handler, which can reach the navigation callback. That is why a later click still works. This does not mean the callback or state is kept forever after the UI is removed.

### 3.5 The complete timeline: composition, click, and UI update

Assume the app starts on Device and the user then selects Data:

1. `ResearchTabletApp` runs to describe the initial UI.
2. `rememberSaveable` supplies the navigation state holder, initially containing `"Device"`.
3. The parent reads `currentScreenName` and derives `currentScreen = TabletScreen.Device`.
4. The parent creates the `onNavigate` lambda. It captures access to the state holder; its body does not run yet.
5. The parent passes the current screen value and the callback to `TabletShell`, which forwards them to `SideNavBar`.
6. `SideNavBar` supplies each `NavigationItem` with a click lambda that captures that row's `item` and the `onNavigate` function.
7. The UI is composed. The composable calls finish, while the displayed UI retains its click handlers and Compose retains the remembered state.
8. The user clicks Data. The UI invokes that entry's `onClick` handler.
9. The click handler calls `onNavigate(item)`, with `item` equal to `TabletScreen.Data`.
10. The parent-created lambda runs. Its `selectedScreen` parameter receives `TabletScreen.Data`.
11. Through its captured access, the lambda updates the remembered state: `currentScreenName = selectedScreen.name`, changing `"Device"` to `"Data"`.
12. Compose observes the change and schedules recomposition of the UI that read the state. The click handler finishes.
13. During recomposition, the parent reads the retained `"Data"` value and derives `currentScreen = TabletScreen.Data`. The state is not reset to its initial value.
14. The updated screen value flows down. The Data entry is selected, and the content lambda's `when` chooses `DataScreen()`. Compose may skip child calls that do not need updating.
15. Event callbacks may be reused or new instances may be supplied for future clicks. Recomposition does not itself invoke the `onNavigate` body.

The distinction in the last step matters: running composable code again can create or supply a function value without executing that function's body. In this example, the navigation body runs when the click handler calls it.

References: [Compose state and state hoisting](https://developer.android.com/develop/ui/compose/state), [Kotlin closures](https://kotlinlang.org/docs/lambdas.html#closures), and [Compose's execution and recomposition model](https://developer.android.com/develop/ui/compose/mental-model).

---

## 4. Does every click cause recomposition?

**No. Invoking a callback does not, by itself, request recomposition of your screen.** What matters is what the callback does.

### 4.1 Logging an event versus changing observed state

Suppose we replace the navigation callback from Section 3 with this body:

```kotlin
onNavigate = { selectedScreen ->
    println("Tab clicked: ${selectedScreen.name}")
}
```

The click still reaches the callback, and the message is printed to developer output (typically Logcat on Android). But this body does not change navigation state, so it does not request a new screen or a new selected tab. `println` does not display text in the app's UI.

Compare that with:

```kotlin
onNavigate = { selectedScreen ->
    currentScreenName = selectedScreen.name
}
```

In Section 3, `currentScreenName` uses `mutableStateOf`, and the parent reads it during composition to choose the screen. Changing it from `"Device"` to `"Data"` schedules recomposition of the code that observed that state.

Compose tracks state reads during composition. A change to an observed state holder tells Compose which composition scopes need updating. Assigning to a plain, non-state variable does not send that notification. See [Compose state tracking](https://developer.android.com/develop/ui/compose/state#state-in-composables).

Two details keep this rule precise:

- With the default `mutableStateOf` behavior, assigning an equal value does not schedule recomposition through that state holder. Selecting Data when `currentScreenName` is already `"Data"` still invokes the callback, but that assignment adds no state change. See the [MutableState reference](https://developer.android.com/reference/kotlin/androidx/compose/runtime/MutableState).
- A logging-only callback does not mean the entire UI becomes inactive. A button can still show press feedback, and other state changes can independently cause UI updates. Interaction feedback may involve drawing updates without recomposing your screen. See [Compose interaction handling](https://developer.android.com/develop/ui/compose/touch-input/user-interactions/handling-interactions).

### 4.2 Displaying the clicked tab message on screen

To show a changing message in the UI, store it as observable state and read it in a `Text` composable. Here is a replacement for `ResearchTabletApp` from Section 3, using the same `TabletShell` and screen components:

```kotlin
@Composable
fun ResearchTabletApp() {
    var currentScreenName by rememberSaveable {
        mutableStateOf(TabletScreen.Device.name)
    }
    var clickedTabMessage by remember {
        mutableStateOf("No tab clicked yet")
    }

    val currentScreen = TabletScreen.valueOf(currentScreenName)

    TabletShell(
        activeScreen = currentScreen,
        onNavigate = { selectedScreen ->
            currentScreenName = selectedScreen.name
            clickedTabMessage = "You clicked the '${selectedScreen.navLabel}' tab!"
        }
    ) {
        Column {
            Text(text = clickedTabMessage)

            when (currentScreen) {
                TabletScreen.Device -> DeviceScreen()
                TabletScreen.Data -> DataScreen()
                TabletScreen.Repository -> RepositoryScreen()
                TabletScreen.Settings -> SettingsScreen()
            }
        }
    }
}
```

The two state values control different UI details: `currentScreenName` controls navigation, and `clickedTabMessage` controls the visible message. The callback captures access to both holders, just as Section 3.4 described.

When the user selects Data:

1. The child invokes `onNavigate(TabletScreen.Data)`.
2. The callback updates the navigation state and message state.
3. Compose schedules the affected UI to recompose using those updated values. Each assignment does not require a separate immediate recomposition.
4. The content displays `You clicked the 'Data' tab!` and `DataScreen()`.

The callback updates the message; `Text` displays it when the UI is composed. You do not call `Text(...)` inside the ordinary click callback.

### 4.3 Showing a message without changing screens

To report the click on screen while keeping the current screen selected, replace only the callback body in the example above:

```kotlin
onNavigate = { selectedScreen ->
    clickedTabMessage = "You clicked the '${selectedScreen.navLabel}' tab!"
}
```

Now `currentScreenName` stays unchanged. A changed `clickedTabMessage` still causes the UI that reads the message to recompose. The message can say Data was clicked while the Device screen remains selected and displayed.

Clicking the same tab again produces the same message string, so that assignment does not request another recomposition. The callback still runs on every click. If the UI should visibly record every click, store additional state such as a click count and include it in the displayed text.

`mutableStateOf` is a simple way to make this message observable. Later lessons also show state supplied by a ViewModel or collected from a flow; the state does not have to be declared locally beside `Text`.

---

## 5. Nullable callbacks

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

The extra parentheses group the whole function type before `?` makes the function value nullable:

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
onDeviceSelect: (String) -> Unit = { _ -> }
```

Use this when doing nothing is a valid default. The `_` means "this callback receives a `String`, but this implementation ignores it." Be careful: a no-operation default can also hide wiring that was accidentally forgotten.

Nullable callback:

```kotlin
onDeviceSelect: ((String) -> Unit)? = null
```

Use this when the presence or absence of a listener has meaning, or when the UI behaves differently if no listener exists.

For important user actions, a required callback is often clearest. A preview can pass an explicit empty lambda:

```kotlin
DeviceScreen(
    onDeviceSelect = { _ -> }
)
```

---

## 6. Callbacks help keep screens independent

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
    onDeviceSelect = { _ ->
        dataViewModel.clearDataScreen()
    }
)
```

The `_` means the parent receives the selected device name but does not need to use it for this particular effect.

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

## 7. Callbacks and ViewModels

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

## 8. Callbacks and recomposition

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

## 9. Callback versus StateFlow versus coroutine

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

## 10. A callback result that arrives later

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

## 11. Common architecture mistakes

### Mistake 1: passing an unrelated ViewModel into a child screen

If the child only needs to report one event, pass a callback rather than an entire unrelated ViewModel.

### Mistake 2: making every callback nullable

Nullability adds another state to reason about. Use a required callback when the action must be handled.

### Mistake 3: doing heavy work directly in a click callback

Callbacks on the main thread should stay short. Let a ViewModel and coroutine coordinate slow work.

---

## 12. Practice exercises

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

## 13. Exercise answers

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
