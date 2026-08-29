# Lesson 6 — Measurement History with data class, MutableList, and LazyColumn

Now we’ll make the app feel much more like a real research data-collection tool. In Lesson 5, the app could show a latest value and a count. That’s useful, but for your kind of research app, you’ll usually need to keep the full measurement history, not just the newest reading.

By the end of this lesson, the app will follow this flow:

```text
User enters sample ID
    ↓
User clicks Start Measurement
    ↓
App generates/receives one measurement
    ↓
App saves it into a list
    ↓
App displays measurement history
    ↓
App calculates mean/min/max
```

## 1. Why we need a Measurement data class

Instead of storing only raw numbers like:

```text
2.43
2.51
2.38
```

we should store each reading together with its useful research metadata:

```text
sample ID
repetition number
measurement value
timestamp
```

So we create a data class:

```kotlin
data class Measurement(
    val sampleId: String,
    val repetition: Int,
    val value: Double,
    val timestamp: Long
)
```

This represents one recorded measurement.

For example:

```kotlin
val measurement = Measurement(
    sampleId = "S001",
    repetition = 1,
    value = 2.438,
    timestamp = System.currentTimeMillis()
)
```

This is much better than trying to manage separate variables for sample ID, value, repetition, and time.

## 2. Where to put the data class

Put it outside ResearchScreen().

For example:

```kotlin
data class Measurement(
    val sampleId: String,
    val repetition: Int,
    val value: Double,
    val timestamp: Long
)

@Composable
fun ResearchScreen() {
    // UI code here
}
```

Don’t define the data class inside the composable function. It describes your data model, so it should sit at the file level, separate from the screen UI.

## 3. Normal MutableList vs Compose state list

In normal Kotlin, you might write:

```kotlin
val measurements = mutableListOf<Measurement>()
```

Then:

```text
measurements.add(newMeasurement)
```

That’s valid Kotlin.

However, in Compose, this may not update the UI automatically, because Compose needs to know when the list changes.

So for Compose UI state, use:

```kotlin
val measurements = remember {
    mutableStateListOf<Measurement>()
}
```

This is a Compose-aware mutable list.

When you call:

```text
measurements.add(newMeasurement)
```

Compose knows the list changed and refreshes the UI.

You need this import:

```kotlin
import androidx.compose.runtime.mutableStateListOf
```

This distinction is important. A normal mutable list stores the data, but a Compose state list stores the data and tells Compose to redraw the relevant UI.

## 4. Add a measurement to the list

Suppose we already have:

```kotlin
val measurements = remember {
    mutableStateListOf<Measurement>()
}
```

When the user clicks the button, we can create a new measurement:

```kotlin
val newMeasurement = Measurement(
    sampleId = sampleId,
    repetition = measurements.size + 1,
    value = Random.nextDouble(0.0, 5.0),
    timestamp = System.currentTimeMillis()
)
```

Then add it:

```text
measurements.add(newMeasurement)
```

Full button logic:

```kotlin
Button(
    onClick = {
        val newMeasurement = Measurement(
            sampleId = sampleId,
            repetition = measurements.size + 1,
            value = Random.nextDouble(0.0, 5.0),
            timestamp = System.currentTimeMillis()
        )

        measurements.add(newMeasurement)
    }
) {
    Text("Start Measurement")
}
```

This is the key transition: the button no longer only changes a single latest value; it creates a proper measurement record and appends it to your dataset.

## 5. Get the latest measurement

If the list may be empty, use:
```kotlin
val latestMeasurement = measurements.lastOrNull()
```

Why not use this?

```kotlin
val latestMeasurement = measurements.last()
```

Because if the list is empty, last() can crash.

This is safer:

```kotlin
val latestMeasurement = measurements.lastOrNull()
```

It returns:

```text
latest Measurement object
or
null if the list is empty
```

Then you can display the latest value safely:

