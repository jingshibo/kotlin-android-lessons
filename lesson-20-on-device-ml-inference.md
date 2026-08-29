# Lesson 20 — On-Device ML Inference

In Lesson 19, we introduced a simple signal-processing pipeline:

```text
DeviceDataSource
 ↓
raw value
 ↓
SignalProcessor
 ↓
processed value / features
 ↓
MeasurementEntity
 ↓
Room database
```

Now we add the next major research-app function:

```text
on-device ML inference
```

This means the Android tablet can run a machine-learning model locally, without sending the data to a remote server.

For our research app, the flow becomes:

```text
collect data
 ↓
process data
 ↓
extract features
 ↓
run ML model on Android tablet
 ↓
produce prediction/result
 ↓
save result to Room
 ↓
show result on screen
```

Android’s current AI/ML guidance lists **LiteRT**, formerly TensorFlow Lite, as the option for running custom on-device ML models on Android. citeturn324983search1turn324983search5

---

## 1. What on-device inference means

Training and inference are different.

```text
Training
 ↓
learn model parameters from data

Inference
 ↓
use an already-trained model to make a prediction
```

For your app, the usual workflow is:

```text
Python / PyTorch / TensorFlow
 ↓
train model on computer
 ↓
convert model to .tflite / LiteRT-compatible model
 ↓
put model inside Android app
 ↓
run inference on tablet
```

So the Android app is usually **not training the model**.

The Android app is using the trained model.

For example:

```text
processed sensor features
 ↓
ML model
 ↓
prediction label: Positive
 ↓
confidence: 0.92
```

---

## 2. Why this matters for a research app

A research data-collection app can have different levels.

### Level 1: Data collection only

```text
collect data
 ↓
save CSV
 ↓
analyse later in Python
```

### Level 2: Data collection + processing

```text
collect data
 ↓
process data
 ↓
save raw and processed values
```

### Level 3: Edge-AI research app

```text
collect data
 ↓
process data
 ↓
run ML model on tablet
 ↓
show result immediately
 ↓
save result with session
```

Lesson 20 moves us toward Level 3.

---

## 3. Where ML inference belongs in the architecture

Do **not** put model inference directly inside the UI.

Avoid this:

```text
ResultScreen
 └── load model
 └── prepare tensor
 └── run inference
```

A better structure is:

```text
ResultScreen
 ↓
ResearchViewModel
 ↓
MeasurementRepository
 ↓
SignalProcessor
 ↓
ModelRunner
 ↓
LiteRT model
```

Or more clearly:

```text
UI
 ↓
ViewModel
 ↓
Repository
 ↓
ML inference class
```

The UI should only display:

```text
prediction label
confidence
status message
```

The ML class should handle:

```text
model loading
input preparation
running inference
reading output
```

---

## 4. First create a prediction result class

Create a simple data class:

```kotlin
data class PredictionResult(
    val label: String,
    val confidence: Double
)
```

This represents the model output.

For example:

```kotlin
PredictionResult(
    label = "Positive",
    confidence = 0.92
)
```

or:

```kotlin
PredictionResult(
    label = "Class A",
    confidence = 0.87
)
```

The name of the labels depends on your model.

For a disease-detection app, labels might be:

```text
Negative
Positive
Uncertain
```

For a material/sample-classification app, labels might be:

```text
Class A
Class B
Class C
```

---

## 5. Create a `ModelRunner` interface

Just like we created `DeviceDataSource`, we can create a model interface.

Create:

```text
ModelRunner.kt
```

Add:

```kotlin
interface ModelRunner {
    suspend fun runInference(
        features: SignalFeatures
    ): PredictionResult
}
```

This means:

```text
Give the model some features.
The model returns a prediction result.
```

Why use an interface?

Because at first we can use a fake model.

Later we can replace it with a real LiteRT model.

```text
FakeModelRunner
 ↓
use while developing app flow

LiteRtModelRunner
 ↓
use later with real .tflite model
```

This follows the same design idea as Lesson 18.

---

## 6. Start with a fake model runner

Before connecting a real `.tflite` model, it is useful to test the app flow with fake inference.

