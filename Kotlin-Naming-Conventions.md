# Kotlin Naming Conventions

This is a quick beginner reference for common Kotlin naming patterns.

## Classes, Interfaces, and Objects

Use **UpperCamelCase**.

```kotlin
class ResearchViewModel
class MeasurementRepository
data class Measurement
object AppConstants
interface MeasurementSource
```

Class names are usually **nouns** because a class represents a thing or concept.

Good examples:

```kotlin
class Measurement
class PatientSession
class ResearchViewModel
class MeasurementRepository
```

Avoid class names that sound like actions:

```kotlin
class SaveMeasurement // less clear
class CreateSession // less clear
```

Better:

```kotlin
class MeasurementSaver
class SessionCreator
```

## Variables and Properties

Use **lowerCamelCase**.

```kotlin
val sampleId = "S001"
var isAcquiring = false
val measurementRepository = MeasurementRepository()
```

Variable names should describe the data they hold.

Good examples:

```kotlin
val sampleId = "S001"
val latestValue = 3.2
val measurements = listOf<Measurement>()
val newMeasurement = Measurement(...)
```

Boolean variables often start with words such as `is`, `has`, `can`, or `should`.

```kotlin
val isAcquiring = false
val hasSavedData = true
val canStartRecording = true
val shouldShowError = false
```

## Functions

Use **lowerCamelCase**.

```kotlin
fun startAcquisition()
fun stopAcquisition()
fun createSimulatedMeasurement()
```

Function names usually describe an **action**, so they often start with a verb.

Common pattern:

```text
verb + noun
```

Examples:

```kotlin
fun startAcquisition()
fun stopAcquisition()
fun saveMeasurements()
fun loadMeasurements()
fun createSimulatedMeasurement()
fun updateSampleId()
fun clearSession()
```

The verb tells you what the function does:

```text
start -> begin something
stop -> end something
save -> write data somewhere
load -> read data from somewhere
create -> make a new object
update -> change existing state
clear -> remove/reset data
```

If a function returns a value without changing much state, names like `get`, `calculate`, `format`, or `convert` are common.

```kotlin
fun getLatestMeasurement(): Measurement?
fun calculateAverageValue(): Double
fun formatMeasurementValue(value: Double): String
fun convertMeasurementToCsvRow(measurement: Measurement): String
```

Avoid vague names:

```kotlin
fun doThing()
fun handleData()
fun process()
```

Better:

```kotlin
fun saveMeasurements()
fun handleStartButtonClick()
fun processSensorReading()
```

## Constants

Use **UPPER_SNAKE_CASE** for real constants.

```kotlin
const val MAX_RETRY_COUNT = 3
const val DEFAULT_SAMPLE_ID = "Sample001"
```

## Packages

Use lowercase words.

```kotlin
package com.example.researchapp
package com.example.researchapp.data
package com.example.researchapp.ui
```

## Files

Use **UpperCamelCase** when the file mainly contains one class.

```text
MeasurementRepository.kt
ResearchViewModel.kt
Measurement.kt
```

Utility files can also use descriptive **UpperCamelCase** names.

```text
MeasurementUtils.kt
StorageHelpers.kt
```

## Most Useful Beginner Rule

```text
Uppercase first letter -> usually a type, class, interface, or object name
lowercase first letter -> usually a variable, property, or function
```

Example:

```kotlin
val measurementRepository = MeasurementRepository()
```

Meaning:

```text
measurementRepository -> variable/property
MeasurementRepository -> class name
```

## Quick Meaning Guide

```text
MeasurementRepository -> a class/type
measurementRepository -> one instance of that class
createSimulatedMeasurement -> a function/action
isAcquiring -> a Boolean state
MAX_RETRY_COUNT -> a fixed constant
```
