# Lesson 29 — Adding Fake ML Inference

In Lesson 28, we connected the signal-processing layer.

The acquisition path became:

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

Now we add the next part:

```text
fake ML inference
```

This corresponds to **Step A7** in Direction A:

```text
Add fake ML model
```

The goal of Lesson 29 is to let the app produce a fake prediction before we connect a real LiteRT/TFLite model.

After this lesson, the data flow becomes:

```text
Measurements in Room
 ↓
recent processed values
 ↓
SignalProcessor.extractFeatures()
 ↓
FakeModelRunner
 ↓
PredictionResult
 ↓
ResultEntity
 ↓
Room database
```

---

## 1. Why use fake ML first?

A real ML model requires many details:

```text
.tflite model file
model input shape
feature order
normalisation
output interpretation
label mapping
model version
```

If we start with the real model too early, many problems may happen at once.

For example:

```text
Is the app code wrong?
Is the model input wrong?
Is the feature order wrong?
Is the model conversion wrong?
Is the output interpretation wrong?
```

That is difficult to debug.

So we first create a fake model.

The fake model lets us test the app flow:

```text
load measurements
 ↓
extract features
 ↓
run inference
 ↓
save result
 ↓
show result later
```

without needing the real model yet.

This follows the same pattern:

```text
fake device first
fake model first
real device/model later
```

---

## 2. Where ML code belongs

In Lesson 23, we planned this folder:

```text
ml
 ├── ModelRunner.kt
 ├── FakeModelRunner.kt
 └── PredictionResult.kt
```

So now create:

```text
app/src/main/java/com/example/researchapp/ml
```

The `ml` package is for:

```text
fake ML inference now
real LiteRT/TFLite inference later
```

Do not put model logic inside:

```text
MeasurementScreen.kt
ResearchViewModel.kt
MeasurementRepository.kt directly
```

The repository can coordinate ML inference, but the model logic itself should live in the `ml` layer.

---

## 3. Create `PredictionResult.kt`

Create:

```text
ml/PredictionResult.kt
```

Code:

```kotlin
package com.example.researchapp.ml

data class PredictionResult(
    val label: String,
    val confidence: Double
)
```

This class represents the output of model inference.

Example:

```kotlin
PredictionResult(
    label = "Positive",
    confidence = 0.90
)
```

or:

```kotlin
PredictionResult(
    label = "Negative",
    confidence = 0.85
)
```

For your real app, labels may later become:

```text
Positive / Negative
Class A / Class B / Class C
Valid / Invalid
Disease / No disease
```

The exact labels depend on your trained model.

---

## 4. Create `ModelRunner.kt`

Create:

```text
ml/ModelRunner.kt
```

Code:

```kotlin
package com.example.researchapp.ml

import com.example.researchapp.processing.SignalFeatures

interface ModelRunner {

    suspend fun runInference(
        features: SignalFeatures
    ): PredictionResult
}
```

This interface says:

```text
Any model runner must take SignalFeatures
and return PredictionResult.
```

The app does not need to know whether the model is:

```text
fake rule-based model
real LiteRT model
real TFLite model
cloud model later
```

It only calls:

```kotlin
modelRunner.runInference(features)
```

---

## 5. Why use a `ModelRunner` interface?

This is the same idea as `DeviceDataSource`.

For device data, we created:

```kotlin
interface DeviceDataSource {
    suspend fun connect()
    suspend fun disconnect()
    suspend fun readValue(): Double
}
```

For ML inference, we now create:

```kotlin
interface ModelRunner {
    suspend fun runInference(
        features: SignalFeatures
    ): PredictionResult
}
```

The pattern is:

```text
Interface
 ↓
stable doorway

Fake implementation
 ↓
used for development

Real implementation
 ↓
added later
```

So:

```text
ModelRunner
 ├── FakeModelRunner
 └── LiteRtModelRunner later
```

This means we can build the app workflow first and replace the model later.

---

## 6. Create `FakeModelRunner.kt`

Create:

```text
ml/FakeModelRunner.kt
```

Code:

```kotlin
package com.example.researchapp.ml

import com.example.researchapp.processing.SignalFeatures
import kotlinx.coroutines.delay

class FakeModelRunner : ModelRunner {

    override suspend fun runInference(
        features: SignalFeatures
    ): PredictionResult {
        delay(500)

        val label = if (features.mean > 2.5) {
            "Positive"
        } else {
            "Negative"
        }

        val confidence = if (features.mean > 2.5) {
            0.90
        } else {
            0.85
        }

        return PredictionResult(
            label = label,
            confidence = confidence
        )
    }
}
```

This fake model uses a simple rule:

```text
if mean > 2.5
 ↓
Positive

if mean <= 2.5
 ↓
Negative
```

This is not real ML.

It is only a fake placeholder.

But it lets the app test the complete inference path.

---

## 7. Why add `delay(500)`?

The fake model contains:

```kotlin
delay(500)
```

This simulates model inference taking some time.

A real model may take:

```text
a few milliseconds
tens of milliseconds
hundreds of milliseconds
```

depending on model size and device performance.

Adding a small delay helps us test loading messages later, such as:

```text
Running inference...
Inference complete
```

The fake model should behave a little like a real asynchronous model call.

---

## 8. What input does the fake model use?

The fake model uses:

```kotlin
features.mean
```

from:

```kotlin
data class SignalFeatures(
    val mean: Double,
    val minimum: Double,
    val maximum: Double,
    val range: Double
)
```

For example:

```text
recent processed values:
2.1, 2.3, 2.5

features.mean = 2.3

Prediction:
Negative
```

Another example:

```text
recent processed values:
2.8, 2.9, 3.0

features.mean = 2.9

Prediction:
Positive
```

Again, this is only fake logic.

Later, the real model may use:

```text
mean
minimum
maximum
range
standard deviation
raw signal window
multi-channel features
frequency-domain features
```

But for now, simple features are enough.

---

## 9. Update `MeasurementRepository`

In Lesson 28, the repository constructor looked like this:

```kotlin
class MeasurementRepository(
    context: Context,
    private val deviceDataSource: DeviceDataSource = FakeDeviceDataSource(),
    private val signalProcessor: SignalProcessor = SignalProcessor()
) {
    ...
}
```

Now add a `ModelRunner`:

```kotlin
class MeasurementRepository(
    context: Context,
    private val deviceDataSource: DeviceDataSource = FakeDeviceDataSource(),
    private val signalProcessor: SignalProcessor = SignalProcessor(),
    private val modelRunner: ModelRunner = FakeModelRunner()
) {
    ...
}
```

Add these imports:

```kotlin
import com.example.researchapp.ml.FakeModelRunner
import com.example.researchapp.ml.ModelRunner
import com.example.researchapp.ml.PredictionResult
```

Now the repository can use:

```text
DeviceDataSource
SignalProcessor
ModelRunner
Room database
```

---

## 10. Add inference function to repository

In Lesson 28, we added:

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

Now we use those features for inference.

Add this function:

```kotlin
suspend fun runInferenceForSession(
    sessionId: Long,
    featureLimit: Int = 10
): PredictionResult? {
    val features = extractFeaturesForSession(
        sessionId = sessionId,
        limit = featureLimit
    ) ?: return null

    return modelRunner.runInference(
        features = features
    )
}
```

This function means:

```text
extract features for this session
 ↓
if features are available, run model
 ↓
return prediction
```

The return type is:

```kotlin
PredictionResult?
```

because inference may not be possible yet.

For example:

```text
no measurements
 ↓
no features
 ↓
no prediction
 ↓
return null
```

---

## 11. Why return nullable `PredictionResult?`

The app may not always have enough data.

For example:

```text
current session has zero valid measurements
```

Then this function:

```kotlin
extractFeaturesForSession(...)
```

returns:

```kotlin
null
```

So this function also returns:

```kotlin
null
```

That means the ViewModel can later show:

```text
Not enough data for inference
```

This is better than crashing.

---

## 12. Save fake inference result to Room

Running inference is not enough.

For a research app, we should save the result.

We already have:

```kotlin
ResultEntity
```

from Lesson 24:

```kotlin
@Entity(tableName = "results")
data class ResultEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,

    val sessionId: Long,
    val label: String,
    val confidence: Double,
    val createdAt: Long = System.currentTimeMillis()
)
```

Now add this repository function:

```kotlin
suspend fun runAndSaveInferenceForSession(
    sessionId: Long,
    featureLimit: Int = 10
): PredictionResult? {
    val prediction = runInferenceForSession(
        sessionId = sessionId,
        featureLimit = featureLimit
    ) ?: return null

    val result = ResultEntity(
        sessionId = sessionId,
        label = prediction.label,
        confidence = prediction.confidence
    )

    insertResult(
        result = result
    )

    return prediction
}
```

This function does:

```text
run inference
 ↓
create ResultEntity
 ↓
save ResultEntity to Room
 ↓
return PredictionResult
```

This is the main fake ML workflow.

---

## 13. Why save `ResultEntity` but return `PredictionResult`?

The repository saves:

```kotlin
ResultEntity
```

because Room stores entities.

But the UI usually does not need the whole database entity immediately.

The UI mainly needs:

```text
label
confidence
```

