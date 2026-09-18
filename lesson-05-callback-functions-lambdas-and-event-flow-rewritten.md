# Lesson 5 - Understanding Callbacks, from Kotlin to Compose

This is an alternative version of [the original Lesson 5](lesson-05-callback-functions-lambdas-and-event-flow.md). It follows one example: selecting a research device.

In Lesson 4, you wrote code like this inside a composable:

```kotlin
Button(
    onClick = { measurementCount++ }
) {
    Text("Measure")
}
```

There are two blocks in this call:

- `{ measurementCount++ }` describes what to do when the button is clicked.
- `{ Text("Measure") }` describes what appears inside the button.

The first block is used as a **callback**: a function supplied to other code so that code can invoke it when appropriate. Supplying the function does not, by itself, run its body.

We will first make that idea concrete in ordinary Kotlin, then bring it back to Compose. By the end, you should be able to read a callback's type, distinguish passing from calling, identify where its arguments come from, and trace a click through a child composable into parent-owned state.

The Kotlin examples below are separate examples to run one at a time. Later snippets build on earlier definitions where stated. Section 8 supplies a complete Compose example with imports. ViewModels, StateFlow, coroutines, and navigation architecture are covered in the [companion architecture note](lesson-05-notes-callbacks-in-compose-architecture.md).

## 1. Start with a function you can call

A named function groups some work:

```kotlin
fun announceSelection() {
    println("A device was selected")
}

fun main() {
    announceSelection()
}
```

When execution reaches `announceSelection()`, Kotlin runs that function's body. The parentheses are the function call.

This function takes no arguments and returns `Unit`. `Unit` means there is no useful result for the caller to use; the purpose here is the printing action.

## 2. A function can also be a value

You already know how to put a string in a variable:

```kotlin
val deviceName = "BT-Sensor-01"
```

Kotlin also lets you put a function value in a variable:

```kotlin
fun main() {
    val selectionAction: () -> Unit = {
        println("A device was selected")
    }

    println("Before the call")
    selectionAction()
    println("After the call")
}
```

Output:

```text
Before the call
A device was selected
After the call
```

Declaring `selectionAction` creates the function value. Its body runs only when `selectionAction()` is called.

Read the declaration in three pieces:

| Piece | Meaning |
|---|---|
| `selectionAction` | The variable holding the function value |
| `() -> Unit` | The function type: no arguments, returns `Unit` |
| `{ println("A device was selected") }` | The lambda expression that creates the function value |

A **lambda expression** is a way of writing a function without declaring a function name. Here, `selectionAction` names the variable holding it.

The distinction we will use throughout the lesson is:

```kotlin
selectionAction    // The function value: you can store it or pass it.
selectionAction()  // A call: run that function's body now.
```

## 3. Pass that function into another function

A function parameter can receive a function value, just as it can receive a string:

```kotlin
fun simulateDeviceSelection(onSelect: () -> Unit) {
    println("1. Simulating a selection")
    onSelect()
    println("3. Selection handled")
}

fun main() {
    val selectionAction: () -> Unit = {
        println("2. Responding to the selection")
    }

    simulateDeviceSelection(onSelect = selectionAction)
}
```

Output:

```text
1. Simulating a selection
2. Responding to the selection
3. Selection handled
```

Follow the handoff:

1. `main` creates the function value and stores it in `selectionAction`.
2. `main` passes that value into `simulateDeviceSelection`.
3. Inside `simulateDeviceSelection`, the parameter `onSelect` refers to the supplied function value.
4. `onSelect()` invokes it, so the message beginning with `2.` is printed.

The supplied function is being used as a **callback**. `main` supplies the behaviour; `simulateDeviceSelection` chooses when to invoke it.

This also makes `simulateDeviceSelection` a **higher-order function**: a function that takes another function as an argument, or returns one.

Notice that this callback runs before `simulateDeviceSelection` returns. A callback does not have to run asynchronously or after a delay. Its receiver determines when it runs. A UI button waits for a click; our simulation calls it directly.

### Write the lambda directly at the call site

Using the same `simulateDeviceSelection` definition, we can replace `main` with:

```kotlin
fun main() {
    simulateDeviceSelection(
        onSelect = {
            println("2. Responding to the selection")
        }
    )
}
```

The intermediate variable is optional. The lambda is now supplied directly as an argument.

