# Lesson 28 — Adding Signal Processing

In Lesson 27, we added the fake device source.

The repository can now:

```text
connect fake device
 ↓
read fake value
 ↓
create MeasurementEntity
 ↓
save to Room
```

But in Lesson 27, we temporarily used this:

```kotlin
processedValue = rawValue
```

That means the app was saving the raw value and processed value as the same thing.

Now we improve this.

This corresponds to **Step A6** in Direction A:

```text
Add simple signal processing
```

The goal of Lesson 28 is to connect the repository to a `SignalProcessor`.

After this lesson, the data flow becomes:

```text
FakeDeviceDataSource
 ↓
rawValue
 ↓
SignalProcessor
 ↓
processedValue + status
 ↓
MeasurementEntity
 ↓
Room database
```

---

## 1. Why add signal processing now?

A real research app usually does not only store raw sensor values.

The raw signal may include:

```text
baseline offset
noise
spikes
invalid values
device variation
daily variation
temperature effects
movement artefacts
```

So the app usually needs a processing step before the value becomes useful.

For example:

```text
rawValue = 2.438
baseline = 0.200
processedValue = 2.238
```

The simple processing idea is:

```text
processedValue = rawValue - baseline
```

This is not advanced yet, but it introduces the correct app structure.

---

## 2. Where signal processing belongs

Do not put processing inside the UI.

Avoid this:

```text
MeasurementScreen
 └── rawValue - baseline
```

Also avoid putting too much processing directly inside the ViewModel.

Better structure:

```text
MeasurementScreen
 ↓
ResearchViewModel
 ↓
MeasurementRepository
 ↓
SignalProcessor
```

The `SignalProcessor` is responsible for:

```text
baseline correction
validity checking
moving average
feature extraction
```

The repository uses the processor when creating measurements.

---

## 3. Create the `processing` package

From Lesson 23, we planned this folder:

```text
processing
 ├── SignalProcessor.kt
 └── SignalFeatures.kt
```

Create this folder:

```text
app/src/main/java/com/example/researchapp/processing
```

Inside it, create:

```text
SignalProcessor.kt
SignalFeatures.kt
```

---

## 4. Create `SignalFeatures.kt`

Create:

```text
processing/SignalFeatures.kt
```

Code:

```kotlin
package com.example.researchapp.processing

data class SignalFeatures(
    val mean: Double,
    val minimum: Double,
    val maximum: Double,
    val range: Double
)
```

This class stores simple summary features.

For example, from a list of processed values:

```text
2.1, 2.3, 2.5
```

we can calculate:

```text
mean = 2.3
minimum = 2.1
maximum = 2.5
range = 0.4
```

These features will be useful later for fake ML inference and real ML inference.

---

## 5. Create `SignalProcessor.kt`

Create:

```text
processing/SignalProcessor.kt
```

Code:

```kotlin
package com.example.researchapp.processing

class SignalProcessor {

    fun baselineCorrect(
        rawValue: Double,
        baseline: Double
    ): Double {
        return rawValue - baseline
    }

    fun isValidValue(
        value: Double
    ): Boolean {
        return value in 0.0..10.0
    }

    fun movingAverage(
        values: List<Double>,
        windowSize: Int
    ): Double? {
        val recentValues = values.takeLast(windowSize)

        if (recentValues.isEmpty()) {
            return null
        }

        return recentValues.average()
    }

    fun extractFeatures(
        values: List<Double>
    ): SignalFeatures? {
        if (values.isEmpty()) {
            return null
        }

        val minimum = values.min()
        val maximum = values.max()

        return SignalFeatures(
            mean = values.average(),
            minimum = minimum,
            maximum = maximum,
            range = maximum - minimum
        )
    }
}
```

This gives us four useful functions:

```text
baselineCorrect()
isValidValue()
movingAverage()
extractFeatures()
```

---

## 6. Understand `baselineCorrect()`

```kotlin
fun baselineCorrect(
    rawValue: Double,
    baseline: Double
): Double {
    return rawValue - baseline
}
```

This subtracts the baseline from the raw value.

Example:

```text
rawValue = 2.45
baseline = 0.20
processedValue = 2.25
```

For now, we use a fixed baseline.

Later, the baseline could come from:

```text
blank device reading
daily calibration
session baseline
device-only measurement
stored calibration value
```

For the current Direction A skeleton, a fixed baseline is enough.

---

## 7. Understand `isValidValue()`

```kotlin
fun isValidValue(
    value: Double
): Boolean {
    return value in 0.0..10.0
}
```

This checks whether the processed value is inside an acceptable range.

For example:

```text
2.25 → valid
-0.50 → invalid
25.00 → invalid
```

For now, the valid range is:

```text
0.0 to 10.0
```

This is only a simple example.

Later, the valid range should match your real sensor or measurement type.

---

## 8. Understand `movingAverage()`

```kotlin
fun movingAverage(
    values: List<Double>,
    windowSize: Int
): Double? {
    val recentValues = values.takeLast(windowSize)

    if (recentValues.isEmpty()) {
        return null
    }

    return recentValues.average()
}
```

This calculates the average of the most recent values.

Example:

```text
values = 2.1, 2.3, 2.5, 2.7, 2.9
windowSize = 3
```

The function uses:

```text
2.5, 2.7, 2.9
```

and returns:

```text
2.7
```

This is useful because many signals are processed in windows.

---

## 9. Understand `extractFeatures()`

```kotlin
fun extractFeatures(
    values: List<Double>
): SignalFeatures? {
    if (values.isEmpty()) {
        return null
    }

    val minimum = values.min()
    val maximum = values.max()

    return SignalFeatures(
        mean = values.average(),
        minimum = minimum,
        maximum = maximum,
        range = maximum - minimum
    )
}
```

This calculates simple features from a list of values.

The return type is:

```kotlin
SignalFeatures?
```

because if the list is empty, there are no features to calculate.

So:

```text
empty list
 ↓
null

non-empty list
 ↓
SignalFeatures
```

This uses Kotlin null safety.

---

## 10. Update the repository constructor

In Lesson 27, `MeasurementRepository` had:

```kotlin
class MeasurementRepository(
    context: Context,
    private val deviceDataSource: DeviceDataSource = FakeDeviceDataSource()
) {
    ...
}
```

Now add a `SignalProcessor`:

```kotlin
class MeasurementRepository(
    context: Context,
    private val deviceDataSource: DeviceDataSource = FakeDeviceDataSource(),
    private val signalProcessor: SignalProcessor = SignalProcessor()
) {
    ...
}
```

Add this import:

```kotlin
import com.example.researchapp.processing.SignalProcessor
```

Now the repository can use both:

```text
DeviceDataSource
SignalProcessor
```

The repository receives raw values from the device source and processes them using the signal processor.

---

## 11. Add a baseline value

For now, we can define a simple fixed baseline inside the repository:

```kotlin
private val baselineValue: Double = 0.2
```

So the repository contains:

```kotlin
class MeasurementRepository(
    context: Context,
    private val deviceDataSource: DeviceDataSource = FakeDeviceDataSource(),
    private val signalProcessor: SignalProcessor = SignalProcessor()
) {
    private val baselineValue: Double = 0.2

    ...
}
```

This means every raw value will be corrected by subtracting:

```text
0.2
```

Later, this could become:

```text
session baseline
device calibration value
user setting
daily blank measurement mean
```

But keep it simple for now.

---

## 12. Update `createMeasurementFromDevice()`

In Lesson 27, we had:

```kotlin
suspend fun createMeasurementFromDevice(
    sessionId: Long,
    repetition: Int
): MeasurementEntity {
    val rawValue = deviceDataSource.readValue()

    return MeasurementEntity(
        sessionId = sessionId,
        repetition = repetition,
        rawValue = rawValue,
        processedValue = rawValue,
        status = "OK"
    )
}
```

Now change it to:

```kotlin
suspend fun createMeasurementFromDevice(
    sessionId: Long,
    repetition: Int
): MeasurementEntity {
    val rawValue = deviceDataSource.readValue()

    val processedValue = signalProcessor.baselineCorrect(
        rawValue = rawValue,
        baseline = baselineValue
    )

    val status = if (
        signalProcessor.isValidValue(processedValue)
    ) {
        "OK"
    } else {
        "INVALID"
    }

    return MeasurementEntity(
        sessionId = sessionId,
        repetition = repetition,
        rawValue = rawValue,
        processedValue = processedValue,
        status = status
    )
}
```