So the function returns:

```kotlin
PredictionResult
```

This is cleaner for display.

The app stores the database record and returns a simple display-friendly result.

---

## 14. Updated repository constructor and inference functions

The important new repository parts are:

```kotlin
class MeasurementRepository(
    context: Context,
    private val deviceDataSource: DeviceDataSource = FakeDeviceDataSource(),
    private val signalProcessor: SignalProcessor = SignalProcessor(),
    private val modelRunner: ModelRunner = FakeModelRunner()
) {
    private val baselineValue: Double = 0.2

    // database and DAO setup stay here

    suspend fun runInferenceForSession(
        sessionId: Long,
        featureLimit: Int = 10
    ): PredictionResult? {
        val features = extractFeaturesForSession(
            sessionId = sessionId,
            limit = featureLimit
        ) ?: return null

        return modelRunner.runInference(
            features = features
        )
    }

    suspend fun runAndSaveInferenceForSession(
        sessionId: Long,
        featureLimit: Int = 10
    ): PredictionResult? {
        val prediction = runInferenceForSession(
            sessionId = sessionId,
            featureLimit = featureLimit
        ) ?: return null

        val result = ResultEntity(
            sessionId = sessionId,
            label = prediction.label,
            confidence = prediction.confidence
        )

        insertResult(
            result = result
        )

        return prediction
    }

    // Other repository functions stay below.
}
```

The repository is now becoming the central coordinator for the research pipeline.

---

## 15. Full inference path after Lesson 29

After this lesson, inference works like this:

```text
Room database
 ↓
getMeasurementsForSession(sessionId)

MeasurementRepository
 ↓
filter valid measurements
 ↓
take latest processed values

SignalProcessor
 ↓
extract SignalFeatures

FakeModelRunner
 ↓
run fake rule-based inference

ResultEntity
 ↓
save to Room

PredictionResult
 ↓
return to ViewModel later
```

This is the complete fake inference path.

---

## 16. Why the model should use processed values

The repository uses:

```kotlin
getRecentProcessedValuesForSession(...)
```

not raw values.

That is intentional.

The model should usually use values after preprocessing.

The flow is:

```text
raw value
 ↓
processed value
 ↓
features
 ↓
model
```

not:

```text
raw value
 ↓
model directly
```

unless your model was specifically trained on raw data.

For this tutorial, the model uses processed features.

That matches our architecture.

---

## 17. Should invalid measurements be used?

In Lesson 28, I suggested this version:

```kotlin
return measurements
    .filter { it.status == "OK" }
    .map { it.processedValue }
    .takeLast(limit)
```

This means:

```text
invalid measurements are saved
but not used for feature extraction
```

That is a good beginner-friendly research rule.

So the inference path uses only:

```text
status = OK
```

measurements.

This prevents clearly invalid values from affecting the fake prediction.

---

## 18. What if there are fewer than 10 measurements?

Currently, `featureLimit = 10`.

But the code can still extract features from fewer values.

For example:

```text
valid processed values:
2.1, 2.2, 2.4

featureLimit = 10
```

Then `takeLast(10)` returns all three values.

The model can still run.

That is okay for a simple fake model.

But for a real ML model, you may need a fixed number of values.

For example:

```text
model requires exactly 100 samples
```

Then you should check:

```text
Do we have enough values?
```

For now, simple is fine.

Later, real ML integration will require stricter input-shape checking.

---

## 19. Optional stricter feature requirement

If you want the fake inference to require at least 10 valid values, you can modify the function.

For example:

```kotlin
suspend fun extractFeaturesForSession(
    sessionId: Long,
    limit: Int = 10
): SignalFeatures? {
    val recentValues = getRecentProcessedValuesForSession(
        sessionId = sessionId,
        limit = limit
    )

    if (recentValues.size < limit) {
        return null
    }

    return signalProcessor.extractFeatures(
        values = recentValues
    )
}
```

Then:

```text
fewer than 10 valid values
 ↓
not enough data
 ↓
no prediction
```

This is more realistic.

But for the first fake workflow, I would not make it too strict yet.

I would allow inference with any non-empty valid values first, then make it stricter later.

---

## 20. Add current prediction to UI state later

The ViewModel is not fully connected yet, but later `ResearchUiState` should include:

```kotlin
val latestPrediction: PredictionResult? = null
```

This requires the import:

```kotlin
import com.example.researchapp.ml.PredictionResult
```

Then the ViewModel can update:

```kotlin
uiState = uiState.copy(
    latestPrediction = prediction,
    message = "Inference complete"
)
```

The UI can show:

```text
Prediction: Positive
Confidence: 0.90
```

