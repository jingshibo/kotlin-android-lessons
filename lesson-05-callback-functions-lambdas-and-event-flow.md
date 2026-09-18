# Lesson 5 - Callback Functions, Lambdas, and Event Flow in Jetpack Compose

In Lesson 4, you used code such as:

```kotlin
Button(
    onClick = {
        measurementCount++
    }
) {
    Text("Measure")
}
```

You also used code such as:

```kotlin
OutlinedTextField(
    value = sampleId,
    onValueChange = { newValue ->
        sampleId = newValue
    }
)
```

Both examples contain **callback functions**.

Callbacks are important in Android because an app spends much of its time waiting:

```text
waiting for a button click
waiting for text input
waiting for a navigation selection
waiting for a permission result
waiting for a file-picker result
waiting for device data
```

Instead of repeatedly asking whether something happened, we often give another component a function and say:

```text
When the event happens, call this function.
```

That function is being used as a callback.

This lesson starts with Kotlin function syntax and then connects it to Compose event handling, state hoisting, navigation, ViewModels, StateFlow, and coroutines.

---

## 1. Learning goals

By the end of this lesson, you should be able to:

- explain what a callback is
- distinguish a named function, lambda, function value, and callback
- read types such as `() -> Unit` and `(String) -> Unit`
- distinguish passing a function from calling a function
- understand who provides a callback argument
- understand `it` and explicit lambda parameter names
- create a composable that accepts callbacks
- follow an event through several composables
- explain "state flows down, events flow up"
- decide when a screen should receive a callback instead of another ViewModel
- distinguish callbacks from StateFlow and coroutines

---

## 2. Start with an ordinary function

Here is a normal named Kotlin function:

```kotlin
fun clearMeasurements() {
    println("Measurements cleared")
}
```

You call it with parentheses:

```kotlin
clearMeasurements()
```

The parentheses mean:

```text
Run this function now.
```

The function takes no arguments and returns no useful result. Its function type can be written as:

```kotlin
() -> Unit // Input -> Output
```

We will unpack that syntax shortly.

---

## 3. Lambda functions: storing a function in a variable

A **lambda** is a function without a declared name.

Kotlin allows a lambda to be stored in a variable. That means a function can also be a value.

### 3.1 A lambda with no arguments

Here is a lambda stored in a variable:

```kotlin
val clearAction: () -> Unit = {
    println("Measurements cleared")
}
```

**In this declaration**:

- `clearAction` is the name of the variable.
- `() -> Unit` is the function type of the value stored in that variable.
- `{ println("Measurements cleared") }` is the lambda.

The variable has a name, but the lambda itself does not declare a name. The whole `{ ... }` expression is the lambda.

In `() -> Unit`, `()` means the function takes no arguments, and `Unit` means it returns no useful value.

You can run the stored function with parentheses:

```kotlin
clearAction()
```

You can also use its built-in `invoke` operation:

```kotlin
clearAction.invoke()
```

These mean the same thing:

```text
clearAction()
clearAction.invoke()
```

The shorter parentheses form is normally preferred for a non-null function.

### 3.2 A lambda with one argument

Now compare `clearAction` with a lambda that takes one argument:

```kotlin
val selectDeviceAction: (String) -> Unit = { deviceName ->
    println("Selected device: $deviceName")
}
```

The type is:

```kotlin
(String) -> Unit
```

That means:

```text
This function needs one String argument.
It does something with that String.
It returns no useful result.
```

You run it by passing a `String` inside the parentheses:

```kotlin
selectDeviceAction("BT-Sensor-01")
```

You can also use `invoke`:

```kotlin
selectDeviceAction.invoke("BT-Sensor-01")
```

Again, these mean the same thing:

```text
selectDeviceAction("BT-Sensor-01")
selectDeviceAction.invoke("BT-Sensor-01")
```

The important difference is that `clearAction` does not need any input, but `selectDeviceAction` must be given a `String` when it runs.

### 3.3 Lambda type inference

The `clearAction` and `selectDeviceAction` examples write the function type explicitly before `=`.

Kotlin can also infer the function type from the lambda itself:

```kotlin
val square = { x: Int -> x * x }
```

Here, `square` is also only the name of the variable. The lambda is:

```kotlin
{ x: Int -> x * x }
```

Because the function type is not written before `=`, Kotlin must infer it from the lambda:

