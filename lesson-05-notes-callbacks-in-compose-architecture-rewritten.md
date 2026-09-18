# Lesson 5 Notes - Choosing State Owners and Connecting Components

This is an alternative to [the original architecture note](lesson-05-notes-callbacks-in-compose-architecture.md). Read it after the [rewritten callback fundamentals](lesson-05-callback-functions-lambdas-and-event-flow-rewritten.md), or when you are comfortable with function types such as `() -> Unit` and `(String) -> Unit`.

This note answers one practical question:

**When several components depend on the same information, where should that information live, and how should they change it?**

We will build a small device-selection screen. One component lets the user select a device. Another displays the selection and provides a clear button. Both must agree about which device is selected.

## 1. Start with the problem: two components need the same selection

Imagine these two components on one screen:

```text
Device picker                 Selection panel
-----------------------       -----------------------
BT-Sensor-01                  Selected: BT-Sensor-02
BT-Sensor-02 (selected)        [Clear selection]
```

If each component declares its own state like this:

```kotlin
var selectedDevice by remember { mutableStateOf("") }
```

then each has a separate state holder. Giving both variables the same name does not connect them.

Selecting a device in the picker would update only the picker's state. The panel could still display “No device selected”. Clearing the panel's state would not clear the picker's selection either.

We need one shared selection, with both components displaying values derived from it.

## 2. Choose an owner that can serve both components

Suppose the composable structure is:

```text
DeviceSelectionScreen
  |
  +-- DevicePicker
  |
  +-- SelectedDevicePanel
```

Here, `DeviceSelectionScreen` is the **parent** because it calls the two child composables. The two children are siblings.

The screen is a suitable owner of `selectedDevice` because it can supply both children with the current value and the actions that change it.

For local UI state, start by finding the lowest common ancestor of the components that need to read or change it. Keep the state at least high enough to serve all of those components. Move it higher when its lifetime or business responsibilities require that.

For this example, there is no reason to put the selection in an activity or app-wide object. It belongs to this screen.

Moving state out of a child into its caller is called **state hoisting**. The child usually receives two things in its place:

- The current value to display.
- A callback for reporting the user's requested change.

For our picker, the relevant parameters are:

```kotlin
selectedDevice: String
onDeviceSelect: (String) -> Unit
```

The panel needs the same value, but its clear action needs no argument:

```kotlin
selectedDevice: String
onClearSelection: () -> Unit
```

These are parameter fragments, not standalone declarations. The complete functions are in the next section.

The screen owns the selection by retaining its state and supplying the handlers that update it. Each child reports a user action through its callback.

## 3. A complete example to try

Put this code after the `package` declaration in a Kotlin file in your existing Material 3 Compose project. Call `DeviceSelectionScreen()` inside the theme block in your activity's existing `setContent` block.

This is a standalone teaching example; do not combine it with another definition of `DeviceSelectionScreen` from the earlier lesson. It selects device names without connecting to hardware.

```kotlin
import androidx.compose.foundation.layout.Column
import androidx.compose.material3.Button
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue

@Composable
fun DeviceSelectionScreen() {
    var selectedDevice by remember { mutableStateOf("") }

    Column {
        DevicePicker(
            selectedDevice = selectedDevice,
            onDeviceSelect = { deviceName ->
                selectedDevice = deviceName
            }
        )

        SelectedDevicePanel(
            selectedDevice = selectedDevice,
            onClearSelection = {
                selectedDevice = ""
            }
        )
    }
}

@Composable
fun DevicePicker(
    selectedDevice: String,
    onDeviceSelect: (String) -> Unit
) {
    val devices = listOf("BT-Sensor-01", "BT-Sensor-02")

    Column {
        Text("Choose a device")

        devices.forEach { deviceName ->
            Button(
                onClick = {
                    onDeviceSelect(deviceName)
                }
            ) {
                Text(
                    text = if (deviceName == selectedDevice) {
                        "$deviceName (selected)"
                    } else {
                        deviceName
                    }
                )
            }
        }
    }
}

@Composable
fun SelectedDevicePanel(
    selectedDevice: String,
    onClearSelection: () -> Unit
) {
    Column {
        Text(
            text = if (selectedDevice.isEmpty()) {
                "No device selected"
            } else {
                "Selected: $selectedDevice"
            }
        )

        Button(
            onClick = onClearSelection,
            enabled = selectedDevice.isNotEmpty()
        ) {
            Text("Clear selection")
        }
    }
}
```

