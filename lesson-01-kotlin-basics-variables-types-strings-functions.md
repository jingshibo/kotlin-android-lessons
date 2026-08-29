# Lesson 1 — Kotlin basics: variables, types, strings, and functions

The goal of this lesson is that you can read and write simple Kotlin code comfortably.

## 1. val vs var

Kotlin has two main ways to define variables:

```kotlin
val sampleId = "S001"
var temperature = 25.3
```

val means the reference cannot be reassigned:

```kotlin
val sampleId = "S001"

sampleId = "S002"   // Error
```

var means it can change:

```kotlin
var temperature = 25.3

temperature = 26.1   // OK
```

A useful rule for Android development is:

Use val by default. Use var only when the value genuinely needs to change.

This is roughly comparable to:

```python
# Python has no direct val equivalent
sample_id = "S001"
temperature = 25.3
```

and:

```cpp
const std::string sampleId = "S001";
double temperature = 25.3;
```

## 2. Kotlin types

The common types you will encounter are:

```kotlin
val age: Int = 30

val measurement: Double = 2.54

val temperature: Float = 23.5f

val sampleName: String = "Sample 1"

val connected: Boolean = true

val letter: Char = 'A'
```

The most important ones for your research applications will probably be:

```text
Int
Long
Float
Double
String
Boolean
```

For example:

```kotlin
val repetition: Int = 5
val frequency: Double = 2.45
val sampleId: String = "U01-S03"
val isConnected: Boolean = true
```

Kotlin usually infers the type automatically, so this:

```kotlin
val repetition = 5
```

is interpreted as:

```kotlin
val repetition: Int = 5
```

Similarly:

```kotlin
val frequency = 2.45
```

becomes a Double.

So you normally don't need to write the type explicitly.

## 3. Int, Float, and Double

One small Kotlin detail is important.

```kotlin
val x = 5
```

is an Int.

```kotlin
val x = 5.0
```

is a Double.

For a Float, you add f:

```kotlin
val x = 5.0f
```

For most scientific calculations, I would generally use:

Double

unless an API or ML model specifically expects Float.

For example:

```kotlin
val sensorValue: Double = 3.742
```

## 4. Strings

Strings use double quotes:

```kotlin
val deviceName = "Microwave Sensor"
```

Kotlin has very convenient string interpolation.

Instead of something like:

```text
print("Sample: " + sample_id)
```

you can write:

```kotlin
val sampleId = "S001"

println("Sample: $sampleId")
```

You can also insert calculations:

```kotlin
val repetition = 5

println("Next repetition: ${repetition + 1}")
```

Output:

Next repetition: 6

The distinction is:

$name

for a simple variable, while:

${expression}

is used for expressions.

For example:

```kotlin
val value = 2.453

println("Measured value = $value")
println("Double value = ${value * 2}")
```

This syntax appears constantly in Android applications.

## 5. Printing

For now, use:

```text
println("Hello")
```

For example:

```kotlin
val sampleId = "S001"
val measurement = 3.52

println("Sample ID: $sampleId")
println("Measurement: $measurement")
```

Output:

- Sample ID: S001
- Measurement: 3.52

Later in Android, we normally use logging such as:

```text
Log.d("ResearchApp", "Measurement = $measurement")
```

instead of println(), but don't worry about that yet.

## 6. Basic arithmetic

The syntax is straightforward:

```kotlin
val a = 10
val b = 3

val sum = a + b
val difference = a - b
val product = a * b
val division = a / b
```

But notice this:

```kotlin
val a = 10
val b = 3

println(a / b)
```

outputs:

3

because both variables are Int.

For decimal division:

```kotlin
val result = 10.0 / 3.0
```

or:

```kotlin
val result = a.toDouble() / b
```

which gives approximately:

3.333333

Kotlin does not automatically convert numeric types as freely as Python does.

For example:

```kotlin
val x: Int = 5
val y: Double = x
```

will not work.

Instead:

```kotlin
val y: Double = x.toDouble()
```

You'll frequently see:

```text
.toInt()
.toFloat()
.toDouble()
.toLong()
.toString()
```

## 7. Functions

Functions start with fun.

