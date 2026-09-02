# Lesson 6 Notes - val, var, remember, and Recomposition

This note explains a confusing but very important Compose idea:

```text
Why do some changing-looking values use val,
while other UI values use var?
```

The short answer:

- Use `val` when the variable is calculated once during the current run of the composable and is not reassigned inside that run.
- Use `var` when your code needs to manually assign a new value later using `=`.
- Use `remember` when a value needs to survive recomposition.

This note is organized in this order:

1. Understand why changing-looking calculated values can still use `val`.
2. Connect that idea to recomposition: the composable can run again.
3. Learn why reassigned UI state needs `var` and `remember`.
4. Compare `val` without `remember`, `val` with `remember`, and `var` with `remember`.
5. Understand when recomposition reruns calculations.
6. Use the decision rule to choose `val`, `var`, and `remember` in your own code.

## 1. The confusing example

In Lesson 6, we calculate values like this:

```kotlin
val latestMeasurement = measurementList.lastOrNull()

val meanText = if (valueList.isNotEmpty()) {
    "%.3f".format(valueList.average())
} else {
    "--"
}

val minText = valueList.minOrNull()?.let {
    "%.3f".format(it)
} ?: "--"

val maxText = valueList.maxOrNull()?.let {
    "%.3f".format(it)
} ?: "--"
```

At first, this can feel strange.

The latest measurement, mean, min, and max can all change when the user adds more measurements. So why are they `val` instead of `var`?

## 2. Why these calculated values use val

In Kotlin, `val` means:

```text
This variable cannot be reassigned inside this same block of code.
```

It does not mean:

```text
This value can never be different when the function runs again.
```

That distinction matters a lot in Compose.

During one run of `ResearchScreen`, these values are calculated and then only displayed. They are not reassigned later in the same function run.

For example:

```kotlin
val meanText = if (valueList.isNotEmpty()) {
    "%.3f".format(valueList.average())
} else {
    "--"
}
```

After `meanText` is calculated, we do not do this:

```kotlin
meanText = "some new value"
```

So it should be a `val`.

## 3. But the displayed mean still changes

The mean can still change over time because Compose reruns the composable function when state changes.

For example:

```text
User clicks Measure
    -> measurementList changes
    -> Compose recomposes/rerun ResearchScreen
    -> valueList is calculated again
    -> meanText is calculated again
    -> the UI displays the new mean
```

So `meanText` is a `val` for one specific recomposition.

On the next recomposition, Compose creates a new `meanText` based on the new data.

## 4. A useful mental model

Think of a composable function as a recipe for drawing the current UI.

When the state changes, Compose follows the recipe again.

During each run:

```text
calculate latestMeasurement
calculate valueList
calculate meanText
calculate minText
calculate maxText
draw the UI for those values
```

Those calculated values can be `val` because each one is fixed for that single run.

Then, when the data changes, Compose runs the recipe again and calculates fresh `val` values.

## 5. Why other values need var

Now compare those calculated values with state values like these:

```kotlin
var sampleId by remember {
    mutableStateOf("")
}

var sampleName by remember {
    mutableStateOf("")
}

var isConnected by remember {
    mutableStateOf(false)
}

var measurementValue by remember {
    mutableStateOf<Double?>(null)
}

var measurementCount by remember {
    mutableIntStateOf(0)
}
```

These use `var` because the code manually changes them later.

For example, when the user types into a text field:

```kotlin
OutlinedTextField(
    value = sampleId,
    onValueChange = {
        sampleId = it
    }
)
```

This line requires `sampleId` to be a `var`:

```kotlin
sampleId = it
```

If `sampleId` were a `val`, Kotlin would not allow this reassignment.

## 6. More examples of why var is needed

For a connect button:

```kotlin
Button(
    onClick = {
        isConnected = !isConnected
    }
) {
    Text("Connect")
}
```

This changes `isConnected`, so `isConnected` must be a `var`.

For a measurement button:

```kotlin
Button(
    onClick = {
        measurementValue = Random.nextDouble(0.0, 5.0)
        measurementCount += 1
    }
) {
    Text("Start Measurement")
}
```