Try selecting `BT-Sensor-02`. Both the picker label and the panel text should reflect the selection. Then clear it from the panel. Both components should return to their unselected appearance.

There is only one `selectedDevice` state holder. The parameters with that name in the children receive the current string value; they do not create additional remembered state.

`remember` retains this state across recompositions while the screen remains in the composition. This example does not preserve the selection across activity recreation. For a simple saveable value such as this string, `rememberSaveable` is an option when restoration is required.

## 4. Trace one event through the shared owner

When the user selects `BT-Sensor-02`:

```text
1. Button invokes its onClick lambda.
2. That lambda calls onDeviceSelect("BT-Sensor-02").
3. The handler supplied by DeviceSelectionScreen receives the name.
4. That handler assigns selectedDevice = deviceName.
5. Compose schedules recomposition for code that reads the changed state.
6. Both children receive the new selectedDevice value.
7. The picker and the panel display the same selection.
```

The picker does not call the panel or change its internal variables. Their common owner supplies the information that connects them.

This gives us the pattern **state flows down, events flow up**:

```text
                   DeviceSelectionScreen
                  owns selectedDevice state
                    /                  \
          current value              current value
                  /                      \
          DevicePicker            SelectedDevicePanel
                  \                      /
            select(name)             clear()
                    \                  /
                   parent-supplied handlers
                       update the state
```

The callback function is passed **down** as an argument. The phrase “events flow up” describes what happens when the child invokes it to report an action to the owner's handler.

### An event requests a change

The child's call to `onDeviceSelect(deviceName)` does not force a particular state update. It invokes whatever behaviour the caller supplied.

For example, a parent could supply this handler instead:

```kotlin
onDeviceSelect = { deviceName ->
    println("Selection requested: $deviceName")
}
```

The click would print a message, but the displayed selection would stay unchanged because this handler does not update `selectedDevice`.

This distinction lets the owner validate a request, reject it, or perform additional work. The child reports the action; the owner decides the response.

## 5. Forward a callback through an intermediate component

As the screen grows, we may extract its layout into `DeviceSelectionContent`. It will receive the current value and forward the callbacks to the appropriate children.

Keep `DevicePicker` and `SelectedDevicePanel` from Section 3. Replace `DeviceSelectionScreen` with the following definition and add `DeviceSelectionContent`:

```kotlin
@Composable
fun DeviceSelectionScreen() {
    var selectedDevice by remember { mutableStateOf("") }

    DeviceSelectionContent(
        selectedDevice = selectedDevice,
        onDeviceSelect = { deviceName ->
            selectedDevice = deviceName
        },
        onClearSelection = {
            selectedDevice = ""
        }
    )
}

@Composable
fun DeviceSelectionContent(
    selectedDevice: String,
    onDeviceSelect: (String) -> Unit,
    onClearSelection: () -> Unit
) {
    Column {
        DevicePicker(
            selectedDevice = selectedDevice,
            onDeviceSelect = onDeviceSelect
        )

        SelectedDevicePanel(
            selectedDevice = selectedDevice,
            onClearSelection = onClearSelection
        )
    }
}
```

Focus on this argument inside `DeviceSelectionContent`:

```kotlin
onDeviceSelect = onDeviceSelect
```

The left side names `DevicePicker`'s parameter. The right side refers to the function value that `DeviceSelectionContent` received from its caller.

No callback is invoked on this line. The intermediate component passes the same function value onwards. It needs neither its own selection state nor a new copy of the handler's logic.

The structure is now:

```text
DeviceSelectionScreen       owns state and supplies handlers
  |
DeviceSelectionContent      arranges children and forwards inputs
  |
  +-- DevicePicker          reports a selected name
  +-- SelectedDevicePanel   reports a clear request
```

The child's eventual invocation still runs the function supplied by `DeviceSelectionScreen`. Each intermediate layer does not need to call back to the layer above in a separate step.

## 6. Why the handler can update state later