- `x: Int` tells Kotlin the input type.
- `x * x` is the last expression, so its result becomes the return value.
- Because `x * x` produces an `Int`, Kotlin infers the return type as `Int`.

So Kotlin infers this function type:

```text
(Int) -> Int
```

### 3.4 Comparison of the two lambda styles

The important difference between `clearAction` and `square` is where Kotlin gets the function type:

```text
clearAction: the function type explicitly:  () -> Unit
square: Kotlin infers the function type:   (Int) -> Int
```

There are also two different uses of `->`:

```text
Function type:  () -> Unit
Lambda syntax:  x: Int -> x * x
```

In a function type, `->` separates input types from the return type. In a lambda, `->` separates the lambda's parameters from its body.

Specifically, in `() -> Unit`, the arrow belongs to the function type. In `{ x: Int -> x * x }`, the arrow belongs to the lambda.

### 3.5 Lambda compared with a named function

Finally, we can also compare `square` lambda with a named function:

```kotlin
fun square(x: Int): Int {
    return x * x
}
```

The named function declares its own name with `fun square`. The lambda does not. In `val square = { x: Int -> x * x }`, `square` is the variable name, not a name declared by the lambda.

---

## 4. Lambda and callback do not mean the same thing

These words are related, but they describe different things.

| Term | Meaning |
|---|---|
| Named function | A function declared with a name, such as `fun clearMeasurements()` |
| Lambda | An anonymous function written with `{ ... }` |
| Function value | A function stored in a variable or passed as an argument |
| Callback | A function given to other code so that code can call it when appropriate |

For example:

```kotlin
val clearAction = {
    println("Measurements cleared")
}
```

This is a lambda stored as a function value.

If it is passed to a button:

```kotlin
Button(onClick = clearAction) {
    Text("Clear")
}
```

the same function value is now being used as a callback.

The important distinction is:

```text
Lambda describes how a function is written.
Callback describes the role a function is playing.
```

A callback does not have to be a lambda. A named function can also be passed as a callback.

A function that receives another function or returns a function is called a **higher-order function**. For example, `Button` is a higher-order composable because it receives function arguments such as `onClick` and `content`.

---

## 5. Reading Kotlin function types

A function type describes the inputs and output of a function.

### No input, no useful return value

```kotlin
() -> Unit
```

Read it as:

```text
A function that takes no arguments and returns Unit.
```

`Unit` means that the function performs an action but does not return a useful value. It is similar to `void` in Java, C, or C++.

Example:

```kotlin
val onClear: () -> Unit = {
    println("Clear")
}
```

### One input

```kotlin
(String) -> Unit
```

Read it as:

```text
A function that receives one String and returns Unit.
```

Example:

```kotlin
val onDeviceSelect: (String) -> Unit = { deviceName ->
    println("Selected: $deviceName")
}
```

### Two inputs

```kotlin
(String, Int) -> Unit
```

Example:

```kotlin
val onMeasurementSave: (String, Int) -> Unit = { sampleId, repetition ->
    println("Saving $sampleId repetition $repetition")
}
```

### A useful return value

```kotlin
(Double) -> Boolean
```

Example:

```kotlin
val isValidMeasurement: (Double) -> Boolean = { value ->
    value in 0.0..100.0
}
```

Most Compose event callbacks return `Unit` because they are mainly used to do something instead of calculating or returning a value. A callback can return another type, and that return is immediate and synchronous. 

### Function-type reference

| Type | Meaning | Example event |
|---|---|---|
| `() -> Unit` | No input; performs an action | Clear button clicked |
| `(String) -> Unit` | Receives one `String` | Device selected |
| `(Boolean) -> Unit` | Receives one `Boolean` | Setting toggled |
| `(TabletScreen) -> Unit` | Receives one screen enum | Navigation tab selected |
| `(String, Double) -> Unit` | Receives two values | Measurement submitted |
| `(Double) -> Boolean` | Receives a value and returns a decision | Validate a reading |

---

## 6. Passing a function versus calling it

This is the most important syntax distinction in the lesson.

Suppose a composable receives this callback:

```kotlin
@Composable
fun ClearButton(
    onClear: () -> Unit
) {
    Button(
        onClick = onClear
    ) {
        Text("Clear")
    }
}
```

This line passes the function to `Button`:

```kotlin
onClick = onClear
```

It means:

```text
Give Button this function.
Button should call it later when the user clicks.
```

This equivalent version wraps the call in another lambda:

```kotlin
Button(
    onClick = {
        onClear()
    }
) {
    Text("Clear")
}
```

The outer `{ ... }` is passed to `Button`. When that outer callback runs, it calls `onClear()`.

This version is wrong:

```kotlin
Button(
    onClick = onClear()
) {
    Text("Clear")
}
```

`onClear()` executes immediately while Compose is building the UI. Its result is `Unit`, but `onClick` needs a function of type `() -> Unit`.

In this example, `onClear` is already a variable containing a function. The rule is:

```text
onClear     -> pass the existing function value
onClear()   -> call that function now
```

A separately declared named function uses `::clearMeasurements` when it is passed as a value. The next section shows that syntax.

### When the lambda wrapper is useful

Use a wrapper when the click must perform extra work:

```kotlin
Button(
    onClick = {
        println("Clear button clicked")
        onClear()
    }
) {
    Text("Clear")
}
```

Or when the callback needs an argument:

```kotlin
Button(
    onClick = {
        onDeviceSelect("BT-Sensor-01")
    } 
) {
    Text("Select device")
}
// here you cannot write it as: onClick = onDeviceSelect("BT-Sensor-01"), which executes the function immediately instead of passing it.
```

---

## 7. Passing a named function with `::`

Suppose you have a normal named function:

```kotlin
fun clearMeasurements() {
    println("Measurements cleared")
}
```

To pass that named function as a value, use a callable reference:

```kotlin
Button(
    onClick = ::clearMeasurements
) {
    Text("Clear")
}
```

You can also use a lambda:

```kotlin
Button(
    onClick = {
        clearMeasurements()
    }
) {
    Text("Clear")
}
```

For a ViewModel member function, you may see:

```kotlin
MeasurementScreen(
    onClear = viewModel::clearMeasurements
)
```

or:

```kotlin
MeasurementScreen(
    onClear = {
        viewModel.clearMeasurements()
    }
)
```

Both can be correct.

Do not add `::` to a variable that already contains a function:

```kotlin
fun ClearButton(onClear: () -> Unit) {
    // onClear is already a function value.
}
```

Inside this function, pass `onClear`, not `::onClear`.

---

## 8. Who provides the callback argument?

Consider this composable parameter:

```kotlin
onDeviceSelect: (String) -> Unit
```

The function type requires a `String`. The code that **invokes** the callback must provide that string.

```kotlin
@Composable
fun DeviceButton(
    deviceName: String,
    onDeviceSelect: (String) -> Unit
) {
    Button(
        onClick = {
            onDeviceSelect(deviceName)
        }
    ) {
        Text(deviceName)
    }
}
```

Here, `DeviceButton` knows which device was clicked, so it supplies `deviceName`.

The parent provides the callback function and defines what to do with the supplied string value:

```kotlin
@Composable
fun DeviceParent() {
    DeviceButton(
        deviceName = "BT-Sensor-01",
        onDeviceSelect = { selectedName ->
            println("Parent received $selectedName")
        }
    )
}
```

Here, `DeviceParent` is the parent composable. It passes the callback down to `DeviceButton`.

The roles are:

```text
DeviceButton:
    produces the event
    provides the selected device name
    invokes the callback

Parent:
    provides the callback implementation
    receives the selected device name
    decides what the event should do
```

The child knows **what happened**. The parent usually knows **what that event should change**.

---

## 9. A callback parameter may be ignored

The producer may provide a value even when one particular consumer does not need it.

Suppose the child composable `DeviceScreen` emits the selected device name:

```kotlin
onDeviceSelect("BT-Sensor-01")
```

Both examples below are parent composables that call `DeviceScreen`.

One parent may use the value:

```kotlin
@Composable
fun DeviceParentThatUsesName(dataViewModel: DataViewModel) {
    DeviceScreen(
        onDeviceSelect = { selectedDeviceName ->
            println("Selected: $selectedDeviceName")
            dataViewModel.clearDataScreen()
        }
    )
}
```

Another parent may care only that the selection changed:

```kotlin
@Composable
fun DeviceParentThatIgnoresName(dataViewModel: DataViewModel) {
    DeviceScreen(
        onDeviceSelect = {
            dataViewModel.clearDataScreen()
        }
    )
}
```

