# Lesson 3 — Classes, Data Classes, Null Safety, and Enums

This lesson is especially important because these features appear constantly in Android code.

We’ll cover:

- classes and objects
- constructors
- data class
- nullable types with ?

safe calls ?.

Elvis operator ?:

- non-null assertion !!
- enum class

## 1. Classes and objects

A class defines a type of object.

For example, suppose every measurement has:

- a sample ID
- a measured value

You can define:

```kotlin
class Measurement(
    val sampleId: String,
    val value: Double
)
```

Then create an object:

```kotlin
val measurement = Measurement(
    "S001",
    2.45
)
```

You can access its properties using .:

```text
println(measurement.sampleId)
println(measurement.value)
```

Output:

- S001
- 2.45

## 2. Constructor syntax

Notice that Kotlin lets you define the constructor directly in the class declaration:

```kotlin
class Measurement(
    val sampleId: String,
    val value: Double
)
```

This is roughly equivalent to much more verbose C++ code such as:

```cpp
class Measurement {
public:
    std::string sampleId;
    double value;

    Measurement(std::string id, double v) {
        sampleId = id;
        value = v;
    }
};
```

Kotlin eliminates a lot of boilerplate.

## 3. val and var inside a class

You can choose whether properties can change.

```kotlin
class Measurement(
    val sampleId: String,
    var value: Double
)
```

Now:

```kotlin
val measurement = Measurement("S001", 2.45)

measurement.value = 2.60
```

is valid.

But:

```text
measurement.sampleId = "S002"
```

is not, because sampleId was declared as val.

Again, the general rule is:

Prefer val unless the property genuinely needs to change.

## 4. Functions inside classes

Classes can contain functions.

```kotlin
class Measurement(
    val sampleId: String,
    val value: Double
) {

    fun printInfo() {
        println("Sample: $sampleId")
        println("Value: $value")
    }
}
```

Then:

```kotlin
val measurement = Measurement("S001", 2.45)

measurement.printInfo()
```

## 5. data class

For research data, you'll very often want a class that mainly stores information.

Kotlin provides:

data class

Example:

```kotlin
data class Measurement(
    val sampleId: String,
    val value: Double,
    val timestamp: Long
)
```

Create one:

```kotlin
val measurement = Measurement(
    sampleId = "S001",
    value = 2.45,
    timestamp = 1755760000
)
```

Notice that I used:

```text
sampleId = "S001"
value = 2.45
timestamp = ...
```

These are called named arguments.

They make code much easier to read.

Instead of:

```kotlin
Measurement("S001", 2.45, 1755760000)
```

you can write:

```kotlin
Measurement(
    sampleId = "S001",
    value = 2.45,
    timestamp = 1755760000
)
```

I strongly recommend this style when a constructor has several parameters.

## Why use data class?

Suppose:

```kotlin
data class Measurement(
    val sampleId: String,
    val value: Double
)
```

Then Kotlin automatically gives you useful functionality such as:

- readable toString()
- equality comparison

```text
copy()
```

For example:

```kotlin
val m1 = Measurement("S001", 2.45)

println(m1)
```

produces something like:

```kotlin
Measurement(sampleId=S001, value=2.45)
```

With a normal class, printing the object would not automatically give such useful output.

## 6. Comparing data classes

Consider:

```kotlin
val m1 = Measurement("S001", 2.45)
val m2 = Measurement("S001", 2.45)
```

With a data class:

```text
println(m1 == m2)
```

returns:

```text
true
```

because Kotlin compares their stored values.

This is very useful.

## 7. copy()

Another useful data class feature:

```kotlin
val original = Measurement(
    sampleId = "S001",
    value = 2.45
)
```

You can make a modified copy:

```kotlin
val corrected = original.copy(
    value = 2.50
)
```

Now:

```text
original.value   = 2.45
corrected.value  = 2.50
```

while the other properties remain the same.

You'll see copy() quite often in modern Android development.

## 8. A more realistic research model

For example:

```kotlin
data class Measurement(
    val sampleId: String,
    val repetition: Int,
    val value: Double,
    val timestamp: Long
)
```

Then:

```kotlin
val measurement = Measurement(
    sampleId = "D1-ETO-W0-U1-S1",
    repetition = 3,
    value = 2.47,
    timestamp = System.currentTimeMillis()
)
```

System.currentTimeMillis() gives the current Unix timestamp in milliseconds.

You don't need to worry about timestamps deeply yet.

### Extending the Measurement data class

As your research app grows, you may want to add more fields to track additional information:

```kotlin
// You can add a status enum field to classify measurement results
enum class MeasurementStatus {
    LOW,
    NORMAL,
    HIGH
}

data class Measurement(
    val sampleId: String,
    val repetition: Int,
    val value: Double,
    val timestamp: Long,
    val status: MeasurementStatus  // New field!
)
```

Then:

```kotlin
val measurement = Measurement(
    sampleId = "D1-ETO-W0-U1-S1",
    repetition = 3,
    value = 2.47,
    timestamp = System.currentTimeMillis(),
    status = MeasurementStatus.NORMAL
)
```

This pattern makes your data model richer without changing the core structure. You can always add more fields as your research needs grow.

## 9. enum class

Suppose your measurement device can be in one of four states:

```text
Disconnected
Connected
Measuring
Error
```

You could use strings:

```kotlin
val status = "CONNECTED"
```

but strings are error-prone.

Someone might accidentally write:

```kotlin
val status = "CONECTED"
```

Instead, Kotlin provides enums:

    ERROR

```kotlin
enum class DeviceState {
    DISCONNECTED,
    CONNECTED,
    MEASURING,
}
```

## Important: Enum constants are not strings, but can be displayed as strings

The items in an enum class (`DISCONNECTED`, `CONNECTED`, `MEASURING`) are **enum constants** of type `DeviceState`, not `String` values. However, they can be displayed directly in text because Kotlin automatically converts them to their name:

```kotlin
val state = DeviceState.CONNECTED
Text(text = "Device status: $state")  // Displays: "Device status: CONNECTED"
```

If you need the name explicitly as a `String`:

```kotlin
val text = state.name  // Returns "CONNECTED" as a String
```

For more readable UI text, use a `when` expression to map enum constants to custom labels:

```kotlin
val displayText = when (state) {
    DeviceState.DISCONNECTED -> "Disconnected"
    DeviceState.CONNECTED -> "Connected"
    DeviceState.MEASURING -> "Measuring"
}
```

Now:

```kotlin
var state = DeviceState.DISCONNECTED
```

Later:

```text
state = DeviceState.CONNECTED
```

## 10. when + enum

Enums work beautifully with when.

```kotlin
val message = when (state) {

    DeviceState.DISCONNECTED ->
        "Device not connected"

    DeviceState.CONNECTED ->
        "Device ready"

    DeviceState.MEASURING ->
        "Measurement in progress"

    DeviceState.ERROR ->
        "Device error"
}
```

Notice something interesting:

There is no:

```text
else
```

because Kotlin knows you've handled every possible value of DeviceState.

That's safer than using arbitrary strings.

## 11. Research example

Let's combine everything.

    ERROR

```kotlin
enum class MeasurementStatus {
    WAITING,
    RECORDING,
    COMPLETE,
}

data class Measurement(
    val sampleId: String,
    val repetition: Int,
    val value: Double?,
    val status: MeasurementStatus
)
```

Notice:

```kotlin
val value: Double?
```

because there may not be a measurement yet.

Now:

```kotlin
val measurement = Measurement(
    sampleId = "S001",
    repetition = 1,
    value = null,
    status = MeasurementStatus.WAITING
)
```

You can display the value safely:

```kotlin
val displayedValue =
    measurement.value?.toString() ?: "No measurement"
```

And state:

```kotlin
val statusMessage = when (measurement.status) {

    MeasurementStatus.WAITING ->
        "Waiting"

    MeasurementStatus.RECORDING ->
        "Recording"

    MeasurementStatus.COMPLETE ->
        "Complete"

    MeasurementStatus.ERROR ->
        "Error"
}
```

This looks much more like real Android application code.

## 12. A complete example

    MEASURING