Consider the handler created by `DeviceSelectionScreen`:

```kotlin
onDeviceSelect = { deviceName ->
    selectedDevice = deviceName
}
```

The lambda can access `selectedDevice` from its surrounding scope. This is a **closure**: a function together with access to the surrounding variables it uses.

`DeviceSelectionScreen` finishes executing after describing its UI. That does not erase the remembered state or the active button's click handler. The UI retains a handler that can invoke the supplied callback, and that callback retains access to the state it updates.

It helps to distinguish two things the parent supplies:

| Supplied input | What the child receives |
|---|---|
| `selectedDevice` | The current string to display |
| `onDeviceSelect` | A function that can update the owner's state when invoked |

The child receives the string value, without receiving the `MutableState` holder directly. The callback provides a controlled way to request a change.

Capturing state does not freeze its original value. The handler can access the remembered state holder when it runs later.

When that observable state changes, Compose can re-run affected composables and supply updated values and handlers. Recomposition itself does not invoke a click handler merely because that handler appears in the composable's code. A handler runs when code invokes it, such as in response to a click.

## 7. When the owner becomes a ViewModel

Local screen state is enough for the example so far. A ViewModel becomes useful when screen state and event handling involve business logic, data sources, or a lifetime that should survive configuration changes.

Moving the owner does not require changing the child components' contracts. `DeviceSelectionContent` can still receive a string and two callbacks.

A common arrangement is:

```text
DeviceSelectionViewModel    owns screen state and operations
  |
DeviceSelectionRoute        connects the ViewModel to the UI
  |
DeviceSelectionContent      receives values and callbacks
  |
DevicePicker / SelectedDevicePanel
```

“Route” is a naming convention here for the composable that connects a screen to its dependencies. It is not a special Kotlin keyword.

### An explicit observable-state example

This optional example uses **Compose state inside the ViewModel**. It does not use StateFlow. That keeps the observation mechanism visible without introducing another API yet.

With the runtime imports from Section 3 and this additional import, the ViewModel can be written as:

```kotlin
import androidx.lifecycle.ViewModel

class DeviceSelectionViewModel : ViewModel() {
    var selectedDevice by mutableStateOf("")
        private set

    fun selectDevice(deviceName: String) {
        selectedDevice = deviceName
    }

    fun clearSelection() {
        selectedDevice = ""
    }
}
```

The route uses `DeviceSelectionContent` from Section 5:

```kotlin
@Composable
fun DeviceSelectionRoute(viewModel: DeviceSelectionViewModel) {
    DeviceSelectionContent(
        selectedDevice = viewModel.selectedDevice,
        onDeviceSelect = viewModel::selectDevice,
        onClearSelection = viewModel::clearSelection
    )
}
```

`selectedDevice` is backed by `mutableStateOf`, so reading it during composition allows Compose to observe its changes. The private setter keeps external callers from assigning it directly; callers use the ViewModel's operations.

To use this version, the host displays `DeviceSelectionRoute` with a lifecycle-managed ViewModel instead of displaying the local-state `DeviceSelectionScreen`. Obtain that ViewModel through the appropriate activity or navigation scope, for example with Compose's `viewModel()` integration. Do not construct a fresh `DeviceSelectionViewModel()` on each recomposition. ViewModel setup is covered in [Lesson 9](lesson-09-app-architecture-and-viewmodel.md).

A ViewModel survives configuration changes within its scope, but does not by itself preserve state across process death.

### If the ViewModel exposes StateFlow instead

When a later lesson changes the representation to `StateFlow<DeviceUiState>`, the route must collect it as Compose state. On Android, the usual shape is:

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

This is an alternative route fragment for a ViewModel with a `uiState` flow, not an extra line to add to the ViewModel above. It requires the lifecycle Compose integration and its `collectAsStateWithLifecycle` import.

Simply writing `val uiState = viewModel.uiState` would retrieve the flow object; it would not collect its values for the UI. [Lesson 10](lesson-10-stateflow-viewmodel-state.md) develops that version.

### Where should cross-screen coordination live?

If choosing a device must also clear temporary measurement data, put that decision in an owner that has responsibility for both operations. Depending on the app, that might be their common screen owner, a shared ViewModel, or business logic behind a ViewModel.