The second lambda still matches `(String) -> Unit`. Kotlin knows it receives one `String`, but the lambda body does not use that parameter.

Do not remove useful event data from the child merely because one current consumer ignores it. Another consumer or future requirement may need it.

---

## 10. Understanding `it`

When a lambda has exactly one parameter, Kotlin lets you use the implicit name `it`.

```kotlin
onDeviceSelect = {
    println("Selected device: $it")
}
```

This is equivalent to:

```kotlin
onDeviceSelect = { selectedDeviceName ->
    println("Selected device: $selectedDeviceName")
}
```

For navigation:

```kotlin
onNavigate = {
    currentScreenName = it.name
}
```

is equivalent to:

```kotlin
onNavigate = { selectedScreen ->
    currentScreenName = selectedScreen.name
}
```

The parameter is not created from nowhere. Its type comes from the expected callback type:

```kotlin
onNavigate: (TabletScreen) -> Unit
```

Therefore, inside the lambda:

```text
it is a TabletScreen
```

Use `it` for a short, obvious lambda. Use an explicit name when the meaning is easier to understand with a label:

```kotlin
onSelectSessionForData = { selectedSession ->
    dataViewModel.selectPatient(selectedSession.patientId)
}
```

---

## 11. Lambda parameters are read-only

A lambda parameter works like a normal function parameter: the parameter variable itself is read-only.

This is not allowed:

```kotlin
val increase: (Int) -> Unit = { count ->
    count++
}
```

`count` is a lambda parameter, so Kotlin treats it like a `val`. To create a changed number, return a new value:

```kotlin
val increase: (Int) -> Int = { count ->
    count + 1
}
```

However, if the parameter refers to a mutable object, the object itself can still be changed:

```kotlin
val addDevice: (MutableList<String>) -> Unit = { devices ->
    devices.add("BT-Sensor-01")
}
```

Here, `devices` cannot be reassigned to a different list, but the `MutableList` object can be changed in place.

### The same rule applies inside callbacks

A callback parameter is still a lambda parameter, so it is read-only too.

Suppose a child composable reports the current count to its parent:

```kotlin
@Composable
fun CountButton(
    measurementCount: Int,
    onCountChange: (Int) -> Unit
) {
    Button(
        onClick = {
            onCountChange(measurementCount)
        }
    ) {
        Text("Measure")
    }
}
```

The parent provides the callback:

```kotlin
@Composable
fun MeasurementScreen() {
    var measurementCount by remember { mutableStateOf(0) }

    CountButton(
        measurementCount = measurementCount,
        onCountChange = { count ->
            count++
        }
    )
}
```

This does not work. `count` is the callback parameter, so it cannot be incremented.

The parent should update its own state instead:

```kotlin
@Composable
fun MeasurementScreen() {
    var measurementCount by remember { mutableStateOf(0) }

    CountButton(
        measurementCount = measurementCount,
        onCountChange = { count ->
            measurementCount = count + 1
        }
    )
}
```

Here, `count` is only the value received from the child. The change happens by assigning a new value to the parent state variable `measurementCount`.

#### Mutable objects inside callback parameters

If the callback parameter is a mutable object, the callback can mutate the object:

```kotlin
@Composable
fun MutableMeasurementListButton(
    measurements: MutableList<Int>,
    onMeasurementsChange: (MutableList<Int>) -> Unit
) {
    Button(
        onClick = {
            onMeasurementsChange(measurements)
        }
    ) {
        Text("Measure")
    }
}
```

The parent callback can change the list in place:

```kotlin
@Composable
fun MeasurementScreen() {
    val measurements = remember { mutableListOf(1, 2, 3) }

    MutableMeasurementListButton(
        measurements = measurements,
        onMeasurementsChange = { list ->
            list.add(4)
        }
    )
}
```

This is allowed by Kotlin. The parameter `list` is read-only, but the `MutableList` object can still be changed.

However, this is usually not the best Compose state pattern. If a plain `MutableList` is changed in place, Compose may not notice that the UI should recompose.

A safer Compose pattern is to keep an immutable list in state and replace it with a new list. The child receives a read-only `List`:

```kotlin
@Composable
fun ReadOnlyMeasurementListButton(
    measurements: List<Int>,
    onMeasurementsChange: (List<Int>) -> Unit
) {
    Button(
        onClick = {
            onMeasurementsChange(measurements)
        }
    ) {
        Text("Measure")
    }
}
```

Then the parent replaces the state value with a new list:

```kotlin
@Composable
fun MeasurementScreen() {
    var measurements by remember { mutableStateOf(listOf(1, 2, 3)) }

    ReadOnlyMeasurementListButton(
        measurements = measurements,
        onMeasurementsChange = { list ->
            measurements = list + 4
        }
    )
}
```

With this pattern, the state variable receives a new list value, so Compose can observe the state change.

The rule is:

```text
You cannot reassign or increment the lambda parameter itself.
You can mutate the object it refers to, if that object is mutable.
```

---

## 12. Callback registration and callback execution happen at different times

Consider:

```kotlin
Button(
    onClick = {
        measurementCount++
    }
) {
    Text("Measure")
}
```

When Compose builds the button, it does not immediately run:

```kotlin
measurementCount++
```

Instead, the sequence is:

```text
1. The composable runs.
2. The Button receives and remembers the onClick function.
3. The UI appears.
4. Time passes.
5. The user taps the button.
6. Button invokes the stored onClick function.
7. measurementCount changes.
8. Compose schedules recomposition for UI that reads that state.
```

This difference is central to callback code:

```text
The callback is written and passed now.
Its body runs when the receiver invokes it.
```

Callbacks are often called later, but "callback" does not mathematically guarantee asynchronous behavior. A regular Kotlin function could invoke a callback immediately. The receiver controls when it calls the function.

---

## 13. `onClick` is a callback

The simplified shape of `Button` is similar to:

```kotlin
@Composable
fun Button(
    onClick: () -> Unit,
    content: @Composable () -> Unit
)
```

`Button` receives two function values:

```text
onClick:
    ordinary callback for a future click

content:
    composable function describing what appears inside the button
```

When you write:

```kotlin
Button(
    onClick = {
        measurementCount++
    }
) {
    Text("Measure")
}
```

you provide both functions.

The `Text("Measure")` block is not the click behavior. It is the button's UI content.

---

## 14. Trailing lambda syntax

Kotlin allows the final lambda argument to be placed outside the parentheses.

This code:

```kotlin
Button(
    onClick = {
        measurementCount++
    }
) {
    Text("Measure")
}
```

is conceptually the same as:

```kotlin
Button(
    onClick = {
        measurementCount++
    },
    content = {
        Text("Measure")
    }
)
```

The block after `Button(...)` is the last parameter, `content`.

This also explains a custom shell:

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

        Box {
            content()
        }
    }
}
```

The caller can write:

```kotlin
TabletShell(
    activeScreen = currentScreen,
    onNavigate = { selectedScreen ->
        currentScreen = selectedScreen
    }
) {
    when (currentScreen) {
        TabletScreen.Device -> DeviceScreen()
        TabletScreen.Data -> DataScreen()
    }
}
```

The final block is passed into the `content` parameter. Inside `TabletShell`, this line invokes it:

```kotlin
content()
```

---

## 15. Creating a reusable composable with callbacks

A reusable UI component should usually receive the data it displays and callbacks for the events it can produce.

```kotlin
@Composable
fun DeviceRow(
    deviceName: String,
    isSelected: Boolean,
    onSelect: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    Row(
        modifier = modifier.clickable {
            onSelect(deviceName)
        }
    ) {
        Text(deviceName)

        if (isSelected) {
            Text("Selected")
        }
    }
}
```

The parent owns the selected device state:

```kotlin
@Composable
fun DeviceList() {
    var selectedDevice by remember {
        mutableStateOf("")
    }

    DeviceRow(
        deviceName = "BT-Sensor-01",
        isSelected = selectedDevice == "BT-Sensor-01",
        onSelect = { selectedName ->
            selectedDevice = selectedName
        }
    )
}
```

The same `DeviceRow` can be reused by another parent that handles `onSelect` differently, even if that parent does not need the selected name:

```kotlin
@Composable
fun DeviceRefreshPanel() {
    var statusMessage by remember {
        mutableStateOf("Waiting")
    }

    DeviceRow(
        deviceName = "BT-Sensor-01",
        isSelected = false,
        onSelect = {
            statusMessage = "Device selection changed"
        }
    )

    Text(statusMessage)
}
```

In `DeviceList`, `onSelect` uses the selected name to change which row is selected. In `DeviceRefreshPanel`, `onSelect` ignores the selected name and only records that a selection happened. The child composable is reusable because it reports useful event data, but each parent decides whether it needs that data.

The child does not need to know where state is stored. It only needs:

```text
data to display
callback to report an event
```

---

## 16. Lambdas can capture surrounding values

A lambda can use values declared outside it:

```kotlin
val deviceName = "BT-Sensor-01"