We will connect this in a later screen/ViewModel lesson.

Lesson 29 focuses on the repository and ML layer.

---

## 21. Example future ViewModel call

Later, the ViewModel can call:

```kotlin
fun runInferenceForCurrentSession() {
    val sessionId = uiState.currentSessionId

    if (sessionId == null) {
        uiState = uiState.copy(
            message = "No active session for inference"
        )
        return
    }

    viewModelScope.launch {
        try {
            val prediction =
                measurementRepository.runAndSaveInferenceForSession(
                    sessionId = sessionId
                )

            if (prediction == null) {
                uiState = uiState.copy(
                    message = "Not enough data for inference"
                )
            } else {
                uiState = uiState.copy(
                    latestPrediction = prediction,
                    message = "Inference complete"
                )
            }
        } catch (e: Exception) {
            uiState = uiState.copy(
                message = "Inference failed"
            )
        }
    }
}
```

This shows how the ViewModel will use the repository later.

But for now, the repository function is the key.

---

## 22. Why not put fake model logic inside repository?

We could write:

```kotlin
val label = if (features.mean > 2.5) {
    "Positive"
} else {
    "Negative"
}
```

directly inside the repository.

But that would mix responsibilities.

Better:

```text
Repository
 ↓
coordinates inference

ModelRunner
 ↓
contains model logic
```

This makes it easier to replace fake ML later.

When the real model is ready, we can replace:

```text
FakeModelRunner
```

with:

```text
LiteRtModelRunner
```

without rewriting the repository logic too much.

---

## 23. What changed from Lesson 28?

In Lesson 28, the repository used:

```text
DeviceDataSource
SignalProcessor
Room database
```

After Lesson 29, it also uses:

```text
ModelRunner
```

So the repository now coordinates:

```text
device reading
processing
database saving
feature extraction
fake ML inference
result saving
```

The architecture becomes:

```text
MeasurementRepository
 ├── ResearchDatabase
 ├── DeviceDataSource
 ├── SignalProcessor
 └── ModelRunner
```

This is now a complete fake research pipeline.

---

## 24. Current architecture after Lesson 29

After this lesson, the app architecture becomes:

```text
ResearchViewModel
 ↓
MeasurementRepository
 ├── DeviceDataSource
 │    └── FakeDeviceDataSource
 │
 ├── SignalProcessor
 │    └── SignalFeatures
 │
 ├── ModelRunner
 │    └── FakeModelRunner
 │
 └── ResearchDatabase
      ├── PatientDao
      ├── SessionDao
      ├── MeasurementDao
      └── ResultDao
```

The fake end-to-end pipeline is:

```text
FakeDeviceDataSource
 ↓
rawValue

SignalProcessor
 ↓
processedValue

Room
 ↓
saved MeasurementEntity

SignalProcessor
 ↓
SignalFeatures from recent values

FakeModelRunner
 ↓
PredictionResult

Room
 ↓
saved ResultEntity
```

This is a strong foundation for the app.

---

## 25. Current files after Lesson 29

After this lesson, the project should include:

```text
ml
 ├── PredictionResult.kt
 ├── ModelRunner.kt
 └── FakeModelRunner.kt
```

And `MeasurementRepository.kt` should now use:

```text
ModelRunner
FakeModelRunner
PredictionResult
ResultEntity
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
 │    ├── PredictionResult.kt
 │    ├── ModelRunner.kt
 │    └── FakeModelRunner.kt
 │
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
fake ML layer
```

---

## 26. What you learned in Lesson 29

You created:

```text
PredictionResult
ModelRunner
FakeModelRunner
```

You learned that:

```text
ModelRunner
```

is the interface for model inference.

```text
FakeModelRunner
```

is a temporary fake model for testing the app workflow.

```text
PredictionResult
```

stores the model output:

```text
label
confidence
```

You updated the repository so it can:

```text
extract features for a session
run fake inference
save ResultEntity
return PredictionResult
```

The most important mental model is:

```text
ML inference should be part of the research data pipeline,
not random code inside the UI.
```

The correct flow is:

```text
processed measurements
 ↓
features
 ↓
ModelRunner
 ↓
PredictionResult
 ↓
ResultEntity
 ↓
Room database
```

This prepares the app for a real LiteRT/TFLite model later.

---

# Lesson 30 preview

In Lesson 30, we will start building the Compose screens for the Direction A app skeleton.
We will create the main UI screens:

```text
PatientListScreen
PatientDetailScreen
MeasurementScreen
ResultScreen
```

At first, they will use simple placeholder data and callbacks.
Then we will connect them to the ViewModel and repository in later lessons.
This will move the project from backend/data-layer structure toward an actual usable app interface.