A reusable picker should still report `onDeviceSelect(deviceName)`. It does not need an unrelated screen's entire ViewModel just to announce a selection.

Choose the owner based on which state must remain consistent and how long it should live. Callback forwarding alone is not a reason to move all state to the highest level of the app.

## 8. Optional follow-up: callback contracts and asynchronous work

### Required, default, or nullable?

These are three possible parameter declarations:

| Declaration | Use when… |
|---|---|
| `onSelect: (String) -> Unit` | The caller must supply a handler |
| `onSelect: (String) -> Unit = { _ -> }` | Doing nothing is a valid default |
| `onSelect: ((String) -> Unit)? = null` | The absence of a handler has a meaning |

The main example uses required callbacks because selecting and clearing are core interactions. A required callback requires the caller to supply a function; it does not guarantee that the function performs useful work. A preview may deliberately pass an empty lambda.

A nullable callback is safely invoked with:

```kotlin
onSelect?.invoke(deviceName)
```

If absence means an action is unavailable, reflect that in the UI, such as by disabling its button. Silently ignoring an apparently available action can confuse the user.

### A callback can start work whose result arrives later

Callbacks, coroutines, and StateFlow can participate in different parts of one operation:

```text
User action
  -> callback reports the request
  -> ViewModel starts a coroutine
  -> operation runs and produces a result
  -> ViewModel updates observable state
  -> UI reflects the result
```

A coroutine does not automatically move work off the main thread. Blocking file I/O needs an appropriate dispatcher, such as `Dispatchers.IO`, or an API that handles that internally. CPU-heavy work generally belongs on `Dispatchers.Default`. A short click callback can start that operation without doing the blocking work itself.

Another pattern uses a later result callback. For example, an export button launches Android's document picker, and its `onResult` handler receives a result after the picker finishes. That result may be `null` if no document was created, such as when the user cancels. Obtaining a destination URI and writing CSV content are separate steps.

These mechanisms do not require changing the basic UI contract: a child reports an event, and the responsible owner handles it. Continue with [CSV export in Lesson 8](lesson-08-exporting-measurements-as-csv.md) and [coroutines in Lesson 12](lesson-12-coroutines-and-background-work.md) for implementation details.

## 9. Practice: decide who owns what

### Exercise 1: add a third reader

Add a `SelectionHeading` composable that displays the selected device above the picker and panel. Should it create another `remember` state variable? Where should the shared state remain?

### Exercise 2: a handler that only logs

Replace the selection handler with:

```kotlin
onDeviceSelect = { deviceName -> println(deviceName) }
```

What changes after a click? Why does the selection label stay unchanged?

### Exercise 3: follow a forwarded callback

In Section 5, which component creates the handler? Which component forwards it? Which component's click handler invokes it with a device name?

### Exercise 4: move the state owner

When changing from the local-state screen to the ViewModel route, do `DevicePicker` and `SelectedDevicePanel` need to receive a ViewModel? Explain why.

### Answers

1. `SelectionHeading` should receive the current selection as a parameter. `DeviceSelectionScreen` can remain the owner if it is the common parent of all three components. Another `remember` variable would create another state holder to keep synchronized.
2. The device name is printed. The handler never updates observable selection state, so the selected labels do not change.
3. `DeviceSelectionScreen` creates the handler, `DeviceSelectionContent` forwards it, and the button handler inside `DevicePicker` invokes it with `deviceName`.
4. No. They keep receiving values and callbacks. The route connects those callbacks to ViewModel operations, and observable state supplies updated values to the UI.

## 10. Questions to ask when designing a component

- What information does this component need to display?
- Which other components need the same information?
- Which owner can keep that shared state consistent for the required lifetime?
- What user actions should this component report?
- What arguments does each event need?
- How will the owner's updated state be observed by Compose?

Official references: [state hoisting](https://developer.android.com/develop/ui/compose/state-hoisting), [observable state in Compose](https://developer.android.com/develop/ui/compose/state), [main-safe coroutine work](https://developer.android.com/kotlin/coroutines/coroutines-adv), and [the CreateDocument activity-result contract](https://developer.android.com/reference/androidx/activity/result/contract/ActivityResultContracts.CreateDocument).