Button(
    onClick = {
        println(deviceName)
    }
) {
    Text("Show device")
}
```

The lambda **captures** `deviceName`.

A navigation menu can use the same idea:

```kotlin
items.forEach { item ->
    NavigationItem(
        label = item.navLabel,
        onClick = {
            onNavigate(item)
        }
    )
}
```

Each callback captures the `item` for its own iteration.

The important idea is that the callback may run later.

For example:

```kotlin
@Composable
fun DeviceButton(deviceName: String) {
    Button(
        onClick = {
            println(deviceName)
        }
    ) {
        Text(deviceName)
    }
}
```

When `DeviceButton` runs, it creates the `onClick` lambda. The lambda uses `deviceName`, even though `deviceName` was declared outside the lambda.

The button appears on screen first. Later, when the user clicks the button, the lambda runs and still has access to `deviceName`.

That is what capturing means:

```text
A lambda can remember values from the place where it was created.
The callback can use those values later, when the event happens.
```

In Compose, if `deviceName` changes, Compose may run `DeviceButton` again. This is recomposition. During recomposition, Compose may create a new `onClick` lambda that captures the new `deviceName`.

For simple event handlers, you usually do not need to manage this manually. The main thing to remember is:

```text
The lambda is created now but this callback may run later.
The values it captured during creation can be remembered and used when it runs.
```

---

## 17. Callback naming conventions

Callback parameter names commonly describe events:

```kotlin
onClick
onClear
onConnect
onDisconnect
onDeviceSelect
onSampleIdChange
onNavigate
onSubmitPatient
onExport
```

The `on...` prefix suggests:

```text
Run this function when this event occurs.
```

Use event-focused names for reusable UI:

```kotlin
onDeviceSelect
```

rather than names tied to one parent's implementation:

```kotlin
clearDataViewModelWhenDeviceChanges
```

The child should report the event. The parent decides what the event means for the wider app.

---

## 18. Common mistakes

### Mistake 1: calling the function while passing it

Incorrect:

```kotlin
Button(onClick = onClear())
```

Correct:

```kotlin
Button(onClick = onClear)
```

or:

```kotlin
Button(onClick = { onClear() })
```

### Mistake 2: forgetting a required callback argument

If the type is:

```kotlin
onDeviceSelect: (String) -> Unit
```

then the callback needs one `String` argument whenever it runs.

This is incorrect:

```kotlin
onDeviceSelect()
```

Kotlin will complain because the callback type says:

```text
To run this function, you must provide one String.
```

Provide a `String` when invoking the callback:

```kotlin
onDeviceSelect(deviceName)
```

For example:

```kotlin
@Composable
fun DeviceRow(
    deviceName: String,
    onDeviceSelect: (String) -> Unit
) {
    Button(
        onClick = {
            onDeviceSelect(deviceName)
        }
    ) {
        Text(deviceName)
    }
}
```

Here:

- `onDeviceSelect` is the callback function.
- `deviceName` is the argument passed into that callback.
- `DeviceRow` knows which device it is displaying, so it can provide `deviceName` when the button is clicked.

The parent provides this callback function and decides what to do with the selected device name:

```kotlin
DeviceRow(
    deviceName = "BT-Sensor-01",
    onDeviceSelect = { selectedDeviceName ->
        selectedDevice = selectedDeviceName
    }
)
```

The event flow is:

```text
User clicks this row
-> DeviceRow calls onDeviceSelect(deviceName)
-> parent lambda receives that value as selectedDeviceName
-> parent updates selectedDevice
```

So the function type tells you what information must be provided when the callback is invoked.

### Mistake 3: confusing where a callback is defined with where it runs

It is easy to misunderstand that this callback runs where it is written:

```kotlin
onNavigate = { selectedScreen ->
    currentScreen = selectedScreen
}
```

But in fact this code only **defines** what should happen later.

For example, the parent might write:

```kotlin
TabletShell(
    currentScreen = currentScreen,
    onNavigate = { selectedScreen ->
        currentScreen = selectedScreen
    }
)
```

At this moment, the parent creates a function value and passes it down to `TabletShell`. The screen has not changed yet merely because the lambda was written there.

Then `TabletShell` may pass the same callback farther down:

```kotlin
SideNavBar(
    currentScreen = currentScreen,
    onNavigate = onNavigate
)
```

This still does not run the callback. It only gives `SideNavBar` a function it can call later.

Finally, the child invokes the callback in response to a user event:

```kotlin
NavigationItem(
    label = screen.label,
    onClick = {
        onNavigate(screen)
    }
)
```

Now the callback actually runs. The child provides the argument `screen` back to the parent, and the parent's lambda receives it as `selectedScreen`:

```text
Child click happens
-> child calls onNavigate(screen)
-> parent lambda runs
-> selectedScreen receives the screen value
-> currentScreen = selectedScreen updates parent state
-> Compose recomposes with the new screen
```

So there are three different ideas:

| Step | What happens | Does the callback run? |
|---|---|---|
| Define | Parent writes `{ selectedScreen -> currentScreen = selectedScreen }` | No |
| Pass/register | Parent and intermediate composables pass `onNavigate` down | No |
| Invoke | Child calls `onNavigate(screen)` after a click event | Yes |

Definition location and execution location are different. The parent usually defines what the event means, but the child decides when to report that the event happened.

### Mistake 4: assuming every `{ ... }` is an event callback

Curly braces can represent many lambdas:

```kotlin
measurements.map { measurement -> measurement.value }
```

This lambda transforms each list item. It is not a UI event callback.

```kotlin
Button(onClick = { clearMeasurements() })
```

This lambda is used as an event callback.

### Mistake 5: using vague parameter names in a long lambda

This may be hard to follow:

```kotlin
onSelectSessionForData = {
    dataViewModel.selectPatient(it.patientId)
    dataViewModel.selectArm(it.arm)
}
```

An explicit name is clearer:

```kotlin
onSelectSessionForData = { selectedSession ->
    dataViewModel.selectPatient(selectedSession.patientId)
    dataViewModel.selectArm(selectedSession.arm)
}
```

---

## 19. Complete mini-example

This example combines state, a reusable child composable, a callback carrying data, and recomposition.

```kotlin
@Composable
fun ResearchApp() {
    var selectedDevice by rememberSaveable {
        mutableStateOf("")
    }

    DeviceSelectionScreen(
        devices = listOf(
            "BT-Sensor-01",
            "BT-Sensor-02"
        ),
        selectedDevice = selectedDevice,
        onDeviceSelect = { selectedName ->
            selectedDevice = selectedName
        },
        onDisconnect = {
            selectedDevice = ""
        }
    )
}