Now the function does more:

```text
read raw value
 ↓
baseline correction
 ↓
validity check
 ↓
create MeasurementEntity
```

---

## 13. What changed?

Before Lesson 28:

```text
rawValue = fake device value
processedValue = rawValue
status = OK
```

After Lesson 28:

```text
rawValue = fake device value
processedValue = rawValue - baseline
status = OK or INVALID
```

So the measurement is now more meaningful.

Example:

```text
rawValue = 2.438
baselineValue = 0.2
processedValue = 2.238
status = OK
```

Another example:

```text
rawValue = 0.100
baselineValue = 0.2
processedValue = -0.100
status = INVALID
```

This is the first real processing step in the app.

---

## 14. Why we still save invalid values

You may wonder:

```text
If the value is invalid, should we save it?
```

For a research app, it is often useful to save it with status:

```text
INVALID
```

rather than silently deleting it.

Why?

Because later you may want to know:

```text
How many invalid readings occurred?
When did they occur?
Was the device unstable?
Was the threshold too strict?
Did invalid values happen in one session only?
```

So for now, we save the measurement but mark it:

```kotlin
status = "INVALID"
```

Later, the app can decide whether invalid values are used for inference or export.

---

## 15. Update `readAndSaveMeasurement()`

The function from Lesson 27 was:

```kotlin
suspend fun readAndSaveMeasurement(
    sessionId: Long,
    repetition: Int
): MeasurementEntity {
    val measurement = createMeasurementFromDevice(
        sessionId = sessionId,
        repetition = repetition
    )

    insertMeasurement(
        measurement = measurement
    )

    return measurement
}
```

This function can stay the same.

Why?

Because `createMeasurementFromDevice()` now already includes processing.

So:

```text
readAndSaveMeasurement()
 ↓
createMeasurementFromDevice()
 ↓
now includes SignalProcessor
```

No extra change is needed here.

That is a good sign.

The architecture allowed us to improve processing without changing the whole app.

---

## 16. Add a helper for recent processed values

Later, ML inference will need recent processed values.

So add this helper function to the repository:

```kotlin
suspend fun getRecentProcessedValuesForSession(
    sessionId: Long,
    limit: Int
): List<Double> {
    val measurements = getMeasurementsForSession(
        sessionId = sessionId
    )

    return measurements
        .map { it.processedValue }
        .takeLast(limit)
}
```

This function does:

```text
load measurements for session
 ↓
take processedValue from each measurement
 ↓
keep only the latest values
```

Example:

```text
processed values:
2.1, 2.2, 2.3, 2.4, 2.5

limit = 3

result:
2.3, 2.4, 2.5
```

This prepares us for Lesson 29.

---

## 17. Add a helper for feature extraction

Now add another repository function:

```kotlin
suspend fun extractFeaturesForSession(
    sessionId: Long,
    limit: Int = 10
): SignalFeatures? {
    val recentValues = getRecentProcessedValuesForSession(
        sessionId = sessionId,
        limit = limit
    )

    return signalProcessor.extractFeatures(
        values = recentValues
    )
}
```

Add this import:

```kotlin
import com.example.researchapp.processing.SignalFeatures
```

This function means:

```text
get recent processed values
 ↓
extract features
 ↓
return SignalFeatures or null
```

This is important for ML.

The fake model in Lesson 29 will use:

```text
SignalFeatures
```

as input.

---

## 18. Why feature extraction is in the repository

You may ask:

```text
Should feature extraction be in the repository or ViewModel?
```

For this beginner Direction A skeleton, putting this coordination in the repository is fine.

The actual math is still inside:

```text
SignalProcessor
```

The repository only coordinates:

```text
load measurements from Room
 ↓
send values to SignalProcessor
 ↓
return features
```

The ViewModel should not manually load measurement rows and calculate features.

A clean flow is:

```text
ViewModel
 ↓
repository.extractFeaturesForSession()
 ↓
SignalProcessor
```

This keeps the ViewModel simpler.

---

## 19. Updated repository imports

At the top of `MeasurementRepository.kt`, you now need:

```kotlin
import android.content.Context
import androidx.room3.Room
import com.example.researchapp.data.entity.MeasurementEntity
import com.example.researchapp.data.entity.PatientEntity
import com.example.researchapp.data.entity.ResultEntity
import com.example.researchapp.data.entity.SessionEntity
import com.example.researchapp.device.DeviceDataSource
import com.example.researchapp.device.FakeDeviceDataSource
import com.example.researchapp.processing.SignalFeatures
import com.example.researchapp.processing.SignalProcessor
```

The new imports are:

```kotlin
import com.example.researchapp.processing.SignalFeatures
import com.example.researchapp.processing.SignalProcessor
```

---

## 20. Updated repository core after Lesson 28

The important updated parts of `MeasurementRepository` are:

```kotlin
class MeasurementRepository(
    context: Context,
    private val deviceDataSource: DeviceDataSource = FakeDeviceDataSource(),
    private val signalProcessor: SignalProcessor = SignalProcessor()
) {
    private val baselineValue: Double = 0.2

    private val database = Room.databaseBuilder(
        context.applicationContext,
        ResearchDatabase::class.java,
        "research_database"
    ).build()

    private val patientDao = database.patientDao()
    private val sessionDao = database.sessionDao()
    private val measurementDao = database.measurementDao()
    private val resultDao = database.resultDao()

    suspend fun connectDevice() {
        deviceDataSource.connect()
    }

    suspend fun disconnectDevice() {
        deviceDataSource.disconnect()
    }

    suspend fun createMeasurementFromDevice(
        sessionId: Long,
        repetition: Int
    ): MeasurementEntity {
        val rawValue = deviceDataSource.readValue()

        val processedValue = signalProcessor.baselineCorrect(
            rawValue = rawValue,
            baseline = baselineValue
        )

        val status = if (
            signalProcessor.isValidValue(processedValue)
        ) {
            "OK"
        } else {
            "INVALID"
        }

        return MeasurementEntity(
            sessionId = sessionId,
            repetition = repetition,
            rawValue = rawValue,
            processedValue = processedValue,
            status = status
        )
    }

    suspend fun readAndSaveMeasurement(
        sessionId: Long,
        repetition: Int
    ): MeasurementEntity {
        val measurement = createMeasurementFromDevice(
            sessionId = sessionId,
            repetition = repetition
        )

        insertMeasurement(
            measurement = measurement
        )

        return measurement
    }

    suspend fun getRecentProcessedValuesForSession(
        sessionId: Long,
        limit: Int
    ): List<Double> {
        val measurements = getMeasurementsForSession(
            sessionId = sessionId
        )

        return measurements
            .map { it.processedValue }
            .takeLast(limit)
    }

    suspend fun extractFeaturesForSession(
        sessionId: Long,
        limit: Int = 10
    ): SignalFeatures? {
        val recentValues = getRecentProcessedValuesForSession(
            sessionId = sessionId,
            limit = limit
        )

        return signalProcessor.extractFeatures(
            values = recentValues
        )
    }

    // The patient/session/measurement/result database functions
    // from Lesson 26 stay below.
}
```

This is not the full repository file.

It shows the important new processing-related parts.

The database functions from Lesson 26 should remain.

---

## 21. Should moving average be used now?

We added:

```kotlin
movingAverage()
```

but we did not use it in `createMeasurementFromDevice()` yet.

That is okay.

At this stage, we use:

```text
baseline correction
validity check
```

first.

Moving average can be added later when we want smoother live display or window-based processing.

For example, later:

```text
read raw value
 ↓
baseline correct
 ↓
add to buffer
 ↓
moving average
 ↓
save smoothed value
```

But that needs a buffer, and we have not introduced a proper streaming buffer yet.

So for Lesson 28:

```text
keep movingAverage() available
but do not force it into the acquisition path yet
```

---

## 22. Should invalid values be used for feature extraction?

Currently, this function uses all processed values:

```kotlin
return measurements
    .map { it.processedValue }
    .takeLast(limit)
```

That includes invalid values.

For a more careful version, we can use only valid measurements:

```kotlin
return measurements
    .filter { it.status == "OK" }
    .map { it.processedValue }
    .takeLast(limit)
```

For a research app, this is probably better.

So I recommend using this version:

```kotlin
suspend fun getRecentProcessedValuesForSession(
    sessionId: Long,
    limit: Int
): List<Double> {
    val measurements = getMeasurementsForSession(
        sessionId = sessionId
    )

    return measurements
        .filter { it.status == "OK" }
        .map { it.processedValue }
        .takeLast(limit)
}
```

Now ML features are extracted from valid values only.

The raw invalid values are still stored, but they are not used for feature extraction.

That is a good balance.

---

## 23. Update display logic later

The UI is not fully connected yet, but later the ViewModel can display:

```text
latest raw value
latest processed value
latest status
```

For example:

```kotlin
uiState = uiState.copy(
    latestRawValue = measurement.rawValue,
    latestProcessedValue = measurement.processedValue,
    latestStatus = measurement.status
)
```

Then the screen can show:

```text
Raw value: 2.438
Processed value: 2.238
Status: OK
```

This helps the researcher understand what is happening.

---

## 24. Why this structure is useful

After Lesson 28, the repository uses three different things:

```text
Room database
DeviceDataSource
SignalProcessor
```

The architecture becomes:

```text
MeasurementRepository
 ├── ResearchDatabase
 ├── DeviceDataSource
 └── SignalProcessor
```

This is useful because each part has a clear responsibility.

```text
DeviceDataSource
 ↓
gets raw data

SignalProcessor
 ↓
processes raw data

ResearchDatabase
 ↓
stores data

MeasurementRepository
 ↓
coordinates the full operation
```

This is the core of a real research acquisition pipeline.

---

## 25. Current architecture after Lesson 28

The fake acquisition path is now:

```text
FakeDeviceDataSource.readValue()
 ↓
rawValue
 ↓
SignalProcessor.baselineCorrect()
 ↓
processedValue
 ↓
SignalProcessor.isValidValue()
 ↓
status
 ↓
MeasurementEntity
 ↓
Room database
```

More generally:

```text
Device source
 ↓
raw data

Processing layer
 ↓
processed data

Repository
 ↓
adds session context and saves

Room
 ↓
stores research data
```

This is a major improvement over simply saving random numbers.

---

## 26. Current files after Lesson 28

After this lesson, the project should include:

```text
processing
 ├── SignalProcessor.kt
 └── SignalFeatures.kt
```

And `MeasurementRepository.kt` should now use:

```text
SignalProcessor
SignalFeatures
```

Your project structure is now:

```text
com.example.researchapp
 ├── data
 │    ├── ResearchDatabase.kt
 │    ├── MeasurementRepository.kt
 │    ├── entity
 │    └── dao
 │
 ├── device
 │    ├── DeviceDataSource.kt
 │    └── FakeDeviceDataSource.kt
 │
 ├── processing
 │    ├── SignalProcessor.kt
 │    └── SignalFeatures.kt
 │
 ├── ml
 ├── ui
 ├── viewmodel
 └── export
```

The app now has:

```text
database layer
repository layer
fake device layer
processing layer
```

---

## 27. What you learned in Lesson 28

You created:

```text
SignalProcessor
SignalFeatures
```

You learned that `SignalProcessor` should handle:

```text
baseline correction
validity checking
moving average
feature extraction
```

You updated the repository so that:

```text
processedValue = rawValue - baseline
```

instead of:

```text
processedValue = rawValue
```

You also added status logic:

```text
processed value inside valid range
 ↓
status = OK

processed value outside valid range
 ↓
status = INVALID
```

The most important mental model is:

```text
Raw data should pass through a processing layer before becoming research data.
```

The new acquisition path is:

```text
DeviceDataSource
 ↓
raw value

SignalProcessor
 ↓
processed value and status

MeasurementRepository
 ↓
MeasurementEntity with session context

Room database
 ↓
saved research record
```

This prepares the app for ML inference because ML should usually receive processed features, not raw unstructured values.

---

## 28. Lesson 29 preview

In Lesson 29, we will add fake ML inference into the real project.

We will implement:

```text
ModelRunner
FakeModelRunner
PredictionResult
```

and update the repository so it can:

```text
load recent processed values
extract SignalFeatures
run fake inference
save ResultEntity
```

This will let the app produce a fake prediction before connecting a real LiteRT/TFLite model.