```kotlin
Text(
    text = latestMeasurement?.value?.let {
        "%.3f".format(it)
    } ?: "--"
)
```

This means:

If there is a latest measurement:

```text
    show its value with 3 decimal places
```

Otherwise:

```text
    show "--"
```

This is the same null-safety pattern from Lesson 3, now used in a real app screen.

## 6. Calculate statistics

To calculate statistics, first extract the measurement values:

```kotlin
val values = measurements.map {
    it.value
}
```

Then:

```kotlin
val mean = values.average()
val min = values.minOrNull()
val max = values.maxOrNull()
```

But when the list is empty, you need to be careful.

A safe display version is:

```kotlin
val values = measurements.map {
    // If measurements is empty, values is an empty list ([]), not null.
    it.value
}

// values is a non-null list, so check whether it is empty instead of using ?.let. There is no built-in method like .averageOrNull() in Kotlin. 
val meanText = if (values.isNotEmpty()) {
    "%.3f".format(values.average())
} else {
    "--"
}

// minOrNull() returns null when values is empty.
val minText = values.minOrNull()?.let {
    "%.3f".format(it)
} ?: "--"

// maxOrNull() returns null when values is empty.
val maxText = values.maxOrNull()?.let {
    "%.3f".format(it)
} ?: "--"
```

This gives you text that can be displayed directly in the UI.

## 7. Display a list with LazyColumn

If you want to display many measurements, use LazyColumn.

It is a vertical scrolling list that only renders items currently visible (unlike Column, which renders all children at once).

You need these imports:

```kotlin
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
```

Basic example:

```kotlin
LazyColumn {
    // The 'items()' function: for each measurement in the list, execute the lambda { }.
    items(measurements) { measurement ->
        // This lambda (transformation lambda) is run once per item.
        // 'measurement' is the current item being processed.
        Text(
            text = "R${measurement.repetition}: ${measurement.value}"
        )
    }
}
```

This means:

For each measurement in the list, display one row of text.

**Key idea**: LazyColumn only renders items that are visible on screen. When you scroll, it "recycles" UI components, making it efficient for large lists.

For a small list, a normal Column might appear to work. But LazyColumn is the right habit for a measurement history, because it handles longer scrolling lists properly.

## 8. Make a row for each measurement

A nicer version:

```kotlin
LazyColumn {
    // The transformation lambda: for each measurement, create a Text component
    items(measurements) { measurement ->
        Text(
            text = "R${measurement.repetition} | ${measurement.sampleId} | ${
                "%.3f".format(measurement.value)
            }"
        )
    }
}
```

Output might look like:

- R1 | S001 | 2.438
- R2 | S001 | 2.517
- R3 | S001 | 2.394

This is already close to a simple measurement log.

## 9. Full Lesson 6 app

Here is the full version.

Replace your MainActivity.kt with this, but keep your own package name if Android Studio generated a different one.