@Composable
fun DeviceSelectionScreen(
    devices: List<String>,
    selectedDevice: String,
    onDeviceSelect: (String) -> Unit,
    onDisconnect: () -> Unit
) {
    Column {
        Text(
            text = if (selectedDevice.isEmpty()) {
                "No device selected"
            } else {
                "Selected: $selectedDevice"
            }
        )

        devices.forEach { deviceName ->
            Button(
                onClick = {
                    onDeviceSelect(deviceName)
                }
            ) {
                Text(deviceName)
            }
        }

        Button(
            onClick = onDisconnect,
            enabled = selectedDevice.isNotEmpty()
        ) {
            Text("Disconnect")
        }
    }
}
```

Follow one click:

```text
User taps BT-Sensor-02
-> Button invokes its onClick callback
-> onClick calls onDeviceSelect("BT-Sensor-02")
-> ResearchApp's callback receives selectedName
-> selectedDevice becomes "BT-Sensor-02"
-> Compose recomposes
-> the text displays "Selected: BT-Sensor-02"
-> Disconnect becomes enabled
```

The child produces events, but the parent owns the state.

---

## 20. Practice exercises

### Exercise 1: read the type

Explain this parameter:

```kotlin
onPatientSelect: (Long) -> Unit
```

### Exercise 2: replace `it`

Rewrite this with an explicit parameter name:

```kotlin
onNavigate = {
    currentScreenName = it.name
}
```

### Exercise 3: pass or call?

Which version correctly gives a callback to a button?

```kotlin
Button(onClick = onClear)
Button(onClick = onClear())
```

### Exercise 4: identify the argument provider

In this code, who provides the `String`?

```kotlin
onDeviceSelect(deviceName)
```

### Exercise 5: identify the captured value

In this code, which value does the `onClick` callback capture?

```kotlin
val deviceName = "BT-Sensor-01"