Create:

```text
FakeModelRunner.kt
```

Add:

```kotlin
class FakeModelRunner : ModelRunner {

    override suspend fun runInference(
        features: SignalFeatures
    ): PredictionResult {
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

This is not a real ML model.

It is only a placeholder.

But it lets us test the full app path:

```text
measurements
 ↓
features
 ↓
prediction
 ↓
save result
 ↓
show result screen
```

This is the same strategy we used earlier:

```text
fake data first
real data later
```

---

## 7. Add model runner to repository

In Lesson 19, the repository used:

```kotlin
DeviceDataSource
SignalProcessor
Room database
```

Now add:

```kotlin
ModelRunner
```

The constructor becomes:

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

Now the repository can:

```text
read data
process data
extract features
run inference
save result
```

---

## 8. Use recent measurements as model input

A model usually should not run on one single value unless it was trained that way.

Commonly, it uses a window of recent measurements.

For a beginner version, we can use the last 10 processed values:

```kotlin
val recentValues = measurements
    .map { it.processedValue }
    .takeLast(10)
```

Then extract features:

```kotlin
val features = signalProcessor.extractFeatures(
    values = recentValues
)
```

If `features` is null, we cannot run inference yet.

```kotlin
if (features == null) {
    // not enough data
}
```

This means:

```text
No values
 ↓
no features
 ↓
no prediction
```

---

## 9. Add inference function to repository

Inside `MeasurementRepository`, add:

```kotlin
suspend fun runInferenceForSession(
    sessionId: Long
): PredictionResult? {
    val measurements = measurementDao.getMeasurementsForSession(
        sessionId = sessionId
    )

    val recentProcessedValues = measurements
        .map { it.processedValue }
        .takeLast(10)

    val features = signalProcessor.extractFeatures(
        values = recentProcessedValues
    ) ?: return null

    return modelRunner.runInference(
        features = features
    )
}
```

The flow is:

```text
get measurements for session
 ↓
take recent processed values
 ↓
extract features
 ↓
run model
 ↓
return prediction
```

This is the first complete ML inference path.

---

## 10. Save inference result to Room

In Lesson 15, we created a `ResultEntity`:

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

Now we can save the model result.

Add this repository function:

```kotlin
suspend fun runAndSaveInferenceForSession(
    sessionId: Long
): PredictionResult? {
    val prediction = runInferenceForSession(
        sessionId = sessionId
    ) ?: return null

    val resultEntity = ResultEntity(
        sessionId = sessionId,
        label = prediction.label,
        confidence = prediction.confidence,
        createdAt = System.currentTimeMillis()
    )

    resultDao.insertResult(resultEntity)

    return prediction
}
```

Now the result is not only displayed.

It is also stored in the database.

---

## 11. Add result state to `ResearchUiState`

The UI needs to show the latest prediction.

Add:

```kotlin
val latestPrediction: PredictionResult? = null
```

So part of `ResearchUiState` becomes:

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

    val latestPrediction: PredictionResult? = null,

    val isLoading: Boolean = false,
    val message: String = ""
)
```

Again, nullable is appropriate:

```text
PredictionResult?
 ↓
there may be a prediction,
or there may be no prediction yet
```

---

## 12. Add ViewModel inference function

Inside `ResearchViewModel`, add:

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
        uiState = uiState.copy(
            isLoading = true,
            message = "Running inference..."
        )

        try {
            val prediction =
                measurementRepository.runAndSaveInferenceForSession(
                    sessionId = sessionId
                )

            if (prediction == null) {
                uiState = uiState.copy(
                    isLoading = false,
                    message = "Not enough data for inference"
                )
            } else {
                uiState = uiState.copy(
                    latestPrediction = prediction,
                    isLoading = false,
                    message = "Inference complete"
                )
            }
        } catch (e: Exception) {
            uiState = uiState.copy(
                isLoading = false,
                message = "Inference failed"
            )
        }
    }
}
```

This function follows a familiar pattern:

```text
check session exists
 ↓
launch coroutine
 ↓
ask repository to run inference
 ↓
save result
 ↓