In `onSelect = { ... }`, the left side names the parameter declared by `simulateDeviceSelection`. The right side supplies the function value for that parameter. This is Kotlin's named-argument syntax.

**Lambda describes how the function is written. Callback describes the role it plays.**

## 4. Give the callback information about the event

So far, our callback reports only that a selection happened. Now we want it to receive the selected device's name.

Change its type from `() -> Unit` to `(String) -> Unit`:

```kotlin
fun simulateDeviceSelection(onSelect: (String) -> Unit) {
    val deviceName = "BT-Sensor-01"
    onSelect(deviceName)
}

fun main() {
    simulateDeviceSelection(
        onSelect = { selectedName ->
            println("Selected: $selectedName")
        }
    )
}
```

Output:

```text
Selected: BT-Sensor-01
```

`(String) -> Unit` means that the function requires one `String` argument and returns no useful result. The code invoking it must supply that string.

There are two different parameters here:

| Name | What it receives |
|---|---|
| `onSelect` | The function value supplied by `main` |
| `selectedName` | The string supplied when that function is invoked |

The value travels through this call:

```text
simulateDeviceSelection calls onSelect(deviceName)
                                       |
                              "BT-Sensor-01"
                                       |
The supplied lambda receives it as selectedName
```

`deviceName` and `selectedName` do not have to match. One is the variable used by the caller; the other is the lambda's local parameter name. The argument is passed by position.

### The two arrows have different jobs

Compare these fragments:

```kotlin
(String) -> Unit
{ selectedName -> println(selectedName) }
```

In the **function type**, `->` separates the input types from the return type. In the **lambda expression**, `->` separates the parameter names from the body.

Kotlin knows that `selectedName` is a `String` because the expected callback type is `(String) -> Unit`.

## 5. Use the same mechanism in a Compose button

Our simulation chose a device and immediately called the callback. A real button should call it when the user clicks.

Here is a child composable that displays one device:

```kotlin
@Composable
fun DeviceButton(
    deviceName: String,
    onSelect: (String) -> Unit
) {
    Button(
        onClick = {
            onSelect(deviceName)
        }
    ) {
        Text(deviceName)
    }
}
```

This has two levels of event handling:

| Function parameter | Type | Meaning |
|---|---|---|
| `Button`'s `onClick` | `() -> Unit` | Run some work when clicked; no argument is supplied |
| `DeviceButton`'s `onSelect` | `(String) -> Unit` | Report which device was selected |

The `onClick` lambda connects them:

```kotlin
onClick = {
    onSelect(deviceName)
}
```

When Compose runs `DeviceButton`, it supplies this lambda to `Button`. The body waits for the click. When invoked, it calls `onSelect` with the device name.

The lambda can use `deviceName` even though that variable is declared outside its body. This is called **capturing** a surrounding variable.

Capturing does not always mean freezing a value at the moment the lambda is created. For example, a captured local `var` can be read after its value changes. In this component, `deviceName` is a read-only function parameter; when Compose updates the component with another name, its click handler must reflect that updated input.

### Why the wrapper is needed

You cannot use `onClick = onSelect` here: their types differ. `Button` requires a function with no arguments, while `onSelect` requires a string.

The wrapper `{ onSelect(deviceName) }` is a no-argument function that supplies the string when it runs.

You also cannot use `onClick = onSelect(deviceName)`. That expression is a function call whose result is `Unit`, while `onClick` requires `() -> Unit`. **This version fails to compile.**

### What is the block after `Button(...)`?

It is another lambda, passed to the button's `content` parameter. Kotlin allows a lambda for the final function parameter to be written outside the parentheses. This is **trailing lambda syntax**.

The button above can also be written as:

```kotlin
Button(
    onClick = { onSelect(deviceName) },
    content = { Text(deviceName) }
)
```

`content` is composable UI content; `onClick` is ordinary event-handling code. Compose uses the content to describe the button's UI, including during recomposition. The click handler runs in response to a click.

## 6. Let the parent decide what changes

In this lesson, **parent** means the composable that calls a child composable. It does not refer to class inheritance.

`DeviceButton` knows which device it displays, but it does not need to decide how the rest of the app responds to a selection. The parent supplies that behaviour:

```kotlin
@Composable
fun DeviceSelectionScreen() {
    var selectedDevice by remember { mutableStateOf("") }

    Column {
        Text("Selected: $selectedDevice")

        DeviceButton(
            deviceName = "BT-Sensor-01",
            onSelect = { selectedName ->
                selectedDevice = selectedName
            }
        )
    }
}
```

The parent creates the state and supplies a lambda that updates it. The child invokes that lambda when clicked.

Trace one click:

```text
User clicks the button
  -> Button invokes its onClick lambda
  -> that lambda calls onSelect("BT-Sensor-01")
  -> the parent's supplied lambda receives selectedName
  -> selectedDevice is assigned "BT-Sensor-01"
  -> the state change schedules recomposition for UI that reads it
  -> the displayed selection updates
```

Writing a lambda inside the parent does not make it execute when the parent is composed. It defines the behaviour that the child can invoke.

### Which variable should change?

In this lambda:

```kotlin
onSelect = { selectedName ->
    selectedDevice = selectedName
}
```

`selectedName` is the incoming argument. Like an ordinary function parameter, it cannot be reassigned. `selectedDevice` is the parent's state variable, and that is what we update.

The callback can access `selectedDevice` because it captures the surrounding state. The child does not need direct access to that state variable.

## 7. Send state down and report events up

We can let `DeviceButton` display whether it is selected by adding an `isSelected: Boolean` parameter. The parent calculates that value from its state and passes it down.

The relationship becomes:

```text
Parent owns selectedDevice
  |
  | deviceName and isSelected go down as inputs
  v
Child displays the device
  |
  | child invokes the supplied callback with the selected name
  v
Parent's callback updates selectedDevice
```

This is what **state flows down, events flow up** means. The callback function itself is passed down to the child; invoking it lets the child report an event to the parent's handler.

Keeping the selection state in the parent lets several children share it. Moving state from a child to its caller is called **state hoisting**.

The child remains reusable: another parent could supply a callback that logs the selection or starts an operation. The child still reports the same event.

## 8. Complete Compose example

Add this code to a Kotlin file in your existing Compose project, after its `package` declaration. It uses Material 3. Display it by calling `DeviceSelectionScreen()` inside the theme block in your activity's existing `setContent` block, as in Lesson 4.

This version replaces the earlier definitions of `DeviceButton` and `DeviceSelectionScreen`; do not paste both versions into the same file. It demonstrates selection only, without connecting to physical hardware.

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
        Text(
            text = if (selectedDevice.isEmpty()) {
                "No device selected"
            } else {
                "Selected: $selectedDevice"
            }
        )

        DeviceButton(
            deviceName = "BT-Sensor-01",
            isSelected = selectedDevice == "BT-Sensor-01",
            onSelect = { selectedName ->
                selectedDevice = selectedName
            }
        )

        DeviceButton(
            deviceName = "BT-Sensor-02",
            isSelected = selectedDevice == "BT-Sensor-02",
            onSelect = { selectedName ->
                selectedDevice = selectedName
            }
        )

        Button(
            onClick = { selectedDevice = "" },
            enabled = selectedDevice.isNotEmpty()
        ) {
            Text("Clear selection")
        }
    }
}

@Composable
fun DeviceButton(
    deviceName: String,
    isSelected: Boolean,
    onSelect: (String) -> Unit
) {
    Button(
        onClick = { onSelect(deviceName) }
    ) {
        Text(
            text = if (isSelected) {
                "$deviceName (selected)"
            } else {
                deviceName
            }
        )
    }
}
```

Try these actions:

1. Initially, the screen displays “No device selected” and the clear button is disabled.
2. Tap `BT-Sensor-02`. The heading and that button's label show the selection.
3. Tap `BT-Sensor-01`. The selection moves to that device.
4. Tap “Clear selection”. Both device labels return to normal and the clear button becomes disabled.

`remember` retains state across recompositions. This example does not preserve the selection across activity recreation; that is a separate state-saving concern.

## 9. Recognize alternative syntax

Once you understand the explicit form, these shorter forms become easier to read.

### `it` names an implicit single parameter

At a call site expecting `(String) -> Unit`, these callbacks do the same work:

```kotlin
onSelect = { selectedName -> selectedDevice = selectedName }
onSelect = { selectedDevice = it }
```

These are alternative argument fragments, not two arguments to supply together. Kotlin can infer the one parameter from the expected function type, so it can name that parameter `it`.

If the handler does not need the string, make that clear with `_`:

```kotlin
onSelect = { _ -> println("A selection happened") }
```

You may also see `{ println("A selection happened") }` in this position. It still accepts the string required by the expected type; its body simply ignores it.

### `::` refers to a named function

A callback can use an existing named function instead of a lambda. With the `(String) -> Unit` version of `simulateDeviceSelection` from Section 4:

```kotlin
fun printSelectedDevice(deviceName: String) {
    println("Selected: $deviceName")
}

