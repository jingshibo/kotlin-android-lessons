# Lesson 3 Notes - Data Classes

This companion note contains the parts of Lesson 3 that focus specifically on Kotlin data classes.

The main lesson continues in [Lesson 3 - Classes, Null Safety, and Enums](lesson-03-classes-null-safety-and-enums.md).

This note covers:

- what a `data class` is
- when to use one
- generated `toString()` and equality behavior
- creating modified objects with `copy()`
- default property values
- a realistic research measurement model
- primary-constructor properties versus class-body properties
- computed properties and custom getters
- when data-class instances are created

## 1. `data class`

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

## 2. Why use a data class?

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
val measurement = Measurement("S001", 2.45)

println(measurement)
```

produces something like:

```text
Measurement(sampleId=S001, value=2.45)
```

With a normal class, printing the object would not automatically give such useful output.

## 3. Comparing data classes

Consider:

```kotlin
val first = Measurement("S001", 2.45)
val second = Measurement("S001", 2.45)
```

With a data class:

```kotlin
println(first == second)
```

returns:

```text
true
```

Kotlin compares the values stored in the primary-constructor properties. This is useful when comparing research records.

## 4. `copy()`

Another useful data-class feature is `copy()`.

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

The properties not supplied to `copy()` retain their values from the original object.

You will see `copy()` often in modern Android development, especially when updating immutable UI state.

## 5. Default values in a data class

Kotlin's general default-parameter feature also applies to data-class constructor properties:

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

The general default-parameter rule is introduced in [Lesson 3, Section 10](lesson-03-classes-null-safety-and-enums.md#10-default-parameter-values).

## 6. A realistic measurement model

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

## 7. Constructor properties and computed properties

The following UI-state class looks different from the earlier data classes because it has properties in both the primary constructor and the class body:

```kotlin
data class SessionDetailsUiState(
    val session: SessionRecord,
    val patient: PatientRecord? = null,
    val deviceRecord: DeviceRecord? = null,
    val displayedResult: ResultRecord? = null,
    val status: String = "Transferred"
) {
    // Session identifiers and timestamps
    val sessionId: String get() = session.sessionId.toString()
    val patientId: String get() = session.patientCode
    val recordingDate: String get() = session.recordingDay
    val transferDate: String
        get() = session.transferredAt?.toFormattedDateTime() ?: "-"
    val device: String get() = session.deviceCode
    val arm: String get() = session.arm.toUiText()

    // AI prediction and confidence
    val prediction: String
        get() = displayedResult?.prediction?.toUiText() ?: "-"
    val confidence: String
        get() = displayedResult?.confidence?.toConfidenceText() ?: "-"
    val modelVersion: String
        get() = displayedResult?.modelVersion ?: "-"
    val predictionDate: String
        get() = displayedResult?.analyzedAt?.toFormattedDateTime() ?: "-"
    val resultsList: List<ResultRecord>
        get() = session.resultsList

    // Patient demographics
    val studyGroup: String
        get() = patient?.studyGroup?.toUiText() ?: "--"
    val location: String
        get() = patient?.location ?: "--"
    val sex: String
        get() = patient?.sex?.toUiText() ?: "--"
    val ageGroup: String
        get() = patient?.ageGroup?.toUiText() ?: "--"

    // Invalidation and audit compliance
    val isVoided: Boolean get() = session.isVoided
    val invalidationReason: String get() = session.voidReason.orEmpty()
    val invalidatedAt: String
        get() = session.voidedAt?.toFormattedDateTime() ?: "-"
}
```

The functions such as `toFormattedDateTime()`, `toUiText()`, and `toConfidenceText()` are formatting helpers. Their exact implementations are not important for understanding the data-class grammar.

### Two grammar areas

The class has two visible areas:

```kotlin
// 1. Primary constructor
data class SessionDetailsUiState(
    val session: SessionRecord,
    val patient: PatientRecord? = null
) {
    // 2. Class body
    val sessionId: String get() = session.sessionId.toString()
}
```

Their roles in this example are:

```text
Primary-constructor properties
    -> store the source-of-truth objects used by this UI state
    -> participate in generated data-class functions

Class-body computed properties
    -> derive convenient UI values from those source objects
    -> run their getter whenever the property is read
```

However, location alone does not determine whether a property is stored. A property in the class body can also have a backing field:

```kotlin
val storedLabel: String = session.sessionId.toString()
```

The important difference is whether the property stores a value or defines only a custom getter.

### Primary-constructor properties

Each constructor parameter declared with `val` or `var` becomes a property:

```kotlin
val session: SessionRecord
val patient: PatientRecord?
val deviceRecord: DeviceRecord?
val displayedResult: ResultRecord?
val status: String
```

These properties hold the values supplied when the object is constructed. For object types such as `SessionRecord`, the property normally holds a reference to the existing object; it does not create another complete copy of that object.

For a data class, only the properties declared in the primary constructor participate in the automatically generated:

- `equals()`
- `hashCode()`
- `toString()`
- `copy()`
- component functions used for destructuring

For example, `status` participates in `copy()` and equality, but the class-body property `sessionId` does not participate directly.

`copy()` is also shallow. It creates a new `SessionDetailsUiState`, but unchanged object properties continue to refer to the same `SessionRecord`, `PatientRecord`, and other source objects:

```kotlin
val updated = detailsState.copy(
    status = "Analysed"
)
```

### Computed property getters

This syntax declares a read-only computed property:

```kotlin
val sessionId: String get() = session.sessionId.toString()
```

It is the single-expression form of:

```kotlin
val sessionId: String
    get() {
        return session.sessionId.toString()
    }
