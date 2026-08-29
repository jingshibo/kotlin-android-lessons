# Lesson 19 — Signal Processing Pipeline in the App

In Lesson 18, we introduced a cleaner data-source structure:

```text
MeasurementScreen
 ↓
ResearchViewModel
 ↓
MeasurementRepository
 ↓
DeviceDataSource
 ↓
Fake / Bluetooth / Wi-Fi / USB device
```

Before Lesson 18, the measurement value came directly from:

```kotlin
Random.nextDouble(0.0, 5.0)
```

After Lesson 18, the repository asked a data source:

```kotlin
deviceDataSource.readValue()
```

This follows the original tutorial’s idea: first use fake/random data, then later replace it with real device input after the app logic works. fileciteturn0file0L140-L144

Now we add the next important part of a research app:

```text
signal processing
```

Many research apps do not simply save the raw device value directly. Usually, the data goes through a pipeline:

```text
raw data
 ↓
processing
 ↓
feature extraction
 ↓
result or classification
 ↓
save/export
```

---

## 1. Why signal processing is needed

A real sensor value may contain:

```text
noise
drift
baseline offset
unstable readings
spikes
invalid values
movement artefacts
temperature effects
device variation
```

If we save or classify raw data directly, the result may be unstable.

For example, suppose the device sends:

```text
2.41
2.43
2.45
8.90
2.44
2.42
```

The value:

```text
8.90
```

may be a spike.

A simple research app should at least think about:

```text
Should we remove invalid values?
Should we smooth the signal?
Should we subtract a baseline?
Should we calculate features?
Should we save raw and processed values?
```

So Lesson 19 introduces a simple processing pipeline.

---

## 2. Raw data vs processed data

This distinction is very important.

```text
Raw data
 ↓
the value received directly from the device

Processed data
 ↓
the value after filtering, smoothing, correction, or feature calculation
```

For example:

```text
Raw value: 2.438

Processed value after baseline correction:
2.438 - 0.200 = 2.238
```

or:

```text
Raw values:
2.41, 2.43, 2.45

Processed feature:
mean = 2.43
```

For a research app, I strongly suggest keeping this habit:

```text
save raw data when possible
save processed data or results separately
```

Why?

Because if you later change your processing method, you can reprocess the original data.

If you only save processed values, you may lose important information.

---

## 3. Where should signal processing code live?

Do **not** put signal-processing code directly inside the UI.

Avoid this:

```text
MeasurementScreen
 └── smoothing, baseline correction, feature extraction
```

A better structure is:

```text
MeasurementScreen
 ↓
ResearchViewModel
 ↓
MeasurementRepository
 ↓
SignalProcessor
 ↓
MeasurementEntity saved to Room
```

The UI should only show:

```text
latest value
processed value
status
result
buttons
measurement history
```

The processing code should live in a separate class.

For Lesson 19, we can create:

```text
SignalProcessor
```

or:

```text
MeasurementProcessor
```

I will use:

```kotlin
SignalProcessor
```

because later it can handle more general signal processing.

---

## 4. First simple `SignalProcessor`

Create a new Kotlin file:

```text
SignalProcessor.kt
```

Start with this:

```kotlin
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
}
```

This gives us two simple operations:

```text
baselineCorrect()
 ↓
subtract baseline from raw value

isValidValue()
 ↓
check whether the value is within an acceptable range
```

This is intentionally simple.

The goal is not advanced signal processing yet.

The goal is to introduce the correct app structure.

---

## 5. Baseline correction

Baseline correction means:

```text
remove a reference value from the measurement
```

For example:

```kotlin
val rawValue = 2.45
val baseline = 0.20

val correctedValue = rawValue - baseline
```

Result:

```text
2.25
```

In code:

```kotlin
fun baselineCorrect(
    rawValue: Double,
    baseline: Double
): Double {
    return rawValue - baseline
}
```

For your research app, the baseline could later come from:

```text
blank measurement
device-only reading
daily calibration
pre-session baseline
stored reference value
```

For now, we can use a fixed baseline:

```kotlin
val baseline = 0.2
```

Later, we can make it user-configurable or session-specific.

---

## 6. Value validation

Before saving a measurement, we may want to reject impossible values.

For example:

```kotlin
fun isValidValue(
    value: Double
): Boolean {
    return value in 0.0..10.0
}
```

This means:

```text
valid if value is between 0 and 10
invalid otherwise
```

Example:

```text
2.45 → valid
-1.00 → invalid
50.00 → invalid
```