```kotlin
enum class DeviceState {
    DISCONNECTED,
    CONNECTED,
}

data class Measurement(
    val sampleId: String,
    val value: Double?,
    val repetition: Int
)

fun main() {

    var deviceState = DeviceState.CONNECTED

    val measurement = Measurement(
        sampleId = "S001",
        value = 2.43,
        repetition = 1
    )

    val deviceMessage = when (deviceState) {
        DeviceState.DISCONNECTED -> "Device disconnected"
        DeviceState.CONNECTED -> "Device ready"
        DeviceState.MEASURING -> "Measurement in progress"
    }

    val displayedValue =
        measurement.value?.toString() ?: "No data"

    println(deviceMessage)
    println("Sample: ${measurement.sampleId}")
    println("Repetition: ${measurement.repetition}")
    println("Value: $displayedValue")
}
```

Output:

```text
Device ready
Sample: S001
Repetition: 1
Value: 2.43
```

## 13. Lists of objects

Now combine Lesson 2 with Lesson 3.

```kotlin
val measurements = listOf(
    Measurement("S001", 1, 2.31, 1000),
    Measurement("S001", 2, 2.45, 2000),
    Measurement("S001", 3, 2.52, 3000)
)
```

You can loop:

```text
for (measurement in measurements) {
    println(measurement.value)
}
```

Or calculate an average:

```kotlin
val mean = measurements
    .map { it.value }
    .average()
```

Let's break that down.

First:

measurements.map { it.value }

turns:

```text
Measurement
Measurement
Measurement
```

into:

```text
2.31
2.45
2.52
```

Then:

```text
.average()
```

calculates the average.

This style becomes very common.

## 14. Null safety

Now we get to one of the most important Kotlin concepts.

Suppose you write:

```kotlin
var deviceName: String = "Sensor A"
```

Kotlin will not let you do:

```text
deviceName = null
```

because String means:

this value must contain a String.

If a variable is allowed to be missing, use:

String?

For example:

```kotlin
var deviceName: String? = null
```

The ? means:

this variable may contain either a String or null.

## 15. Why this matters in Android

Android code constantly deals with things that might not exist yet:

```text
Bluetooth device might not be connected
sensor reading might not have arrived
user might not have entered a sample ID
file might not exist
database query might return nothing
```

So you'll encounter nullable types all the time.

For example:

```kotlin
var latestReading: Double? = null
```

Before data arrives:

```text
latestReading = null
```

After a measurement:

```text
latestReading = 2.45
```

## 16. Kotlin protects you from null errors

Suppose:

```kotlin
val name: String? = null
```

This won't compile:

```text
println(name.length)
```

Why?

Because Kotlin says:

name might be null, so calling .length could crash.

You must explicitly handle that possibility.

This is a major reason Kotlin is safer than Java.

## 17. Safe-call operator ?.

You can write:

```text
println(name?.length)
```

The operator:

?.

means:

Call this function/property only if the object is not null.

If:

```text
name = "Sensor"
```

then:

name?.length

returns:

6

If:

```text
name = null
```

then:

name?.length

returns:

```text
null
```

rather than crashing.

## 18. A very common Android pattern

Suppose:

```kotlin
var deviceName: String? = null
```

You can write:

```text
println(deviceName?.uppercase())
```

If the device exists:

SENSOR A

If it is null:

```text
null
```

## 19. Elvis operator ?:

Usually you don't want to display null.

You might want a default value.

That's what this operator does:

?:

Example:

```kotlin
val name: String? = null

val displayedName = name ?: "Unknown device"
```

This means:

Use name if it isn't null; otherwise use "Unknown device".

So:

```text
println(displayedName)
```

outputs:

Unknown device

This operator is called the Elvis operator, because:

?:

apparently looks a little like Elvis Presley's hair and eyes when turned sideways.

You will see it frequently.

## 20. Combining ?. and ?:

This is very common:

```kotlin
val name: String? = null

val length = name?.length ?: 0
```

Read it as:

If name exists, get its length. Otherwise use 0.

For Android research code:

```kotlin
val displayValue = latestReading?.toString() ?: "No data"
```

Very useful.

## 21. Standard if null check

You can also do:

```text
if (deviceName != null) {
    println(deviceName.length)
}
```

Inside this block Kotlin understands that deviceName isn't null.

This is called smart casting.

You don't have to manually convert it from:

String?

to:

String

## 22. !! — non-null assertion