```

The custom getter lets callers use normal property syntax:

```kotlin
val id = detailsState.sessionId
```

Behind that property access, Kotlin executes the getter. Conceptually, this is similar to calling a function such as `detailsState.getSessionId()`.

Because the property has only a custom getter and no initializer or setter, it does not need its own per-instance backing field. Its result is calculated from `session` each time it is read.

That does not mean the calculation uses literally no memory. The getter's execution and any returned or temporary objects may still use memory. The precise runtime behavior is also subject to JVM optimization. The useful Kotlin-level rule is simply:

```text
Stored property
    -> has a value retained by the object

Computed property with only get()
    -> has no separate stored value for that property
    -> calculates its result when read
```

In Compose, a getter may be read again during recomposition. Keep computed UI properties inexpensive, deterministic, and free of side effects. Do not perform database queries, file access, or expensive processing inside them.

### Null-safety inside the getters

The getters use Kotlin's null-safety operators because some source records may be absent.

Safe calls stop the chain when a nullable value is `null`:

```kotlin
displayedResult?.prediction?.toUiText()
```

The Elvis operator supplies display text when the result is `null`:

```kotlin
displayedResult?.prediction?.toUiText() ?: "-"
```

Similarly:

```kotlin
patient?.location ?: "--"
```

means:

```text
If patient exists and location exists, use location.
Otherwise, use "--".
```

`orEmpty()` is a standard Kotlin helper for nullable strings:

```kotlin
session.voidReason.orEmpty()
```

It returns the string when it exists and `""` when it is `null`.

### When is the data-class object created?

Declaring a class defines a type, but it does not create an instance:

```kotlin
data class SessionDetailsUiState(...)
```

An instance is created when the constructor is called:

```kotlin
val detailsState = SessionDetailsUiState(
    session = sessionRecord,
    patient = patientRecord,
    deviceRecord = deviceRecord,
    displayedResult = resultRecord,
    status = "Analysed"
)
```

The class must therefore be instantiated before instance properties such as `detailsState.sessionId` can be read.

A ViewModel may create one instance for every session while mapping domain records into UI state:

```kotlin
val detailsStates = sessions.map { sessionRecord ->
    SessionDetailsUiState(
        session = sessionRecord
    )
}
```

Calling `copy()` creates another instance:

```kotlin
val selectedState = detailsState.copy(
    status = "Selected"
)
```

At the Kotlin level, think of each constructor or `copy()` call as producing an object. On Android, the runtime manages the underlying allocation and may optimize it. When an object is no longer reachable, the garbage collector can eventually reclaim its memory.

### What does "stored as a field" mean?

A backing field is the storage associated with a property inside an object. In this example:

| Property | Separate stored value in `SessionDetailsUiState`? | Reason |
|---|---:|---|
| `session` | Yes | Primary-constructor `val` |
| `patient` | Yes | Primary-constructor `val` |
| `deviceRecord` | Yes | Primary-constructor `val` |
| `displayedResult` | Yes | Primary-constructor `val` |
| `status` | Yes | Primary-constructor `val` |
| `sessionId` | No | Calculated by a getter |
| `prediction` | No | Calculated by a getter |
| `studyGroup` | No | Calculated by a getter |

Avoid estimating the object as a fixed number of bytes from the number of properties. Object headers, alignment, nullable references, runtime configuration, and JVM optimizations all affect the real memory layout.

### When this pattern is useful

Constructor source objects plus computed properties work well when the derived values are:

- inexpensive to calculate
- always derived from the source objects
- read-only
- useful as convenient UI-facing property names

Store a separate value instead when it represents independent state, such as a user's editable draft, a snapshot that must not change with its source, or an expensive result that should be calculated only once.

## What to remember

```text
Normal class
    -> represents an object with properties and behavior

Data class
    -> mainly represents stored values
    -> generates useful value-based functions

Primary-constructor property
    -> participates in generated data-class functions

Computed property with get()
    -> derives a value when read
    -> has no separate backing field for that property
```

The core pattern is:

```kotlin
data class Measurement(
    val sampleId: String,
    val value: Double
)

val original = Measurement("S001", 2.45)

val corrected = original.copy(
    value = 2.50
)
```

Return to [Lesson 3 - Classes, Null Safety, and Enums](lesson-03-classes-null-safety-and-enums.md) for classes, null safety, and enums.
