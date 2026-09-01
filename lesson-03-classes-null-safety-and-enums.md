# Lesson 3 - Classes, Data Classes, Null Safety, and Enums

This lesson is especially important because these features appear constantly in Android code.

The lesson has four main parts:

```text
Part 1: Classes
    -> define your own object types

Part 2: Data classes
    -> store research data cleanly

Part 3: Null safety
    -> handle values that may not exist yet

Part 4: Enums
    -> represent fixed states safely
```

We will cover:

- classes and objects
- constructors
- `val` and `var` inside classes
- `val` and `var` for object variables versus object properties
- functions inside classes
- class inheritance
- `data class`
- named arguments
- `copy()`
- default parameter values
- lists of objects
- nullable types with `?`
- safe calls with `?.`
- Elvis operator `?:`
- non-null assertion `!!`
- `enum class`
- `when` with enums

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

You can access its properties using `.`:

```kotlin
println(measurement.sampleId)
println(measurement.value)
```

Output:

```text
S001
2.45
```

Simple mental model:

```text
Class = the blueprint
Object = one actual thing created from that blueprint
```

## 2. Constructor syntax

Kotlin lets you define the constructor directly in the class declaration:

```kotlin
class Measurement(
    val sampleId: String,
    val value: Double
)
```

This means:

```text
To create a Measurement, provide:
    sampleId
    value
```

For example:

```kotlin
val measurement = Measurement(
    sampleId = "S001",
    value = 2.45
)
```

The `sampleId =` and `value =` parts are called named arguments.

Named arguments make code easier to read, especially when a constructor has several values.

This:

```kotlin
Measurement(
    sampleId = "S001",
    value = 2.45
)
```

is clearer than:

```kotlin
Measurement("S001", 2.45)
```

## 3. val and var inside a class

You can choose whether properties inside a class can change.

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

is valid because `value` is `var`.

But:

```kotlin
measurement.sampleId = "S002"
```

is not allowed because `sampleId` is `val`.

General rule:

```text
Prefer val unless the property genuinely needs to change.
```

## 4. val and var for object variables vs class properties

A common confusion is that `val` can appear in two different places.

First, `val` or `var` can describe the variable that holds the object:

```kotlin
val measurement = Measurement("S001", 2.45)
```

Second, `val` or `var` can describe the properties inside the object:

```kotlin
class Measurement(
    val sampleId: String,
    var value: Double
)
```

These are separate decisions.

This is the key idea:

```text
val measurement
```

does not mean:

```text
Nothing inside the Measurement object can change.
```

It only means:

```text
The measurement variable cannot point to a different object.
```

Because `measurement` is `val`, this is not allowed:

```kotlin
measurement = Measurement("S002", 3.10) // Not allowed
```

But because `value` inside the class is `var`, this is allowed:

```kotlin
measurement.value = 2.60 // Allowed
```

However, because `sampleId` inside the class is `val`, you cannot change that property:

```kotlin
measurement.sampleId = "S002" // Not allowed
```

So there are two levels:

```text
val measurement
    -> the variable cannot point to a different object

var value inside Measurement
    -> this property can change inside the same object

val sampleId inside Measurement
    -> this property cannot change inside the same object
```

## 5. Functions inside classes

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

Output:

```text
Sample: S001
Value: 2.45
```

This means an object can store data and also have functions related to that data.

## 6. Class inheritance

Sometimes one class needs to be a special kind of another class.

In Kotlin, this uses `:`.

For example, in Android you will see code like this:

```kotlin
class MainActivity : ComponentActivity()
```

This means:

```text
Create a class named MainActivity.
MainActivity inherits from ComponentActivity.
So MainActivity is a kind of ComponentActivity.
```

The same idea appears later with ViewModel (lesson 08):

```kotlin
class ResearchViewModel : ViewModel()
```

This means:

```text
Create a class named ResearchViewModel.
ResearchViewModel inherits from ViewModel.
So ResearchViewModel is a kind of ViewModel.
```

The parent class gives your class useful behavior.

For example:

- `ComponentActivity` gives `MainActivity` the behavior needed to be an Android screen.
- `ViewModel` gives `ResearchViewModel` the behavior needed to hold UI state outside the composable screen.

The parentheses are there because you are calling the parent class constructor:

```kotlin
ViewModel()
ComponentActivity()
```

Simple mental model:

```text
class Child : Parent()
```

means:

```text
Child is a kind of Parent.
Child gets behavior from Parent.
```


### Class inheritance with your own constructor properties

Sometimes your class needs its own constructor properties and also needs to inherit from a parent class.

In that case, your class properties go inside the first parentheses, before the `:`.

General shape:

```kotlin
class ChildClass(
    val myValue: String,
    var myNumber: Int
) : ParentClass()
```

This means:

```text
ChildClass has its own constructor properties:
    myValue
    myNumber

ChildClass also inherits from ParentClass.
```

For example:

```kotlin
class ResearchViewModel(
    private val repository: MeasurementRepository
) : ViewModel()
```

This means:

```text
Create a class named ResearchViewModel.
When creating it, provide a MeasurementRepository.
Store that repository inside the ViewModel.
ResearchViewModel still inherits from ViewModel.
```

The `private val repository` part means:

```text
repository is a property of ResearchViewModel,
but only code inside ResearchViewModel can use it directly.
```

This is still a property inside the class. Kotlin lets you declare it in the primary constructor.

These two styles are similar:

```kotlin
class ResearchViewModel(
    private val repository: MeasurementRepository = MeasurementRepository()
) : ViewModel()
```

```kotlin
class ResearchViewModel : ViewModel() {
    private val repository: MeasurementRepository = MeasurementRepository()
}
```

The constructor style makes it clearer that `ResearchViewModel` needs a repository.

It also makes the repository easier to replace later, for example in testing.

So compare these two patterns:

```kotlin
class ResearchViewModel : ViewModel()
```

This class inherits from `ViewModel`, but does not ask for anything when it is created. The ViewModel always creates its own repository, and it is harder to replace during testing. 

```kotlin
class ResearchViewModel(
    private val repository: MeasurementRepository = MeasurementRepository()
) : ViewModel()
```

This class inherits from `ViewModel`with a default repository, and it also asks for a `MeasurementRepository` when it is created, so you can replace it with your own when needed.

For now, you do not need to design your own inheritance structure. You mainly need to recognize this syntax because Android uses it often.

Later, if you create your own parent classes, Kotlin has extra inheritance rules, such as marking a parent class as `open`. You do not need that yet for these Android examples.



### Inheritance and assignment

Inheritance also matters when code wants to work with a general category instead of one exact class.

For example, a function might not care whether an animal is a `Dog`, `Cat`, or `Bird`. It may only care that the object is some kind of `Animal`.

That is why inheritance creates one important assignment rule:

For example:

```kotlin
open class Animal

class Dog : Animal()
```

This is allowed:

```kotlin
val animal: Animal = Dog()
```

because a `Dog` is a kind of `Animal`.

The pattern is:

```text
val variable: ParentType = ChildType()
```

That means:

```text
The variable is typed as the parent.
The actual object is created from the child class.
This works because the child is a kind of the parent.
```

The Android version of that idea would be:

```kotlin
class ResearchViewModel : ViewModel()
```

So this kind of assignment would be allowed:

```kotlin
val generalViewModel: ViewModel = ResearchViewModel()
```

because a `ResearchViewModel` is a kind of `ViewModel`.

This is only an inheritance example. It is not the same pattern as the Compose line you will see in Lesson 8:

```kotlin
val researchViewModel: ResearchViewModel = viewModel()
```

That Lesson 8 line is mainly about calling a function named `viewModel()`.

For now, keep these two ideas separate:

```text
Inheritance:
val parent: Parent = Child()

Function returns an object:
val thing: Thing = getThing()
```

If you create the exact class yourself, the normal Kotlin pattern is:

```kotlin
val researchViewModel: ResearchViewModel = ResearchViewModel()
```

Here the variable type and the object type are exactly the same.


## 7. data class

For research data, you will very often want a class that mainly stores information.

Kotlin provides `data class` for this.

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

Use a `data class` when the main purpose of the class is to hold data.

## 8. Why use data class?

Suppose:

```kotlin
data class Measurement(
    val sampleId: String,
    val value: Double
)
```

Kotlin automatically gives you useful functionality such as:

- readable `toString()`
- equality comparison by stored values
- `copy()`

For example:

```kotlin
val m1 = Measurement("S001", 2.45)

println(m1)
```

produces something like:

```text
Measurement(sampleId=S001, value=2.45)
```

With a normal class, printing the object would not automatically give such useful output.

## 9. Comparing data classes

Consider:

```kotlin
val m1 = Measurement("S001", 2.45)
val m2 = Measurement("S001", 2.45)
```

With a data class:

```kotlin
println(m1 == m2)
```

returns:

```text
true
```

because Kotlin compares their stored values.

This is very useful when comparing research records.

## 10. copy()

Another useful data class feature is `copy()`.

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

You will see `copy()` often in modern Android development.

## 11. Default parameter values

Kotlin allows default values in functions.

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

You will see functions like:

```kotlin
Text(
    text = "Hello",
    fontSize = 20.sp
)
```

where many other parameters have defaults.

## 12. Default values in a data class

You can also write default values in a data class.

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

Default values are useful when some information has a normal starting value, but other information still needs to be provided.

## 13. A realistic measurement model

For a research app, a measurement often needs more than one value.

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

`System.currentTimeMillis()` gives the current Unix timestamp in milliseconds.

You do not need to worry about timestamps deeply yet.

## 14. Lists of objects

Now combine Lesson 2 with Lesson 3.

Using the `Measurement` data class from the previous section:

```kotlin
val measurements = listOf(
    Measurement("S001", 1, 2.31, 1000),
    Measurement("S001", 2, 2.45, 2000),
    Measurement("S001", 3, 2.52, 3000)
)
```

You can loop:

```kotlin
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

Breakdown:

```kotlin
measurements.map { it.value }
```

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

```kotlin
.average()
```

calculates the average.

This style becomes very common in the research app.

## 15. Null safety

Now we get to one of the most important Kotlin concepts.

Suppose you write:

```kotlin
var deviceName: String = "Sensor A"
```

Kotlin will not let you do:

```kotlin
deviceName = null
```

because `String` means:

```text
This value must contain a String.
```

If a variable is allowed to be missing, use `?`.

For example:

```kotlin
var deviceName: String? = null
```

The `?` means:

```text
This variable may contain either a String or null.
```

## 16. Why null safety matters in Android

Android code constantly deals with things that might not exist yet:

```text
Bluetooth device might not be connected
sensor reading might not have arrived
user might not have entered a sample ID
file might not exist
database query might return nothing
```

So you will encounter nullable types all the time.

For example:

```kotlin
var latestReading: Double? = null
```

Before data arrives:

```kotlin
latestReading = null
```

After a measurement:

```kotlin
latestReading = 2.45
```

## 17. Kotlin protects you from null errors

Suppose:

```kotlin
val name: String? = null
```

This will not compile:

```kotlin
println(name.length)
```

Why?

Because Kotlin says:

```text
name might be null, so calling .length could crash.
```

You must explicitly handle that possibility.

This is a major reason Kotlin is safer than Java.

## 18. Safe-call operator ?.

You can write:

```kotlin
println(name?.length)
```

The operator `?.` means:

```text
Call this function or property only if the object is not null.
```

If:

```kotlin
val name: String? = "Sensor"
```

then:

```kotlin
name?.length
```

returns:

```text
6
```

If:

```kotlin
val name: String? = null
```

then:

```kotlin
name?.length
```

returns:

```text
null
```

rather than crashing.

## 19. A very common Android pattern

Suppose:

```kotlin
var deviceName: String? = null
```

You can write:

```kotlin
println(deviceName?.uppercase())
```

If the device exists:

```text
SENSOR A
```

If it is null:

```text
null
```

This is safe, but usually you do not want to display `null` to the user.

That is where the Elvis operator helps.

## 20. Elvis operator ?:

Usually you do not want to display `null`.

You might want a default value.

That is what this operator does:

```text
?:
```

Example:

```kotlin
val name: String? = null