fun main() {
    simulateDeviceSelection(onSelect = ::printSelectedDevice)
}
```

`::printSelectedDevice` supplies a function reference. `printSelectedDevice("BT-Sensor-01")` calls the function.

If you already have a function value stored in `selectionAction`, pass `selectionAction` directly. It does not need `::`.

### Matching types allow direct forwarding

When a component receives `onClear: () -> Unit`, it can give that function directly to a button:

```kotlin
@Composable
fun ClearSelectionButton(onClear: () -> Unit) {
    Button(onClick = onClear) {
        Text("Clear selection")
    }
}
```

`onClick = { onClear() }` would also work here. The wrapper is useful if you need extra work or need to supply an argument. Direct forwarding is enough when the types match and there is nothing to add.

## 10. Practice: predict, explain, and modify

### Exercise 1: predict the order

Before running this Kotlin program, write down its output:

```kotlin
fun reportSelection(onSelect: (String) -> Unit) {
    println("B")
    onSelect("BT-Sensor-02")
    println("D")
}

fun main() {
    val handler: (String) -> Unit = { name ->
        println("C: $name")
    }

    println("A")
    reportSelection(onSelect = handler)
}
```

Then identify who supplies the function and who supplies the string.

### Exercise 2: explain the type mismatch

Inside `DeviceButton`, why does this argument fail to compile?

```kotlin
onClick = onSelect(deviceName)
```

Write the correct argument and explain when its body runs.

### Exercise 3: extend the UI

Add a third device, `BT-Sensor-03`, to the complete Compose example. It should show “(selected)” when selected, and the other devices should lose that label.

Do you need a separate state variable for each button?

### Exercise 4: distinguish event data from state

In the complete example, explain the roles of:

```text
deviceName
onSelect
selectedName
selectedDevice
```

### Answers

**1.** The output is:

```text
A
B
C: BT-Sensor-02
D
```

`main` supplies the function stored in `handler`. `reportSelection` supplies the string when it calls `onSelect`. Creating `handler` does not print `C`.

**2.** `onSelect(deviceName)` is a call returning `Unit`; `onClick` requires a function of type `() -> Unit`. The correct argument is:

```kotlin
onClick = { onSelect(deviceName) }
```

The wrapper body runs when the button invokes it in response to a click.

**3.** Add this call inside the existing `Column`:

```kotlin
DeviceButton(
    deviceName = "BT-Sensor-03",
    isSelected = selectedDevice == "BT-Sensor-03",
    onSelect = { selectedName ->
        selectedDevice = selectedName
    }
)
```

No additional state variable is needed. One `selectedDevice` value represents the single selection and determines every button's `isSelected` input.

**4.** `deviceName` is the child's input identifying its device. `onSelect` holds the callback supplied by the parent. `selectedName` receives the string when that callback runs. `selectedDevice` is the parent's observable state, updated by the callback.

## 11. Quick reference

| When reading code, ask… | In our example… |
|---|---|
| What function type is required? | `onSelect: (String) -> Unit` |
| Who supplies the behaviour? | `DeviceSelectionScreen` supplies the lambda |
| Who invokes it? | `DeviceButton`'s click handler calls `onSelect(deviceName)` |
| Who supplies the argument? | The child supplies its `deviceName` |
| What changes? | The lambda updates the parent's `selectedDevice` state |

For further detail, see the official [Kotlin function and lambda documentation](https://kotlinlang.org/docs/lambdas.html), [Compose button guide](https://developer.android.com/develop/ui/compose/components/button), and [Compose state and state-hoisting guide](https://developer.android.com/develop/ui/compose/state).

Continue with the [companion architecture note](lesson-05-notes-callbacks-in-compose-architecture.md) when you are ready to follow callbacks through multiple composables and connect them to ViewModels and asynchronous work.