A very simple function is:

```kotlin
fun sayHello() {
    println("Hello")
}
```

Call it using:

```text
sayHello()
```

A function with an input:

```kotlin
fun printMeasurement(value: Double) {
    println("Measurement: $value")
}
```

Call:

```kotlin
printMeasurement(2.45)
```

## 8. Returning values

The return type appears after the parameters.

```kotlin
fun calculateMean(a: Double, b: Double): Double {
    return (a + b) / 2
}
```

This may look slightly unusual if you're coming from C++.

C++:

```cpp
double calculateMean(double a, double b) {
    return (a + b) / 2;
}
```

Kotlin:

```kotlin
fun calculateMean(a: Double, b: Double): Double {
    return (a + b) / 2
}
```

So the pattern is:

```kotlin
fun functionName(parameter: Type): ReturnType
```

## 9. Kotlin's short function syntax

For simple functions, Kotlin lets you write:

```kotlin
fun square(x: Double): Double = x * x
```

And Kotlin can even infer the return type:

```kotlin
fun square(x: Double) = x * x
```

Both are equivalent to:

```kotlin
fun square(x: Double): Double {
    return x * x
}
```

You'll see this style often.

## 10. Example relevant to a research app

Imagine your application receives five sensor readings.

You might write:

```kotlin
fun calculateMean(
    a: Double,
    b: Double,
    c: Double,
    d: Double,
    e: Double
): Double {

    return (a + b + c + d + e) / 5.0
}
```

Then:

```kotlin
val mean = calculateMean(
    2.1,
    2.4,
    2.3,
    2.6,
    2.5
)

println("Mean measurement = $mean")
```

Output:

Mean measurement = 2.38

Later, we'll replace those five individual arguments with a List<Double>.

## 11. A small complete Kotlin program

A standalone Kotlin program can look like this:

```kotlin
fun calculateMean(a: Double, b: Double): Double {
    return (a + b) / 2.0
}

fun main() {

    val sampleId = "S001"

    var reading1 = 2.4
    var reading2 = 2.8

    val mean = calculateMean(reading1, reading2)

    println("Sample: $sampleId")
    println("Reading 1: $reading1")
    println("Reading 2: $reading2")
    println("Mean: $mean")
}
```

You don't need to memorize main() for Android development. Android doesn't normally start applications through your own main() function, but it's useful while learning basic Kotlin.

## 12. One Kotlin difference worth remembering

Kotlin generally doesn't require semicolons.

You write:

```kotlin
val x = 5
val y = 10
println(x + y)
```

rather than:

```cpp
int x = 5;
int y = 10;
```

So visually, Kotlin tends to feel closer to Python than C++.

## Your first exercise

Try to understand what this code does before looking at my explanation:

```kotlin
val sampleId = "D1-ETO-W0-U1-S1"

var repetition = 1

val measurement1 = 2.35
val measurement2 = 2.51

val mean = (measurement1 + measurement2) / 2.0

println("Sample: $sampleId")
println("Repetition: $repetition")
println("Mean: $mean")

repetition = repetition + 1

println("Next repetition: $repetition")
```

It should output approximately:

```text
Sample: D1-ETO-W0-U1-S1
Repetition: 1
Mean: 2.43
Next repetition: 2
```

The important concepts here are:

- val          immutable value
- var          mutable value

```text
Int          integer
Double       decimal number
String       text
Boolean      true/false
```

- $variable    insert variable into string
- ${...}       insert expression into string

: Double     specify type

```kotlin
fun          define function
return       return result
```

## What you need to retain from Lesson 1

Don't try to memorize every syntax detail. If these six examples already look understandable, you're ready to continue:

```kotlin
val name = "Sample 1"

var count = 0

val value: Double = 2.45

println("Value = $value")

fun square(x: Double): Double {
    return x * x
}

val result = square(value)
```

For Lesson 2, I suggest we cover the control-flow and collection features you'll use constantly in a research app:

if, when, for, while, ranges, List, MutableList, and basic array/data processing.

That will let us start writing things like “process 10 sensor measurements, reject invalid ones, calculate statistics, and determine the measurement status.”