These lines reassign state:

```kotlin
measurementValue = Random.nextDouble(0.0, 5.0)
measurementCount += 1
```

`measurementCount += 1` is just a shorter way of writing:

```kotlin
measurementCount = measurementCount + 1
```

So both values need to be `var`.

## 7. Why the variables do not reset every time

If Compose reruns the composable function, this question naturally appears:

```text
Why does sampleId not reset to an empty string every time the UI updates?
```

The reason is `remember`.

This:

```kotlin
var sampleId by remember {
    mutableStateOf("")
}
```

means:

```text
Create this state the first time the composable runs.
Keep it remembered across recompositions.
When the composable runs again, give me the remembered value.
```

Without `remember`, a normal local variable would be recreated every time.

For example:

```kotlin
var count = 0
```

This would reset to `0` whenever the function runs again.

But this:

```kotlin
var count by remember {
    mutableIntStateOf(0)
}
```

survives recomposition.

## 8. Why measurementList can be val

A Compose state list is often declared like this:

```kotlin
val measurementList = remember {
    mutableStateListOf<Measurement>()
}
```

This can be `val` even though the list contents change.

The reason is that we are not replacing the list variable with a different list.

We are changing the contents of the same remembered list:

```kotlin
measurementList.add(newMeasurement)
measurementList.clear()
```

This is allowed because `val` only prevents reassignment of the variable itself.

It prevents this:

```kotlin
measurementList = mutableStateListOf()
```

But it does not prevent changing the contents of a mutable object.

So the pattern is:

```text
val measurementList
    -> the list object stays the same
    -> the contents inside the list can change
```

**You can think of a list as a bag:**
```text
   -> The bag itself does not change
   -> The content in the bag can change
```

But:

```text
var sampleId
    -> the whole String value is replaced when the user types
```

## 9. val without remember vs val with remember vs var with remember

These three forms mean different things.

### val without remember

Use this for temporary calculated values during one recomposition:

```kotlin
val latestMeasurement = measurementList.lastOrNull()
val valueList = measurementList.map { it.value }
val meanText = if (valueList.isNotEmpty()) {
    "%.3f".format(valueList.average())
} else {
    "--"
}
```

These values are recalculated when the composable reruns.

### val with remember

Use this when the object itself should survive recomposition, but you do not replace the object:

```kotlin
val measurementList = remember {
    mutableStateListOf<Measurement>()
}
```

The list object is remembered. You can still add or remove items inside it.

### var with remember

Use this when the value must survive recomposition and your code will reassign it:

```kotlin
var sampleId by remember {
    mutableStateOf("")
}

var isConnected by remember {
    mutableStateOf(false)
}
```

These values are remembered, and your code can replace them with new values.

## 10. Does recomposition rerun all the code?

Conceptually, yes:

```text
When relevant state changes, Compose reruns the composable function
so it can describe the new UI.
```

This is called recomposition.

For learning, it is useful to imagine that the function starts again from the top:

```text
valueList is recalculated
meanText is recalculated
minText is recalculated
maxText is recalculated
the UI is described again
```

Compose is also smart. It can skip parts of the UI that do not need to change, so recomposition is usually efficient.

But there is an important detail:

```text
If ResearchScreen itself recomposes,
normal code inside ResearchScreen runs again.
```

So if `meanText` is written directly inside `ResearchScreen`, it is recalculated whenever `ResearchScreen` recomposes, even if the measurement list did not change.

For example, typing in `sampleId` might cause `ResearchScreen` to recompose. In that case, this code may run again too:

```kotlin
val valueList = measurementList.map {
    it.value
}

val meanText = if (valueList.isNotEmpty()) {
    "%.3f".format(valueList.average())
} else {
    "--"
}
```

That does not mean Compose is inefficient. It just means simple local calculations are normal Kotlin code, so they run when the function runs.

For simple calculations like formatting a mean, min, or max, there is usually no need to worry.

## 11. When to use remember for calculations

Most small calculations are fine directly inside the composable:

```kotlin
val valueList = measurementList.map {
    it.value
}

val meanText = if (valueList.isNotEmpty()) {
    "%.3f".format(valueList.average())
} else {
    "--"
}
```