val displayedName = name ?: "Unknown device"
```

This means:

```text
Use name if it is not null.
Otherwise use "Unknown device".
```

So:

```kotlin
println(displayedName)
```

outputs:

```text
Unknown device
```

You will see this operator frequently.

## 21. Combining ?. and ?:

This is very common:

```kotlin
val name: String? = null

val length = name?.length ?: 0
```

Read it as:

```text
If name exists, get its length.
Otherwise use 0.
```

For Android research code:

```kotlin
val displayValue = latestReading?.toString() ?: "No data"
```

Very useful.

## 22. Standard if null check

You can also do:

```kotlin
if (deviceName != null) {
    println(deviceName.length)
}
```

Inside this block Kotlin understands that `deviceName` is not null.

This is called smart casting.

You do not have to manually convert it from:

```text
String?
```

to:

```text
String
```

## 23. !! - non-null assertion

You will also encounter:

```text
!!
```

Example:

```kotlin
val name: String? = "Sensor"

println(name!!.length)
```

`!!` means:

```text
I promise this is not null.
```

But if it actually is null:

```kotlin
val name: String? = null

println(name!!.length)
```

your application can crash.

So my recommendation is:

```text
Avoid !! whenever possible.
```

Prefer:

```text
?.
?:
explicit null check
```

For example, prefer:

```kotlin
val deviceName = name ?: "Unknown"
```

over:

```kotlin
val deviceName = name!!
```

## 24. A practical nullable measurement example

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
    // let enters this block only if latestReading is not null.
    // it refers to the non-null latestReading value inside this block.
    "Reading = $it"
} ?: "Waiting for measurement"
```

Key idea:

```text
?.let means: if not null, enter this block.
?: means: otherwise use this value.
```

Do not worry too much about `let` yet. We will encounter lambdas and scope functions later.

The first `if` version is perfectly good while learning.

## 25. enum class

Sometimes a value should only be one of a small fixed set of options.

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

```kotlin
enum class DeviceState {
    DISCONNECTED,
    CONNECTED,
    MEASURING,
    ERROR
}
```

Now the value must be one of those enum constants:

```kotlin
var state = DeviceState.DISCONNECTED
```

Later:

```kotlin
state = DeviceState.CONNECTED
```

## 26. Enum constants are not strings

The items in an enum class are enum constants of type `DeviceState`, not `String` values.

For example:

```kotlin
val state = DeviceState.CONNECTED
```

`state` is a `DeviceState`, not a `String`.

However, enum constants can be displayed because Kotlin can convert them to their name:

```kotlin
println("Device status: $state")
```

This displays:

```text
Device status: CONNECTED
```

If you need the name explicitly as a `String`:

```kotlin
val text = state.name
```

For more readable UI text, use a `when` expression to map enum constants to custom labels:

```kotlin
val displayText = when (state) {
    DeviceState.DISCONNECTED -> "Disconnected"
    DeviceState.CONNECTED -> "Connected"
    DeviceState.MEASURING -> "Measuring"
    DeviceState.ERROR -> "Error"
}
```

## 27. when + enum

Enums work beautifully with `when`.