update UI state
```

---

## 13. Add inference button to Measurement Screen

In the `MeasurementScreen`, add:

```kotlin
Button(
    onClick = {
        viewModel.runInferenceForCurrentSession()
    },
    enabled =
        uiState.currentSessionId != null &&
        uiState.measurements.isNotEmpty() &&
        uiState.acquisitionState != AcquisitionState.RECORDING
) {
    Text("Run Inference")
}
```

Why disable it during recording?

Because for the beginner version, it is simpler and safer:

```text
collect data first
stop acquisition
run inference
show result
```

Later, we may support live inference during acquisition.

But not yet.

---

## 14. Display prediction in the UI

Create display text:

```kotlin
val predictionLabel =
    uiState.latestPrediction?.label ?: "No prediction"

val predictionConfidence =
    uiState.latestPrediction?.let {
        "%.2f".format(it.confidence)
    } ?: "--"
```

Then display:

```kotlin
Text("Prediction: $predictionLabel")
Text("Confidence: $predictionConfidence")
```

Example:

```text
Prediction: Positive
Confidence: 0.90
```

This gives immediate feedback to the user.

---

## 15. Result Screen after Lesson 20

The `ResultScreen` can show:

```text
Session ID
Prediction label
Confidence
Number of measurements
Timestamp
Export button later
```

A simple version:

```kotlin
@Composable
fun ResultScreen(
    uiState: ResearchUiState,
    onBackClick: () -> Unit
) {
    val predictionLabel =
        uiState.latestPrediction?.label ?: "No prediction"

    val predictionConfidence =
        uiState.latestPrediction?.let {
            "%.2f".format(it.confidence)
        } ?: "--"

    Column(
        modifier = Modifier.padding(16.dp)
    ) {
        Button(
            onClick = onBackClick
        ) {
            Text("Back")
        }

        Spacer(modifier = Modifier.height(16.dp))

        Text("Result")

        Text("Prediction: $predictionLabel")
        Text("Confidence: $predictionConfidence")

        Spacer(modifier = Modifier.height(16.dp))

        Text("Measurements: ${uiState.measurements.size}")

        if (uiState.message.isNotEmpty()) {
            Text(uiState.message)
        }
    }
}
```

Later, the Result Screen should load saved results from Room by `sessionId`.

For now, we keep it simple and use the latest UI state.

---

## 16. What a real LiteRT runner would do

So far, `FakeModelRunner` is not a real model.

A real model runner would do roughly this:

```text
load .tflite model file
 ↓
create interpreter/runtime
 ↓
convert features to FloatArray
 ↓
run inference
 ↓
read model output
 ↓
convert output to label/confidence
```

The TensorFlow Lite/LiteRT Android documentation explains that running inference requires a runtime environment, a model, and input data; it also notes that Android inference can use Java/Kotlin APIs through the TensorFlow Lite interpreter. citeturn324983search0turn324983search4

So the real app will need three things:

```text
model file
model input preparation
model output interpretation
```

---

## 17. Where to put the model file

For a beginner Android app, the model file is often placed in:

```text
app/src/main/assets/model.tflite
```

For example:

```text
app
 └── src
      └── main
           └── assets
                └── disease_model.tflite
```

If the `assets` folder does not exist, you can create it.

The idea is:

```text
The model is packaged inside the Android app.
The app loads it when needed.
```

Android Studio also has ML Model Binding support for importing `.tflite` files and generating model interface classes, which can make integration easier for some projects. citeturn324983search14

For this tutorial, I will explain the general manual structure first, because it helps you understand what is happening.

---

## 18. Convert features to model input

Suppose our model expects four features:

```text
mean
minimum
maximum
range
```

From Lesson 19, we had:

```kotlin
data class SignalFeatures(
    val mean: Double,
    val minimum: Double,
    val maximum: Double,
    val range: Double
)
```

Most mobile ML models expect `Float`, not `Double`.

So we convert:

```kotlin
fun featuresToFloatArray(
    features: SignalFeatures
): FloatArray {
    return floatArrayOf(
        features.mean.toFloat(),
        features.minimum.toFloat(),
        features.maximum.toFloat(),
        features.range.toFloat()
    )
}
```

This is extremely important.

The feature order must match the order used during model training.

If Python training used:

```text
[mean, minimum, maximum, range]
```

then Android must use the same order:

```kotlin
floatArrayOf(
    mean,
    minimum,
    maximum,
    range
)
```

If the order is wrong, the model output may be meaningless.

---

## 19. Normalisation must match training

This is one of the most important practical points.

If your Python model was trained using standardisation:

```text
x_scaled = (x - mean) / standard_deviation
```

then Android must apply the same transformation.

For example:

```kotlin
fun normalizeFeature(
    value: Float,
    mean: Float,
    std: Float
): Float {
    return (value - mean) / std
}
```

If training used:

```text
StandardScaler
```

then you need to export the scaler parameters:

```text
mean values
standard deviation values
```

and use them in Android.

A common mistake is:

```text
train model with normalised features
 ↓