But if a calculation is expensive, you can remember the result and only recalculate when the input changes.

For example:

```kotlin
val meanText = remember(measurementList) {
    val valueList = measurementList.map {
        it.value
    }

    if (valueList.isNotEmpty()) {
        "%.3f".format(valueList.average())
    } else {
        "--"
    }
}
```

This tells Compose:

```text
Only recalculate meanText when measurementList changes.
```

For the beginner version of the app, the direct calculation is easier to read and completely fine.

## 12. Quick decision rule

How to decide if you should use val or var? Ask this question:

```text
Will I assign a new value to this variable later using = ?
```

If yes, use `var`.

Example:

```kotlin
sampleId = it
isConnected = !isConnected
measurementValue = Random.nextDouble(0.0, 5.0)
measurementCount = measurementCount + 1
```

If no, use `val`.

Example:

```kotlin
val latestMeasurement = measurementList.lastOrNull()
val meanText = ...
val minText = ...
val maxText = ...
```

Then ask:

```text
Does this value need to survive recomposition?
```

If yes, use `remember`.

If no, let it be recalculated.

## 13. What to remember

The key idea is:

```text
Changing over time does not automatically mean var.
```

A value can change over time because the whole composable is rerun and a new `val` is calculated.

Use:

- `val` for calculated results that are not reassigned during the current function run.
- `var` for values your code will manually reassign.
- `remember` for state that must survive recomposition.
- `mutableStateOf` or `mutableIntStateOf` for state changes that should trigger UI updates.
- `mutableStateListOf` for a list whose content changes should trigger UI updates.

Final mental model:

```text
State changes
    -> Compose recomposes
    -> calculated val values are created again
    -> remembered var state keeps its previous value
    -> the UI displays the latest state
```
## Appendix - Original Uploaded Text, Formatted

This appendix keeps the original Q/A material for future reference, but formats it in the same Markdown and Kotlin style as the lesson above.

### Appendix Question 1: Why do these variables use val, not var?

Question:

```kotlin
val latestMeasurement = measurementList.lastOrNull()

// values list might be an empty list, but not null,
// so we cannot use ?.let here since values is not null.
val meanText = if (valueList.isNotEmpty()) {
    "%.3f".format(valueList.average())
} else {
    "--"
}

// minOrNull() returns Double or null if the values list is empty.
val minText = valueList.minOrNull()?.let {
    "%.3f".format(it)
} ?: "--"

// maxOrNull() returns Double or null if the values list is empty.
val maxText = valueList.maxOrNull()?.let {
    "%.3f".format(it)
} ?: "--"
```

Answer:

In Kotlin, the choice between `val` and `var` depends on whether you intend to reassign the variable within the same block of code.

#### Short Answer

We use `val` because these variables are read-only constants for the duration of a single UI refresh, or recomposition.

Once they are calculated at the start of the function, they never need to be changed to a different value before the function finishes.

#### How It Works In Compose

It might feel like they should be `var` because the mean or min values change as you add measurements.

However, here is what actually happens:

1. Recomposition: When you add a measurement, the `measurementList` changes. This triggers Compose to rerun the entire `ResearchScreen` function from the top.
2. New execution: During this new run, the function calculates the mean, min, and max again based on the updated list.
3. Local constants: For this specific run, `meanText` is calculated once and stays that way until the function finishes drawing the UI. Therefore, it is a `val`.

#### Comparison

| Keyword | Use case | Example in your code |
|---|---|---|
| `var` | Things you manually change using `=` later in the code | `sampleId`, `measurementValue` |
| `val` | Things that are calculated results or fixed data | `meanText`, `latestMeasurement` |

#### Why val Is Better

In Kotlin, we prefer `val` unless `var` is necessary.

- Safety: You cannot accidentally overwrite a `val` later in the function.
- Readability: When a developer sees `val`, they know this value is set once and they do not need to search for later reassignment.

Summary:

Even though the mean changes over time, each specific frame of your UI uses a single, unchanging value for that mean. Every time the data changes, a brand new frame is built with a brand new `val`.

