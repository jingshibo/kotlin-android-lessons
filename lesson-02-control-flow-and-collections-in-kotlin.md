# Lesson 2 — Control Flow and Collections in Kotlin

This lesson covers the parts of Kotlin you will use constantly when processing sensor data or controlling app logic:

```text
if
when
for
while
ranges
List
MutableList
basic array/data processing
```

By the end, you should be able to process a small set of measurements and make simple decisions from them.

## 1. if

Basic syntax:

```kotlin
val temperature = 28.5

if (temperature > 30) {
    println("High temperature")
} else {
    println("Normal temperature")
}
```

You can also use multiple conditions:

```text
if (temperature > 30) {
    println("High")
} else if (temperature > 20) {
    println("Normal")
} else {
    println("Low")
}
```

A useful difference from many languages is that if can return a value.

Instead of:

```kotlin
var status = ""

if (temperature > 30) {
    status = "High"
} else {
    status = "Normal"
}
```

you can write:

```kotlin
val status = if (temperature > 30) {
    "High"
} else {
    "Normal"
}
```

Or even:

```kotlin
val status = if (temperature > 30) "High" else "Normal"
```

This style is very common in Kotlin.

## 2. Logical operators

You will frequently use:

- &&    // AND
- ||    // OR
- !     // NOT

Example:

```kotlin
val temperature = 25.0
val connected = true

if (temperature < 30 && connected) {
    println("Ready to measure")
}
```

For OR:

```text
if (temperature > 40 || temperature < 5) {
    println("Temperature out of range")
}
```

For NOT:

```text
if (!connected) {
    println("Device disconnected")
}
```

Comparison operators are familiar:

- >
- <
- >=
- <=
- ==
- !=

For example:

```text
if (sampleId == "S001") {
    println("Correct sample")
}
```

## 3. when

when is Kotlin's more powerful version of switch.

Example:

```kotlin
val status = 2

when (status) {
    0 -> println("Idle")
    1 -> println("Measuring")
    2 -> println("Completed")
    else -> println("Unknown")
}
```

You can also return a value:

```kotlin
val message = when (status) {
    0 -> "Idle"
    1 -> "Measuring"
    2 -> "Completed"
    else -> "Unknown"
}
```

For research apps, when is useful for things like device states:

```kotlin
val deviceState = "CONNECTED"

val message = when (deviceState) {
    "CONNECTED" -> "Device ready"
    "MEASURING" -> "Measurement in progress"
    "ERROR" -> "Device error"
    else -> "Unknown state"
}

println(message)
```

## 4. when without a variable

A particularly useful form is:

```kotlin
val measurement = 2.7

val status = when {
    measurement > 3.0 -> "High"
    measurement > 2.0 -> "Normal"
    else -> "Low"
}
```

This is often cleaner than:

```text
if (...) {
} else if (...) {
} else {
}
```

## 5. Ranges

Kotlin makes numerical ranges very easy.

1..5

means:

1, 2, 3, 4, 5

You can loop through it:

```text
for (i in 1..5) {
    println(i)
}
```

Output:

```text
1
2
3
4
5
```

Be careful: .. includes the end value.

So:

1..5

contains 5.

## 6. until

If you do not want to include the final value:

```text
for (i in 0 until 5) {
    println(i)
}
```

Output:

```text
0
1
2
3
4
```

This is extremely common when working with array indexes.

## 7. step

You can change the step size:

```text
for (i in 0..10 step 2) {
    println(i)
}
```

Output:

```text
0
2
4
6
8
10
```

## 8. downTo

For reverse loops:

```text
for (i in 5 downTo 1) {
    println(i)
}
```

Output:

```text
5
4
3
2
1
```

## 9. for loops

A typical loop:

```text
for (i in 1..5) {
    println("Measurement $i")
}
```

You can also iterate directly over values rather than indexes.

For example:

```kotlin
val readings = listOf(2.1, 2.4, 2.7)

for (value in readings) {
    println(value)
}
```

This is usually preferable when you don't actually need the index.

## 10. Lists

A list stores multiple values.

```kotlin
val readings = listOf(
    2.1,
    2.3,
    2.5,
    2.4
)
```

Kotlin automatically infers this as approximately:

List<Double>

You can explicitly write:

```kotlin
val readings: List<Double> = listOf(
    2.1,
    2.3,
    2.5
)
```

A normal List is read-only.

So this will not work:

```kotlin
val readings = listOf(2.1, 2.3)
```

readings.add(2.5)   // Error

## 11. Accessing list elements

Like Python and C++, indexing starts from 0.

```kotlin
val readings = listOf(
    2.1,
    2.3,
    2.5
)

println(readings[0])
println(readings[1])
```

Output:

- 2.1
- 2.3

Useful properties include:

readings.size

```text
readings.first()
readings.last()
```

For example:

```text
println("Number of readings = ${readings.size}")
```

## 12. MutableList

When you need to add or remove measurements, use:

```kotlin
val readings = mutableListOf<Double>()
```

Then:

```text
readings.add(2.1)
readings.add(2.3)
readings.add(2.5)
```

You can also remove:

```text
readings.remove(2.3)
```

or modify an element:

readings[0] = 5.0

This distinction is important:

List<T>

means read-only access.

MutableList<T>

means the contents can be modified.

## 13. A common point of confusion: val + mutable list

You can write:

```kotlin
val readings = mutableListOf<Double>()
```

and still do:

```text
readings.add(2.4)
```

Why?

Because val means:

The variable readings cannot point to a different list.

It does not mean that the list itself cannot change.

This is valid:

```kotlin
val readings = mutableListOf<Double>()

readings.add(2.4)
readings.add(2.8)
```

But this is not:

```text
readings = mutableListOf(5.0, 6.0)
```

because you're trying to reassign the val.

## 14. Useful list operations

Kotlin has many convenient operations.

For example:

```kotlin
val readings = listOf(
    2.1,
    2.3,
    2.5,
    2.8
)
```

Mean:

```kotlin
val mean = readings.average()
```

Minimum:

```kotlin
val minimum = readings.min()
```

Maximum:

```kotlin
val maximum = readings.max()
```

Sum:

```kotlin
val total = readings.sum()
```

Number of values:

```kotlin
val count = readings.size
```

So:

```text
println("Mean = ${readings.average()}")
println("Min = ${readings.min()}")
println("Max = ${readings.max()}")
```

## 15. Filtering measurements

This is particularly useful in research applications.

Suppose:

```kotlin
val readings = listOf(
    2.1,
    -5.0,
    2.4,
    100.0,
    2.3
)
```

and you only consider measurements between 0 and 10 valid.

You can write:

```kotlin
val validReadings = readings.filter {
    // The curly braces { } define a lambda (anonymous function).
    // 'it' is a special Kotlin keyword that represents the current element being processed.
    it >= 0.0 && it <= 10.0
}
```

**Key concept**: `it` means "the current element being processed" in the lambda.

So `readings.filter { it >= 0.0 }` keeps only elements where the condition is true.

The result would be:

```text
2.1
2.4
2.3
```

This is similar to Python's list comprehension: `[x for x in readings if x >= 0.0 and x <= 10.0]`

## 16. map

map transforms every element using a lambda.

For example:

```kotlin
val readings = listOf(1.0, 2.0, 3.0)

val doubled = readings.map {
    // The lambda transforms each element:
    // 'it' refers to the current element being processed.
    it * 2
}
```

Result:

2.0, 4.0, 6.0

For example, suppose a sensor value needs calibration:

```kotlin
val calibrated = readings.map {
    it * 1.05  // 'it' is each reading, multiply by 1.05
}
```

**Key concept**: `map` creates a NEW list by transforming each element according to the lambda. The original list is unchanged.

This concept will become very useful later for preprocessing, and also in Lessons 6 and 7 when converting measurement objects to text for display or export.

## 17. forEach

Another way to loop:

readings.forEach {

```text
    println(it)
}
```

Equivalent to:

```text
for (reading in readings) {
    println(reading)
}
```

Both are common.

While you're learning, I actually recommend using:

```text
for (reading in readings)
```

first because it is more explicit.

Then gradually become comfortable with:

```text
forEach
map
filter
```

## 18. Getting both index and value

Sometimes you need both.

Example:

```kotlin
val readings = listOf(
    2.1,
    2.3,
    2.5
)

for ((index, value) in readings.withIndex()) {
    println("Reading $index = $value")
}
```

Output:

- Reading 0 = 2.1
- Reading 1 = 2.3
- Reading 2 = 2.5

If you want human-readable numbering:

```text
for ((index, value) in readings.withIndex()) {
    println("Reading ${index + 1} = $value")
}
```

## 19. while

Basic syntax:

```kotlin
var count = 0

while (count < 5) {
    println(count)

    count++
}
```

count++ means:

```text
count = count + 1
```

For an Android research application, you might conceptually see:

```text
while (isMeasuring) {
    // acquire measurement
}
```

However, later I will show you why you generally should not use a blocking while loop directly in an Android UI, because it can freeze the interface.

For now, just learn the syntax.

## 20. break

You can stop a loop:

        break

```text
for (i in 1..10) {

    if (i == 5) {
    }

    println(i)
}
```

Output:

```text
1
2
3
4
```

## 21. continue

You can skip one iteration:

        continue

```text
for (i in 1..5) {

    if (i == 3) {
    }

    println(i)
}
```

Output:

```text
1
2
4
5
```

This can be useful for rejecting invalid measurements.

## 22. Research example: filtering measurements

Suppose your device collects:

```kotlin
val readings = listOf(
    2.31,
    2.45,
    -1.0,
    2.52,
    50.0,
    2.39
)
```

You want to keep only values between:

0 and 10

A simple loop-based solution is:

```kotlin
val validReadings = mutableListOf<Double>()

for (reading in readings) {

    if (reading >= 0.0 && reading <= 10.0) {
        validReadings.add(reading)
    }
}
```

Then:

```text
println(validReadings)
```

produces approximately:

[2.31, 2.45, 2.52, 2.39]

The more Kotlin-style version is:

```kotlin
val validReadings = readings.filter {
    it >= 0.0 && it <= 10.0
}
```

Both are correct.

I recommend understanding the first one before relying heavily on the second.

## 23. Calculate the mean

Now:

```kotlin
val mean = validReadings.average()

println("Mean = $mean")
```

Then determine status:

```kotlin
val status = when {
    mean > 3.0 -> "High"
    mean >= 2.0 -> "Normal"
    else -> "Low"
}

println("Status = $status")
```

This already resembles real research processing logic.

## 24. Putting everything together

Here's a complete small example:

```kotlin
fun main() {

    val sampleId = "S001"

    val readings = listOf(
        2.31,
        2.45,
        -1.0,
        2.52,
        50.0,
        2.39
    )

    val validReadings = mutableListOf<Double>()

    for (reading in readings) {

        if (reading >= 0.0 && reading <= 10.0) {
            validReadings.add(reading)
        }
    }

    println("Sample: $sampleId")

    println("Total readings: ${readings.size}")

    println("Valid readings: ${validReadings.size}")

    if (validReadings.isNotEmpty()) {

        val mean = validReadings.average()

        val status = when {
            mean > 3.0 -> "High"
            mean >= 2.0 -> "Normal"
            else -> "Low"
        }

        println("Mean = $mean")
        println("Status = $status")

    } else {

        println("No valid measurements")
    }
}
```

Notice this:

```text
validReadings.isNotEmpty()
```

It checks whether the list contains at least one value.

The opposite is:

```text
validReadings.isEmpty()
```

## 25. A more concise Kotlin version

Once you're comfortable with the syntax, the same processing could become:

```kotlin
fun main() {

    val readings = listOf(
        2.31,
        2.45,
        -1.0,
        2.52,
        50.0,
        2.39
    )

    val validReadings = readings.filter {
        it in 0.0..10.0
    }

    if (validReadings.isNotEmpty()) {

        val mean = validReadings.average()

        val status = when {
            mean > 3.0 -> "High"
            mean >= 2.0 -> "Normal"
            else -> "Low"
        }

        println("Mean = $mean")
        println("Status = $status")
    }
}
```

Notice another useful Kotlin expression:

it in 0.0..10.0

means:

it >= 0.0 && it <= 10.0

That is very readable.

## 26. Arrays vs Lists

You'll also encounter:

Array

For example:

```kotlin
val values = arrayOf(
    1.0,
    2.0,
    3.0
)
```

and primitive arrays:

```kotlin
val values = doubleArrayOf(
    1.0,
    2.0,
    3.0
)
```

For general Android application data, you'll frequently use:

- List
- MutableList

For numerical buffers, ML input tensors, sensor processing, or performance-sensitive code, you'll often encounter:

```text
FloatArray
DoubleArray
IntArray
```

For example:

```kotlin
val modelInput = FloatArray(100)
```

creates an array containing 100 floats initialized to zero.

That will become important when we eventually work with a LiteRT model.

## What I want you to retain from Lesson 2

If these examples make sense, you know enough:

```kotlin
if (value > 5) {
    println("High")
}
val status = when {
    value > 5 -> "High"
    value > 2 -> "Normal"
    else -> "Low"
}
for (i in 1..5) {
    println(i)
}
val readings = listOf(
    2.1,
    2.3,
    2.5
)
val valid = readings.filter {
    it > 0
}
val mean = valid.average()
val readings = mutableListOf<Double>()

readings.add(2.5)
```

At this point, you already have enough Kotlin syntax to write simple sensor-processing logic.

Next I recommend we cover one of the most important Kotlin topics for Android:

classes, objects, data class, null safety (?, ?., ?:, !!), and enums.

For example, we'll turn raw values like:

- "S001"
- 2.43
- 1755701234

into a structured object:

```kotlin
data class Measurement(
    val sampleId: String,
    val value: Double,
    val timestamp: Long
)
```

That is where Kotlin will start to look much more like real Android application code.