This connects back to Lesson 2, where we learned to filter invalid readings using ranges and conditions. The original tutorial used examples like keeping only readings between 0 and 10. fileciteturn1file0L231-L237

---

## 7. Add status from processing

Instead of always storing:

```kotlin
status = "OK"
```

we can set status based on validation.

For example:

```kotlin
val status = if (signalProcessor.isValidValue(correctedValue)) {
    "OK"
} else {
    "INVALID"
}
```

So the processing flow becomes:

```text
raw value
 ↓
baseline correction
 ↓
validity check
 ↓
status = OK or INVALID
```

This is already more realistic than blindly saving every value as OK.

---

## 8. Update `MeasurementEntity`

In earlier lessons, our measurement entity had:

```kotlin
data class MeasurementEntity(
    val id: Long = 0,
    val sessionId: Long,
    val repetition: Int,
    val value: Double,
    val timestamp: Long,
    val status: String
)
```

Now we may want to store both:

```text
rawValue
processedValue
```

So a better version is:

```kotlin
@Entity(tableName = "measurements")
data class MeasurementEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,

    val sessionId: Long,
    val repetition: Int,

    val rawValue: Double,
    val processedValue: Double,

    val timestamp: Long = System.currentTimeMillis(),
    val status: String = "OK"
)
```

Important note:

```text
Changing a Room entity changes the database schema.
```

For learning, if you are using an emulator or test tablet, you may uninstall and reinstall the app to reset the database.

For a real deployed research app, you would need a proper Room migration.

Do not ignore migrations once real data has been collected.

---

## 9. Update repository to use `SignalProcessor`

In Lesson 18, the repository had:

```kotlin
class MeasurementRepository(
    context: Context,
    private val deviceDataSource: DeviceDataSource = FakeDeviceDataSource()
) {
    ...
}
```

Now add a signal processor:

```kotlin
class MeasurementRepository(
    context: Context,
    private val deviceDataSource: DeviceDataSource = FakeDeviceDataSource(),
    private val signalProcessor: SignalProcessor = SignalProcessor()
) {
    ...
}
```

This means the repository can:

```text
read raw value from device
process it
create MeasurementEntity
save it to Room
```

The ViewModel still does not need to know the processing details.

---

## 10. Create measurement with processing

Before:

```kotlin
suspend fun createMeasurementFromDevice(
    sessionId: Long,
    repetition: Int
): MeasurementEntity {
    val value = deviceDataSource.readValue()

    return MeasurementEntity(
        sessionId = sessionId,
        repetition = repetition,
        value = value,
        timestamp = System.currentTimeMillis(),
        status = "OK"
    )
}
```

Now:

```kotlin
suspend fun createMeasurementFromDevice(
    sessionId: Long,
    repetition: Int
): MeasurementEntity {
    val rawValue = deviceDataSource.readValue()

    val baseline = 0.2

    val processedValue = signalProcessor.baselineCorrect(
        rawValue = rawValue,
        baseline = baseline
    )

    val status = if (signalProcessor.isValidValue(processedValue)) {
        "OK"
    } else {
        "INVALID"
    }

    return MeasurementEntity(
        sessionId = sessionId,
        repetition = repetition,
        rawValue = rawValue,
        processedValue = processedValue,
        timestamp = System.currentTimeMillis(),
        status = status
    )
}
```

The flow is now:

```text
deviceDataSource.readValue()
 ↓
rawValue
 ↓
baselineCorrect()
 ↓
processedValue
 ↓
isValidValue()
 ↓
MeasurementEntity
```

This is a simple signal-processing pipeline.

---

## 11. What should the UI display?

Previously, the UI displayed only:

```text
latest value
```

Now it can display:

```text
latest raw value
latest processed value
latest status
```

So update `ResearchUiState`.

Before:

```kotlin
val latestValue: Double? = null
```

Now:

```kotlin
val latestRawValue: Double? = null
val latestProcessedValue: Double? = null
val latestStatus: String = ""
```

Example:

```kotlin
data class ResearchUiState(
    val patientCode: String = "",
    val sessionName: String = "",
    val currentPatientId: Long? = null,
    val currentSessionId: Long? = null,

    val deviceConnectionState: DeviceConnectionState = DeviceConnectionState.DISCONNECTED,
    val acquisitionState: AcquisitionState = AcquisitionState.IDLE,
    val bluetoothPermissionState: PermissionState = PermissionState.UNKNOWN,

    val measurements: List<MeasurementEntity> = emptyList(),

    val latestRawValue: Double? = null,
    val latestProcessedValue: Double? = null,
    val latestStatus: String = "",

    val isLoading: Boolean = false,
    val message: String = ""
)
```