### Appendix Question 2: Does each UI update rerun the code?

Question:

```text
So each update of the UI would rerun all the code involved?
```

Answer:

Yes. This is one of the most important concepts in Jetpack Compose, and it is called recomposition.

Think of your `@Composable` function not as a setup script, but as a recipe that the phone follows every time it needs to draw the screen.

#### How The Rerun Works

Whenever a piece of state changes, such as `sampleId` or `measurementList`, Compose needs to work out what the new UI should look like.

It can run the composable code again:

1. It recalculates `valueList`.
2. It recalculates `meanText`, `minText`, and `maxText`.
3. It checks the `if`/`else` statements to see which text should be shown.

#### If It Reruns, Why Do Variables Not Reset To 0?

This is where `remember` comes in.

If you wrote this:

```kotlin
var count = 0
```

it would reset to `0` every time the UI updated.

But if you write this:

```kotlin
var count by remember {
    mutableStateOf(0)
}
```

you are telling Compose:

```text
Store this value outside the normal rerun.
When the function reruns, give me the remembered value instead of creating a new one.
```

#### Is This Inefficient?

Compose is smart.

For simple things like `average()` or `format()`, modern phones can do the calculation very quickly.

Compose can also skip UI work when parts of the UI do not need to change.

#### When To Worry About The Rerun

If you have a very heavy calculation, such as processing a high-resolution image or a 10,000-item list, you might not want it to rerun every time the user types one letter.

In those cases, you can remember the calculation too:

```kotlin
// This only recalculates if the list itself changes,
// not when the user types in the Sample ID box.
val meanText = remember(measurementList) {
    if (valueList.isNotEmpty()) {
        "%.3f".format(valueList.average())
    } else {
        "--"
    }
}
```

Summary:

- Top-level code reruns when relevant state changes.
- `remember` is the memory that lets data survive the rerun.
- Recomposition is the process of rerunning the function to stay in sync with the data.

### Appendix Question 3: Why do these values need to be var?

Question:

```kotlin
var sampleId by remember {
    mutableStateOf("")
}

var sampleName by remember {
    mutableStateOf("")
}

var isConnected by remember {
    mutableStateOf(false)
}

// Dynamic state:
// Create a variable that starts at null.
// Remember it so it does not reset when the screen redraws.
// Make it state so that changes trigger the UI to update.
// The by keyword allows us to use it like a normal variable.
var measurementValue by remember {
    mutableStateOf<Double?>(null)
}

var measurementCount by remember {
    mutableIntStateOf(0)
}
```

Answer:

These variables must be `var` because you manually change their values later in the code using the equals sign, `=`.

In Kotlin, if you define something with `val`, you can never say:

```kotlin
x = somethingElse
```

later.

Since these variables represent inputs or settings that change, they need that flexibility.

#### User Input

When the user types in the text field, the `onValueChange` callback runs.

You need to overwrite the old string with the new string:

```kotlin
onValueChange = {
    sampleId = it
}
```

This line requires `sampleId` to be a `var`.

#### Interaction

When the user clicks the Connect button, you flip the Boolean value:

```kotlin
onClick = {
    isConnected = !isConnected
}
```

Changing from `false` to `true` requires `var`.

#### Logic Results

When the user clicks Start Measurement, you generate a new random number and increment the count:

```kotlin
onClick = {
    measurementValue = Random.nextDouble(...)
    measurementCount += 1
}
```

This:

```kotlin
measurementCount += 1
```

is the same as:

```kotlin
measurementCount = measurementCount + 1
```

So `measurementCount` must be a `var`.

#### The by Detail

You might notice that `measurementList` uses `val`, even though you add items to it.

```kotlin
val measurementList = remember {
    mutableStateListOf<Measurement>()
}
```

That is because you are not replacing the list itself. You are only adding things into the same list object.

By contrast, `sampleId` is different. You replace the entire string with a brand new string every time a letter is typed.

Summary:

- Use `var + by remember` for values that you manually overwrite using `=`.
- Use `val + remember` for objects, like lists, where you change the contents but never replace the object itself.
- Use `val` without `remember` for temporary calculations that only exist during the current UI draw.