run Android inference with raw unnormalised features
 ↓
bad predictions
```

So the Android app must reproduce the same preprocessing used during training.

---

## 20. Example input preparation

Suppose the training feature means were:

```kotlin
val featureMeans = floatArrayOf(
    2.5f,
    1.0f,
    4.0f,
    3.0f
)
```

and standard deviations:

```kotlin
val featureStds = floatArrayOf(
    0.5f,
    0.3f,
    0.8f,
    0.7f
)
```

Then:

```kotlin
fun prepareModelInput(
    features: SignalFeatures
): Array<FloatArray> {
    val rawInput = floatArrayOf(
        features.mean.toFloat(),
        features.minimum.toFloat(),
        features.maximum.toFloat(),
        features.range.toFloat()
    )

    val normalizedInput = FloatArray(rawInput.size)

    for (i in rawInput.indices) {
        normalizedInput[i] =
            (rawInput[i] - featureMeans[i]) / featureStds[i]
    }

    return arrayOf(normalizedInput)
}
```

Why `Array<FloatArray>`?

Because many simple models expect input shape like:

```text
[1, number_of_features]
```

For example:

```text
[1, 4]
```

So:

```kotlin
arrayOf(normalizedInput)
```

represents one sample with four features.

---

## 21. Example real model runner structure

A real LiteRT/TFLite-style runner might conceptually look like this:

```kotlin
class LiteRtModelRunner(
    private val context: Context
) : ModelRunner {

    override suspend fun runInference(
        features: SignalFeatures
    ): PredictionResult {
        val input = prepareModelInput(features)

        val output = Array(1) {
            FloatArray(2)
        }

        // Pseudocode:
        // interpreter.run(input, output)

        val probabilities = output[0]

        val predictedIndex =
            probabilities.indices.maxBy {
                probabilities[it]
            }

        val labels = listOf(
            "Negative",
            "Positive"
        )

        return PredictionResult(
            label = labels[predictedIndex],
            confidence = probabilities[predictedIndex].toDouble()
        )
    }
}
```

Notice I wrote:

```kotlin
// Pseudocode:
// interpreter.run(input, output)
```

because the exact runtime setup depends on which LiteRT/TensorFlow Lite integration route you use.

The main learning point is:

```text
features
 ↓
FloatArray input
 ↓
model inference
 ↓
probabilities
 ↓
label + confidence
```

---

## 22. Understanding output probabilities

Suppose the model output is:

```kotlin
val probabilities = floatArrayOf(
    0.12f,
    0.88f
)
```

and labels are:

```kotlin
val labels = listOf(
    "Negative",
    "Positive"
)
```

Then:

```text
Negative probability = 0.12
Positive probability = 0.88
```

The predicted class is:

```text
Positive
```

with confidence:

```text
0.88
```

In Kotlin:

```kotlin
val predictedIndex =
    probabilities.indices.maxBy {
        probabilities[it]
    }

val predictedLabel = labels[predictedIndex]
val confidence = probabilities[predictedIndex]
```

This is a common pattern for classification models.

---

## 23. Binary vs Multi-Class Output

Different models may produce different output shapes.

### Binary Output With One Value

Some models output:

```text
[0.87]
```

This may mean:

> Probability of positive class = 0.87

Then the logic is:

```kotlin
val positiveProbability = output[0][0]