```kotlin
val state = DeviceState.CONNECTED

val message = when (state) {
    DeviceState.DISCONNECTED -> "Device not connected"
    DeviceState.CONNECTED -> "Device ready"
    DeviceState.MEASURING -> "Measurement in progress"
    DeviceState.ERROR -> "Device error"
}
```

Notice:

```text
There is no else.
```

Why?

Because Kotlin knows you have handled every possible value of `DeviceState`.

That is safer than using arbitrary strings.

## 28. Research example

Now we can combine data classes, null safety, and enums.

```kotlin
enum class MeasurementStatus {
    WAITING,
    RECORDING,
    COMPLETE,
    ERROR
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

And display the status:

```kotlin
val statusMessage = when (measurement.status) {
    MeasurementStatus.WAITING -> "Waiting"
    MeasurementStatus.RECORDING -> "Recording"
    MeasurementStatus.COMPLETE -> "Complete"
    MeasurementStatus.ERROR -> "Error"
}
```

This looks much more like real Android application code.

## 29. A complete example

```kotlin
enum class DeviceState {
    DISCONNECTED,
    CONNECTED,
    MEASURING,
    ERROR
}

data class Measurement(
    val sampleId: String,
    val value: Double?,
    val repetition: Int
)

fun main() {

    val deviceState = DeviceState.CONNECTED

    val measurement = Measurement(
        sampleId = "S001",
        value = 2.43,
        repetition = 1
    )

    val deviceMessage = when (deviceState) {
        DeviceState.DISCONNECTED -> "Device disconnected"
        DeviceState.CONNECTED -> "Device ready"
        DeviceState.MEASURING -> "Measurement in progress"
        DeviceState.ERROR -> "Device error"
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

## 30. What you need to remember from Lesson 3

If you remember only these patterns, you are doing well.

Class:

```kotlin
class Measurement(
    val sampleId: String,
    val value: Double
)
```

Data class:

```kotlin
data class Measurement(
    val sampleId: String,
    val value: Double
)
```

Create an object:

```kotlin
val measurement = Measurement(
    sampleId = "S001",
    value = 2.45
)
```

Copy with one changed field:

```kotlin
val corrected = measurement.copy(
    value = 2.50
)
```

Nullable value:

```kotlin
var reading: Double? = null
```

Safe access:

```kotlin
reading?.toString()
```

Fallback value:

```kotlin
val display = reading?.toString() ?: "No data"
```

Enum:

```kotlin
enum class DeviceState {
    DISCONNECTED,
    CONNECTED,
    MEASURING,
    ERROR
}
```

`when` with enum:

```kotlin
val state = DeviceState.CONNECTED

val message = when (state) {
    DeviceState.DISCONNECTED -> "Disconnected"
    DeviceState.CONNECTED -> "Ready"
    DeviceState.MEASURING -> "Measuring"
    DeviceState.ERROR -> "Error"
}
```

## 31. One mental model for ?, ?., ?:, !!

This is worth memorizing:

| Syntax | Meaning |
|---|---|
| `String` | must contain a string |
| `String?` | string or null |
| `x?.foo()` | call `foo()` only if `x` exists |
| `x ?: default` | use default if `x` is null |
| `x!!` | assume non-null; crash if wrong |

Of these, the three you will use most are:

```text
?
?.
?:
```

Treat `!!` as something to be suspicious of rather than something to use routinely.

## Next: Lesson 4 - Your first actual Android app

At this point, you have learned enough pure Kotlin for us to stop doing only console examples.

In Lesson 4, we move into Android Studio and Jetpack Compose:

```text
Android project
    -> MainActivity
    -> setContent
    -> @Composable
    -> Text / Button / TextField
    -> State
```

Then we build a first small screen:

```text
Research Measurement

Sample ID
[ S001             ]

Current value
2.43

[ Start Measurement ]
```

That is where the Kotlin you learned starts becoming an actual Android app.