You'll also encounter:

!!

Example:

```kotlin
val name: String? = "Sensor"

println(name!!.length)
```

!! means:

I promise this is not null.

But if it actually is null:

```kotlin
val name: String? = null

println(name!!.length)
```

your application can crash.

So my recommendation is:

Avoid !! whenever possible.

Prefer:

?.

or:

?:

or an explicit null check.

For example, prefer:

```kotlin
val deviceName = name ?: "Unknown"
```

over:

```kotlin
val deviceName = name!!
```

## 23. A practical nullable measurement example

Suppose:

```kotlin
var latestReading: Double? = null
```

You can display:

```kotlin
val message = if (latestReading != null) {
    "Reading = $latestReading"
} else {
    "Waiting for measurement"
}
```

Or more concisely with safe-call and Elvis operator:

```kotlin
val message = latestReading?.let {
    // 'let' enters this block only if latestReading is not null.
    // 'it' refers to the non-null latestReading value inside this block.
    "Reading = $it"
} ?: "Waiting for measurement"
// If latestReading was null, the Elvis operator ?: provides the fallback.
```

**Key idea**: `?.let` means "if not null, enter this block". The `?:` means "otherwise use this value".

Don't worry too much about `let` yet. We'll encounter lambdas and scope functions later.

The first `if` version is perfectly good while learning.

## 24. Default parameter values

Kotlin also allows default values in functions.

For example:

```kotlin
fun createMeasurement(
    sampleId: String,
    value: Double,
    repetition: Int = 1
) {
    println("$sampleId, $value, $repetition")
}
```

You can call:

```kotlin
createMeasurement(
    sampleId = "S001",
    value = 2.45
)
```

and Kotlin automatically uses:

```text
repetition = 1
```

Or override it:

```kotlin
createMeasurement(
    sampleId = "S001",
    value = 2.45,
    repetition = 5
)
```

This feature is extremely common in Compose later.

You'll see functions like:

```kotlin
Text(
    text = "Hello",
    fontSize = 20.sp
)
```

where many other parameters have defaults.

## 25. Default values in a data class

You can also write:

```kotlin
data class Measurement(
    val sampleId: String,
    val value: Double,
    val valid: Boolean = true
)
```

Then:

```kotlin
val measurement = Measurement(
    sampleId = "S001",
    value = 2.45
)
```

automatically has:

```text
valid = true
```

## The most important part of Lesson 3

If you remember only these patterns, you're doing well:

```kotlin
data class Measurement(
    val sampleId: String,
    val value: Double
)
```

Create it:

```kotlin
val m = Measurement(
    sampleId = "S001",
    value = 2.45
)
```

Nullable value:

```kotlin
var reading: Double? = null
```

Safe access:

reading?.toString()

Fallback value:

```kotlin
val display = reading?.toString() ?: "No data"
```

Enum:

    MEASURING

```kotlin
enum class DeviceState {
    DISCONNECTED,
    CONNECTED,
}
```

And:

```text
when (state) {
    DeviceState.DISCONNECTED -> println("Disconnected")
    DeviceState.CONNECTED -> println("Ready")
    DeviceState.MEASURING -> println("Measuring")
}
```

## One mental model for ?, ?., ?:, !!

This is worth memorising:

| Syntax | Meaning |
| --- | --- |
| String | must contain a string |
| String? | string or null |
| x?.foo() | call foo() only if x exists |
| x ?: default | use default if x is null |
| x!! | assume non-null; crash if wrong |

Of these, the three you'll use most are:

?

?.

?:

and I would treat:

!!

as something to be suspicious of rather than something to use routinely.

## Next: Lesson 4 — Your first actual Android app

At this point you've learned enough pure Kotlin for us to stop doing only console examples.

In Lesson 4, I'd move into Android Studio + Jetpack Compose and explain:

```text
Android project
    ↓
MainActivity
    ↓
setContent
    ↓
@Composable
    ↓
Text / Button / TextField
    ↓
State
```

Then we'll build a first small screen such as:

Research Measurement

- Sample ID
- [ S001             ]

- Current value
- 2.43

[ Start Measurement ]

That will be the point where the Kotlin you've just learned starts becoming an actual Android app.