```kotlin
package com.example.researchapp

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.Button
import androidx.compose.material3.Card
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateListOf
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import kotlin.random.Random

data class Measurement(
    val sampleId: String,
    val repetition: Int,
    val value: Double,
    val timestamp: Long
)

class MainActivity : ComponentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContent {
            MaterialTheme {
                ResearchScreen()
            }
        }
    }
}

@Composable
fun ResearchScreen() {

    var sampleId by remember {
        mutableStateOf("")
    }

    var isConnected by remember {
        mutableStateOf(false)
    }

    val measurements = remember {
        mutableStateListOf<Measurement>()
    }

    val latestMeasurement = measurements.lastOrNull()

    val values = measurements.map {
        // If measurements is empty, values is an empty list ([]), not null.
        it.value
    }

    // values is a non-null list, so check whether it is empty instead of using ?.let.
    val meanText = if (values.isNotEmpty()) {
        "%.3f".format(values.average())
    } else {
        "--"
    }

    // minOrNull() returns null when values is empty.
    val minText = values.minOrNull()?.let {
        "%.3f".format(it)
    } ?: "--"

    // maxOrNull() returns null when values is empty.
    val maxText = values.maxOrNull()?.let {
        "%.3f".format(it)
    } ?: "--"

    Column(
        modifier = Modifier
            .padding(16.dp)
            .fillMaxWidth(),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        Text(
            text = "Research Measurement App",
            fontSize = 26.sp
        )

        OutlinedTextField(
            value = sampleId,
            onValueChange = {
                sampleId = it
            },
            label = {
                Text("Sample ID")
            },
            modifier = Modifier.fillMaxWidth()
        )

        if (sampleId.isBlank()) {
            Text("Please enter a sample ID before measuring.")
        }

        Text(
            text = if (isConnected) {
                "Device status: Connected"
            } else {
                "Device status: Disconnected"
            }
        )

        Button(
            onClick = {
                isConnected = !isConnected
            },
            modifier = Modifier.fillMaxWidth()
        ) {
            Text(
                text = if (isConnected) {
                    "Disconnect"
                } else {
                    "Connect"
                }
            )
        }

        Card(
            modifier = Modifier.fillMaxWidth()
        ) {
            Column(
                modifier = Modifier.padding(16.dp),
                verticalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                Text("Latest value")

                Text(
                    text = latestMeasurement?.value?.let {
                        "%.3f".format(it)
                    } ?: "--",
                    fontSize = 40.sp
                )

                Text("Measurements: ${measurements.size}")
                Text("Mean: $meanText")
                Text("Min: $minText")
                Text("Max: $maxText")
            }
        }

        Row(
            modifier = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.spacedBy(12.dp)
        ) {
            Button(
                onClick = {
                    val newMeasurement = Measurement(
                        sampleId = sampleId,
                        repetition = measurements.size + 1,
                        value = Random.nextDouble(
                            from = 0.0,
                            until = 5.0
                        ),
                        timestamp = System.currentTimeMillis()
                    )

                    measurements.add(newMeasurement)
                },
                enabled = sampleId.isNotBlank() && isConnected,
                modifier = Modifier.weight(1f)
            ) {
                Text("Measure")
            }

            Button(
                onClick = {
                    measurements.clear()
                },
                modifier = Modifier.weight(1f)
            ) {
                Text("Clear")
            }
        }

        Text(
            text = "Measurement History",
            fontSize = 20.sp
        )

        LazyColumn(
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            items(measurements) { measurement ->

                Card(
                    modifier = Modifier.fillMaxWidth()
                ) {
                    Column(
                        modifier = Modifier.padding(12.dp)
                    ) {
                        Text("Sample: ${measurement.sampleId}")
                        Text("Repetition: ${measurement.repetition}")
                        Text("Value: ${"%.3f".format(measurement.value)}")
                    }
                }
            }
        }
    }
}
```

## 10. What this app can now do

This version can:

- enter a sample ID

- connect/disconnect fake device

- enable measurement only when ready

- create a new Measurement object

- add it to a history list

- display latest value

- display measurement count

- calculate mean/min/max

- show measurement history

- clear all measurements

This is a big step. You now have the basic structure for a real data-collection screen: a label, a measurement event, a stored record, live summary statistics, and a visible history.

## 11. Important concept: UI state as data

The most important line in this lesson is:

```kotlin
val measurements = remember {
    mutableStateListOf<Measurement>()
}
```

The measurement history is now the central state.

The UI reads from it:

```kotlin
measurements.size
measurements.lastOrNull()
measurements.map { it.value }.average()
```

The button changes it:

```text
measurements.add(newMeasurement)
```

The list displays it:

LazyColumn {

```text
    items(measurements) { measurement ->
        ...
    }
}
```

So the pattern is:

```text
data changes
    ↓
Compose UI updates
```

This is one of the most important ideas in Compose. You don’t manually refresh the screen; you update the state, and Compose redraws what depends on that state.

## 12. One limitation in the current version

Right now, the repetition number is:

```text
repetition = measurements.size + 1
```

That works if all measurements are for one sample.

But if you change the sample ID from S001 to S002, the repetition count continues from the whole list.

For a more realistic research app, repetition should often be counted per sample ID.

For example:

```kotlin
val repetitionForThisSample =
    measurements.count {
        it.sampleId == sampleId
    } + 1
```

Then:

```kotlin
val newMeasurement = Measurement(
    sampleId = sampleId,
    repetition = repetitionForThisSample,
    value = Random.nextDouble(0.0, 5.0),
    timestamp = System.currentTimeMillis()
)
```

This is better if your app measures multiple samples in one session.

## 13. Better button logic using sample-specific repetition

Replace this:

```text
repetition = measurements.size + 1
```

with this:

```kotlin
val repetitionForThisSample = measurements.count {
    it.sampleId == sampleId
} + 1
```

Then create the measurement like this:

```kotlin
val newMeasurement = Measurement(
    sampleId = sampleId,
    repetition = repetitionForThisSample,
    value = Random.nextDouble(
        from = 0.0,
        until = 5.0
    ),
    timestamp = System.currentTimeMillis()
)
```

Now the measurement history might become:

```text
S001 R1
S001 R2
S002 R1
S002 R2
S001 R3
```

That is much more suitable for research data collection.

## 14. Cleaner measurement row

The row inside LazyColumn can be separated into its own composable.

Add this function outside ResearchScreen():

```kotlin
@Composable
fun MeasurementRow(
    measurement: Measurement
) {
    Card(
        modifier = Modifier.fillMaxWidth()
    ) {
        Column(
            modifier = Modifier.padding(12.dp)
        ) {
            Text("Sample: ${measurement.sampleId}")
            Text("Repetition: ${measurement.repetition}")
            Text("Value: ${"%.3f".format(measurement.value)}")
        }
    }
}
```

Then inside LazyColumn, write:

```kotlin
LazyColumn(
    verticalArrangement = Arrangement.spacedBy(8.dp)
) {
    items(measurements) { measurement ->
        MeasurementRow(measurement)
    }
}
```

This makes the code easier to read:

```text
ResearchScreen = whole screen
MeasurementRow = one measurement item
```

As the app grows, this separation will become very helpful. You can keep the whole screen readable instead of putting every UI detail into one large function.

## 15. What you need to remember from Lesson 6

The most important patterns are:

LazyColumn {

```kotlin
data class Measurement(
    val sampleId: String,
    val repetition: Int,
    val value: Double,
    val timestamp: Long
)
val measurements = remember {
    mutableStateListOf<Measurement>()
}
measurements.add(newMeasurement)
measurements.clear()
val latestMeasurement = measurements.lastOrNull()
val values = measurements.map {
    it.value
}
val mean = values.average()
    items(measurements) { measurement ->
        Text("${measurement.sampleId}: ${measurement.value}")
    }
}
```

If you understand these, you can now create and display a simple dataset inside an Android app.

## Small exercise

Modify the app so each measurement also stores a status:

```kotlin
enum class MeasurementStatus {
    LOW,
    NORMAL,
    HIGH,
}
```

Then update the data class:

```kotlin
data class Measurement(
    val sampleId: String,
    val repetition: Int,
    val value: Double,
    val timestamp: Long,
    val status: MeasurementStatus
)
```

When creating a new measurement, calculate:

```kotlin
val status = when {
    value > 4.0 -> MeasurementStatus.HIGH
    value < 1.0 -> MeasurementStatus.LOW
    else -> MeasurementStatus.NORMAL
}
```

Then display the status in each measurement row.

## Lesson 7 preview

Next, we should cover saving data to a CSV file, because that is essential for a research app:

```text
Measurement objects
    ↓
convert to CSV text
    ↓
save to Android storage
    ↓
export/share the file
```

That will make the app much more practically useful.