Button(
    onClick = {
        println(deviceName)
    }
) {
    Text("Show device")
}
```

### Exercise 6: choose the better callback name

Which callback name is better for a reusable child composable?

```text
onDeviceSelect
clearDataViewModelWhenDeviceChanges
```

---

## 21. Exercise answers

### Answer 1

```text
onPatientSelect is a function parameter.
The function receives one Long and returns Unit.
The code invoking it must provide the Long patient ID.
```

### Answer 2

```kotlin
onNavigate = { selectedScreen ->
    currentScreenName = selectedScreen.name
}
```

### Answer 3

Correct:

```kotlin
Button(onClick = onClear)
```

`onClear()` calls the function immediately instead of passing it.

### Answer 4

The code that invokes the callback provides `deviceName`. Usually that is the child component that knows which device was selected.

### Answer 5

```text
The callback captures deviceName.
That means the lambda can still use deviceName inside its body.
```

### Answer 6

```text
onDeviceSelect is better for a reusable child composable.
It describes the event instead of one parent's implementation.
```

---

## 22. Final mental model

When you encounter callback code, ask five questions:

```text
1. What is the callback's function type?
2. Who defines the callback behavior?
3. Who invokes the callback?
4. What arguments does the invoker provide?
5. What state or operation changes when it runs?
```

For this example:

```kotlin
DeviceScreen(
    onDeviceSelect = { selectedName ->
        println(selectedName)
        dataViewModel.clearDataScreen()
    }
)
```

the answers are:

```text
Type:
    (String) -> Unit

Defined by:
    the parent calling DeviceScreen. DeviceScreen is currently called by another function.

Invoked by:
    DeviceScreen when a device is selected

Argument provided by:
    DeviceScreen, because it knows the selected device name

Effect:
    the parent receives the name and clears data-screen state
```

One confusing point is why `onDeviceSelect` is not the one providing the callback. 

Think of it like this:

```kotlin
DeviceScreen(
    onDeviceSelect = SOME_FUNCTION_FROM_PARENT
)
```

The left side:

```kotlin
onDeviceSelect =
```

is the parameter name that `DeviceScreen` declared.

The right side:

```kotlin
{ selectedName ->
    println(selectedName)
    dataViewModel.clearDataScreen()
}
```

is the actual callback function provided by the parent.

Inside `DeviceScreen`, it may look like this:

```kotlin
@Composable
fun DeviceScreen(
    onDeviceSelect: (String) -> Unit
) {
    Button(
        onClick = {
            onDeviceSelect("BT-Sensor-01")
        }
    ) {
        Text("Select device")
    }
}
```

So the full story is:

```text
DeviceScreen defines the callback parameter:
onDeviceSelect: (String) -> Unit

Parent provides the callback body:
{ selectedName -> ... }

DeviceScreen invokes the callback later:
onDeviceSelect("BT-Sensor-01")

The parent's lambda receives that value:
selectedName = "BT-Sensor-01"
```

So the best wording is probably:

```text
DeviceScreen declares the callback parameter that it needs.
The parent provides the callback implementation/body.
DeviceScreen calls that callback when a device is selected.
```

**The most important rules are:**

```text
A lambda is one way to create a function value.
A callback is a function given to other code to invoke.

onClear passes an existing function value.
onClear() calls that function.
::clearMeasurements passes a reference to a named function.
clearMeasurements() calls the named function.

The callback invoker provides the required arguments.
The callback implementation decides what to do with them.
```

With this model, code such as `onClick`, `onValueChange`, `onNavigate`, `onResult`, and `onDeviceSelect` follows the same underlying idea.

For the larger Compose architecture pattern built from these callbacks, see [Lesson 5 notes - callbacks in Compose architecture](lesson-05-notes-callbacks-in-compose-architecture.md).