This makes the UI more informative.

---

## 12. Update ViewModel after saving measurement

In Lesson 18, after adding a measurement, the ViewModel updated:

```kotlin
latestValue = newMeasurement.value
```

Now the measurement has:

```kotlin
rawValue
processedValue
status
```

So update:

```kotlin
uiState = uiState.copy(
    measurements = updatedMeasurements,
    latestRawValue = newMeasurement.rawValue,
    latestProcessedValue = newMeasurement.processedValue,
    latestStatus = newMeasurement.status,
    message = "Measurement processed and saved"
)
```

The full function becomes:

```kotlin
private suspend fun addMeasurementFromDevice(
    sessionId: Long
) {
    val newMeasurement =
        measurementRepository.createMeasurementFromDevice(
            sessionId = sessionId,
            repetition = uiState.measurements.size + 1
        )

    measurementRepository.insertMeasurement(newMeasurement)

    val updatedMeasurements =
        measurementRepository.getMeasurementsForSession(sessionId)

    uiState = uiState.copy(
        measurements = updatedMeasurements,
        latestRawValue = newMeasurement.rawValue,
        latestProcessedValue = newMeasurement.processedValue,
        latestStatus = newMeasurement.status,
        message = "Measurement processed and saved"
    )
}
```

This keeps the ViewModel simple.

It does not do the processing itself.

It only receives a processed `MeasurementEntity`.

---

## 13. Display raw and processed values

In `MeasurementScreen`, create display text:

```kotlin
val latestRawText = uiState.latestRawValue?.let {
    "%.3f".format(it)
} ?: "No data"

val latestProcessedText = uiState.latestProcessedValue?.let {
    "%.3f".format(it)
} ?: "No data"
```

Then display:

```kotlin
Text("Latest raw value: $latestRawText")
Text("Latest processed value: $latestProcessedText")
Text("Latest status: ${uiState.latestStatus}")
```

Example UI:

```text
Latest raw value: 2.438
Latest processed value: 2.238
Latest status: OK
```

This helps the user understand the data flow.

---

## 14. Add a simple moving average

Baseline correction processes one value at a time.

Another common idea is smoothing.

A simple smoothing method is:

```text
moving average
```

For example, if the latest values are:

```text
2.1, 2.3, 2.5
```

The moving average is:

```text
(2.1 + 2.3 + 2.5) / 3 = 2.3
```

Add this to `SignalProcessor`:

```kotlin
fun movingAverage(
    values: List<Double>
): Double? {
    if (values.isEmpty()) {
        return null
    }

    return values.average()
}
```

This returns:

```text
Double
```

if values exist, or:

```text
null
```

if the list is empty.

---

## 15. Moving average over recent values

Usually, we do not average all historical values.

We average only the recent window.

For example, the last 5 values:

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

Example:

```kotlin
val smoothedValue = signalProcessor.movingAverage(
    values = listOf(2.1, 2.3, 2.5, 2.7, 2.9),
    windowSize = 3
)
```

This uses:

```text
2.5, 2.7, 2.9
```

and returns:

```text
2.7
```

This is a simple version of window-based processing.

---

## 16. Why window-based processing matters

Many real research signals are not processed sample by sample.

They are processed in windows.

For example:

```text
collect 100 samples
 ↓
calculate mean, max, min, standard deviation
 ↓
feed features into ML model
 ↓
get result
```

This is common for:

```text
wearable sensor signals
EMG
IMU
VNA frequency responses
temperature/time-series signals
biosensor readings
```

So this lesson starts with simple values, but the mental model is larger:

```text
streaming data
 ↓
buffer/window
 ↓
features
 ↓
result
```

---

## 17. Add simple feature extraction

A feature is a summary value calculated from raw or processed data.

Examples:

```text
mean
minimum
maximum
range
standard deviation
slope
peak value
```

Create a simple data class:

```kotlin
data class SignalFeatures(
    val mean: Double,
    val minimum: Double,
    val maximum: Double,
    val range: Double
)
```

Add a function to `SignalProcessor`:

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

This function takes a list of values and returns useful summary features.

This connects the app to later ML inference.

---

## 18. Use recent measurements for features

Inside the ViewModel or repository, we can get recent processed values:

```kotlin
val recentProcessedValues = updatedMeasurements
    .map { it.processedValue }
    .takeLast(10)
```

Then:

```kotlin
val features = signalProcessor.extractFeatures(
    values = recentProcessedValues
)
```

But where should this happen?

For now, keep feature extraction in the repository or a processing class, not the UI.

A simple pattern is:

```text
Repository gets data
 ↓
SignalProcessor extracts features
 ↓
Repository/ViewModel updates result state
 ↓
UI displays features
```

---

## 19. Should features be saved?

For a research app, there are two possible approaches.

### Option 1: Save raw/processed measurements only

Then calculate features later when needed.

```text
Room stores raw and processed measurements.
Features are calculated on demand.
```

This is flexible.

### Option 2: Save features too

```text
Room stores measurements.
Room also stores features/results.
```

This is useful if feature extraction is expensive or if the exact feature values used for an ML prediction must be recorded.

For this tutorial, I would use:

```text
save raw and processed measurements first
calculate features when needed
save final result later
```

This keeps the app simpler.

---

## 20. Add processing status to the app flow

After Lesson 19, the acquisition path becomes:

```text
Start Acquisition
 ↓
DeviceDataSource.readValue()
 ↓
rawValue
 ↓
SignalProcessor.baselineCorrect()
 ↓
processedValue
 ↓
SignalProcessor.isValidValue()
 ↓
MeasurementEntity(rawValue, processedValue, status)
 ↓
Room database
 ↓
UI state updates
```

This is a real processing pipeline.

Still simple, but correctly structured.

---

## 21. Updated architecture after Lesson 19

The app now looks like this:

```text
MeasurementScreen
 ↓
ResearchViewModel
 ↓
MeasurementRepository
 ├── DeviceDataSource
 ├── SignalProcessor
 └── Room database
```

More explicitly:

```text
DeviceDataSource
 ↓
gets raw value

SignalProcessor
 ↓
processes raw value

MeasurementRepository
 ↓
creates and saves MeasurementEntity

ResearchViewModel
 ↓
updates UI state

MeasurementScreen
 ↓
shows latest raw/processed/status
```

This is the right direction.

---

## 22. Why this prepares us for ML inference

Machine-learning models usually do not want unstructured app state.

They usually expect:

```text
specific input shape
specific feature order
specific scaling/normalisation
specific data type
```

For example, later we may have:

```text
recent 100 processed values
 ↓
extract features
 ↓
convert to FloatArray
 ↓
run LiteRT/TFLite model
 ↓
get prediction
```

So signal processing is the bridge between:

```text
raw sensor data
```

and:

```text
ML inference
```

That is why this lesson belongs before on-device ML.

---

## 23. Important research-app habit

Do not hide too much processing from yourself.

For research, it is useful to keep track of:

- raw value
- baseline value
- processed value
- status
- processing method/version
- timestamp
- session ID

Because later, when analysing the data, you may need to answer:

```text
Was this result based on raw or processed data?
What baseline was used?
Were invalid values excluded?
What feature window was used?
Which processing version was used?
```

A production research app should eventually record this metadata.

For now, we keep it simple.

But the habit is important.

---

## 24. What You Learned in Lesson 19

The key new class is:

```kotlin
class SignalProcessor {
    ...
}
```

The key processing functions are:

```kotlin
fun baselineCorrect(
    rawValue: Double,
    baseline: Double
): Double
```

```kotlin
fun isValidValue(
    value: Double
): Boolean
```

```kotlin
fun movingAverage(
    values: List<Double>,
    windowSize: Int
): Double?
```

```kotlin
fun extractFeatures(
    values: List<Double>
): SignalFeatures?
```

The measurement entity can now store:

```kotlin
val rawValue: Double
val processedValue: Double
val status: String
```

The most important mental model is:

```text
Device data should pass through a processing pipeline before becoming a research result.
```

The pipeline is:

```text
raw data
 ↓
processing
 ↓
features
 ↓
ML/result later
 ↓
save/export
```

For a research app, this is essential because the quality of the final result depends not only on the model, but also on how the sensor data is processed.

## Lesson 20 Preview

In Lesson 20, we should introduce:

**On-device ML inference**

The next step is to take processed values or extracted features and feed them into a model.
Lesson 20 should cover:

- what on-device inference means
- where the model file goes
- how app data becomes model input
- `FloatArray` input
- prediction output
- saving result to Room
- displaying result on ResultScreen

That will make the app move from:

```text
data collection app
    ↓
edge-AI research app
```