val label = if (positiveProbability >= 0.5f) {
    "Positive"
} else {
    "Negative"
}
```

### Multi-Class Output

Other models output:

```text
[0.10, 0.20, 0.70]
```

This means three class probabilities.
Then choose the largest value:

```kotlin
val predictedIndex =
    probabilities.indices.maxBy {
        probabilities[it]
    }
```

The important point is:

> Android output interpretation must match your trained model output.

## 24. Save Result With Session Context

Do not save only:

```text
Positive
```

Save it with context:

- sessionId
- label
- confidence
- timestamp

That is why ResultEntity has:

```kotlin
val sessionId: Long
val label: String
val confidence: Double
val createdAt: Long
```

For a research app, this is essential because later you need to know:

- which session produced this result
- when the prediction was made
- what confidence was reported
- which measurements were used

Later, you may also add:

- model version
- processing version
- feature window size
- threshold

But for now, the simple result entity is enough.

## 25. Important Research Warning: Model Version

In real research, the model can change.
For example:

```text
model_v1.tflite
model_v2.tflite
model_v3_quantized.tflite
```

If you save predictions but not the model version, later you may not know which model produced which result.
So eventually, ResultEntity should include:

```kotlin
val modelVersion: String
```

For now, we can keep the entity simple.
But the habit is important:

> Prediction result should be linked to the model version.

## 26. Current App Flow After Lesson 20

The research-app flow now becomes:

```text
Patient List
 ↓
Patient Detail
 ↓
Create / select Session
 ↓
Measurement Screen
 ↓
Connect Device
 ↓
Start Acquisition
 ↓
Read data from DeviceDataSource
 ↓
Process data with SignalProcessor
 ↓
Save MeasurementEntity to Room
 ↓
Stop Acquisition
 ↓
Run Inference
 ↓
Save ResultEntity to Room
 ↓
Show Result Screen
```

This is now an edge-AI research app structure.

## 27. Architecture After Lesson 20

The app now has these main parts:

```text
Presentation layer
 ├── PatientListScreen
 ├── PatientDetailScreen
 ├── MeasurementScreen
 └── ResultScreen

ViewModel layer
 └── ResearchViewModel

Repository layer
 └── MeasurementRepository

Data-source layer
 └── DeviceDataSource

Processing layer
 └── SignalProcessor

ML layer
 └── ModelRunner

Storage layer
 └── Room database
```

More directly:

```text
DeviceDataSource
 ↓
raw value

SignalProcessor
 ↓
processed value / features

ModelRunner
 ↓
prediction

Room
 ↓
measurements and results

UI
 ↓
shows status, values, prediction
```
## 28. What You Learned in Lesson 20

The key new interface is:

```kotlin
interface ModelRunner {
    suspend fun runInference(
        features: SignalFeatures
    ): PredictionResult
}
```

The fake implementation is:

```kotlin
class FakeModelRunner : ModelRunner {
    ...
}
```

The repository can now do:

```kotlin
runInferenceForSession(sessionId)
```

and:

```kotlin
runAndSaveInferenceForSession(sessionId)
```

The ViewModel can expose:

```kotlin
fun runInferenceForCurrentSession()
```

The UI can show:

```text
Prediction: Positive
Confidence: 0.90
```

The most important mental model is:

```text
processed measurements
 ↓
features
 ↓
model input
 ↓
prediction
 ↓
saved result
```

For a research app, ML inference should not be treated as a random button action. It should be part of a controlled data pipeline:

```text
session
 ↓
measurements
 ↓
processing
 ↓
features
 ↓
model
 ↓
result
```

## Lesson 21 Preview

In Lesson 21, we should improve export.
Earlier, the app exported simple measurement CSV files.
But now the app has richer research data:

- Patient
- Session
- Measurement
- Result

So Lesson 21 should cover:

- exporting complete research data
- exporting measurements with session metadata
- exporting results
- CSV vs JSON
- file naming rules
- making exported data easy to analyse in Python
That will help connect the Android app back to your research workflow outside the tablet.