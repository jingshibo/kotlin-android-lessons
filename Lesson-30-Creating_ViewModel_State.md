# Lesson 30 — Building the Compose Screens

In Lesson 29, we completed the main backend/data pipeline for the fake research app.

The app now has:

```text
Room database
Repository
Fake device source
SignalProcessor
FakeModelRunner
```

So the backend structure can already support this kind of flow:

```text
create patient
 ↓
create session
 ↓
connect fake device
 ↓
read fake value
 ↓
process value
 ↓
save measurement
 ↓
run fake inference
 ↓
save result
```

Now we move toward the user interface.

This corresponds to **Step A8** in Direction A:

```text
Build Compose screens
```

The goal of Lesson 30 is to create the main screen files:

```text
PatientListScreen.kt
PatientDetailScreen.kt
MeasurementScreen.kt
ResultScreen.kt
```

At this stage, we focus on screen structure and callbacks.

We will not fully connect the screens to the ViewModel yet. That will come in Lesson 31.

---

## 1. Where we are in the project structure

From Lesson 23, we planned this folder:

```text
ui
 ├── ResearchApp.kt
 ├── PatientListScreen.kt
 ├── PatientDetailScreen.kt
 ├── MeasurementScreen.kt
 └── ResultScreen.kt
```

In this lesson, we focus on:

```text
PatientListScreen
PatientDetailScreen
MeasurementScreen
ResultScreen
```

These screens represent the main research workflow:

```text
Patient List
 ↓
Patient Detail
 ↓
Measurement
 ↓
Result
```

---

## 2. What each screen should do

Each screen should have one main purpose.

| Screen | Main purpose |
|---|---|
| `PatientListScreen` | Show all patients/samples and allow creating/opening one |
| `PatientDetailScreen` | Show one patient and their sessions |
| `MeasurementScreen` | Show acquisition controls and latest values |
| `ResultScreen` | Show prediction/result information |

This is better than putting everything into one large screen.

A bad structure would be:

```text
One giant screen:
patient list
patient detail
session creation
device connection
measurement values
ML result
export button
```

A better structure is:

```text
PatientListScreen
 ↓
PatientDetailScreen
 ↓
MeasurementScreen
 ↓
ResultScreen
```

Each screen stays understandable.

---

## 3. Screens should receive data and callbacks

For now, each screen should be written as a reusable composable.

That means the screen receives:

```text
data to display
callbacks for user actions
```

For example:

```kotlin
@Composable
fun PatientListScreen(
    patientCodes: List<String>,
    onCreatePatientClick: () -> Unit,
    onPatientClick: (Long) -> Unit
) {
    ...
}
```

This means:

```text
The screen displays patient data.
The screen does not decide how to create a patient.
The screen does not decide how navigation works.
```

The screen only reports user actions:

```text
Create patient was clicked.
Patient 1 was clicked.
```

The ViewModel/navigation layer will decide what happens next.

---

## 4. Why callbacks are important

A beginner mistake is to make the screen do everything.

For example:

```text
PatientListScreen
 └── directly writes to Room
 └── directly navigates
 └── directly creates repository
```

Avoid that.

A cleaner pattern is:

```text
Screen
 ↓
calls callback

Parent/ViewModel
 ↓
decides what to do
```

For example:

```kotlin
Button(
    onClick = onCreatePatientClick
) {
    Text("Create Patient")
}
```

The button does not know the details.

It only says:

```text
The user clicked Create Patient.
```

This makes screens easier to reuse and easier to test.

---

## 5. Create `PatientListScreen.kt`

Create this file:

```text
app/src/main/java/com/example/researchapp/ui/PatientListScreen.kt
```

Code:

```kotlin
package com.example.researchapp.ui

import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Button
import androidx.compose.material3.Card
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun PatientListScreen(
    patientCode: String,
    onPatientCodeChange: (String) -> Unit,
    patientItems: List<PatientListItem>,
    onCreatePatientClick: () -> Unit,
    onPatientClick: (Long) -> Unit
) {
    Column(
        modifier = Modifier.padding(16.dp)
    ) {
        Text("Patient List")

        Spacer(modifier = Modifier.height(16.dp))

        OutlinedTextField(
            value = patientCode,
            onValueChange = onPatientCodeChange,
            label = {
                Text("New patient/sample code")
            },
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(modifier = Modifier.height(8.dp))

        Button(
            onClick = onCreatePatientClick,
            enabled = patientCode.isNotBlank()
        ) {
            Text("Create Patient")
        }

        Spacer(modifier = Modifier.height(24.dp))

        Text("Existing patients")

        Spacer(modifier = Modifier.height(8.dp))

        if (patientItems.isEmpty()) {
            Text("No patients yet")
        } else {
            patientItems.forEach { patient ->
                PatientListRow(
                    patient = patient,
                    onClick = {
                        onPatientClick(patient.id)
                    }
                )

                Spacer(modifier = Modifier.height(8.dp))
            }
        }
    }
}
```

This screen shows:

```text
title
patient code input
Create Patient button
existing patient list
```

---

## 6. Create `PatientListItem`

In the same file, add:

```kotlin
data class PatientListItem(
    val id: Long,
    val patientCode: String,
    val createdAt: Long
)
```

This is a small UI-friendly data class.

It represents what the patient list screen needs to display.

You may wonder:

```text
Why not use PatientEntity directly?
```

For a beginner app, using `PatientEntity` directly is also possible.

But using `PatientListItem` makes the UI less dependent on the database entity.

For now, this is a clean but still simple approach.

---

## 7. Create `PatientListRow`

Add this below the screen:

```kotlin
@Composable
fun PatientListRow(
    patient: PatientListItem,
    onClick: () -> Unit
) {
    Card(
        modifier = Modifier.fillMaxWidth(),
        onClick = onClick
    ) {
        Column(
            modifier = Modifier.padding(12.dp)
        ) {
            Text("Patient code: ${patient.patientCode}")
            Text("Patient ID: ${patient.id}")
        }
    }
}
```

This row shows one patient.

The important part is:

```kotlin
onClick = onClick
```

When the user clicks a patient card, the screen reports the click through the callback.

Later, navigation can move to:

```text
PatientDetailScreen(patientId)
```

---

## 8. Patient List screen mental model

The `PatientListScreen` does not know:

```text
how patient is inserted into Room
how patient list is loaded
which repository function is called
which screen comes next
```

It only knows:

```text
display patient input
display patient rows
report button clicks
```

That is a good UI design.

---

## 9. Create `PatientDetailScreen.kt`

Create:

```text
app/src/main/java/com/example/researchapp/ui/PatientDetailScreen.kt
```

This screen should show:

```text
selected patient information
session name input
Create Session button
list of sessions
```

Code:

```kotlin
package com.example.researchapp.ui

import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Button
import androidx.compose.material3.Card
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun PatientDetailScreen(
    patientId: Long,
    patientCode: String,
    sessionName: String,
    onSessionNameChange: (String) -> Unit,
    sessionItems: List<SessionListItem>,
    onCreateSessionClick: () -> Unit,
    onSessionClick: (Long) -> Unit,
    onBackClick: () -> Unit
) {
    Column(
        modifier = Modifier.padding(16.dp)
    ) {
        Button(
            onClick = onBackClick
        ) {
            Text("Back")
        }

        Spacer(modifier = Modifier.height(16.dp))

        Text("Patient Detail")
        Text("Patient ID: $patientId")
        Text("Patient code: $patientCode")

        Spacer(modifier = Modifier.height(24.dp))

        OutlinedTextField(
            value = sessionName,
            onValueChange = onSessionNameChange,
            label = {
                Text("New session name")
            },
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(modifier = Modifier.height(8.dp))

        Button(
            onClick = onCreateSessionClick,
            enabled = sessionName.isNotBlank()
        ) {
            Text("Create Session")
        }

        Spacer(modifier = Modifier.height(24.dp))

        Text("Sessions")

        Spacer(modifier = Modifier.height(8.dp))

        if (sessionItems.isEmpty()) {
            Text("No sessions yet")
        } else {
            sessionItems.forEach { session ->
                SessionListRow(
                    session = session,
                    onClick = {
                        onSessionClick(session.id)
                    }
                )

                Spacer(modifier = Modifier.height(8.dp))
            }
        }
    }
}
```

This screen is the bridge between patient and measurement session.

---

## 10. Create `SessionListItem`

In the same file, add:

```kotlin
data class SessionListItem(
    val id: Long,
    val sessionName: String,
    val startedAt: Long,
    val endedAt: Long?
)
```

This is a UI-friendly representation of a session.

It contains:

```text
session ID
session name
start time
end time
```

The `endedAt` field is nullable because a session may not have ended yet.

---

## 11. Create `SessionListRow`

Add:

```kotlin
@Composable
fun SessionListRow(
    session: SessionListItem,
    onClick: () -> Unit
) {
    Card(
        modifier = Modifier.fillMaxWidth(),
        onClick = onClick
    ) {
        Column(
            modifier = Modifier.padding(12.dp)
        ) {
            Text("Session: ${session.sessionName}")
            Text("Session ID: ${session.id}")

            if (session.endedAt == null) {
                Text("Status: Not ended")
            } else {
                Text("Status: Ended")
            }
        }
    }
}
```

This row lets the user open a session.

Later, clicking a session will navigate to:

```text
MeasurementScreen(sessionId)
```

---

## 12. Patient Detail screen mental model

This screen does not create sessions directly.

It receives:

```kotlin
onCreateSessionClick: () -> Unit
```

and calls it when the button is clicked.

It also receives:

```kotlin
onSessionClick: (Long) -> Unit
```

and calls it when the user selects a session.

So the screen remains UI-focused.

The repository and ViewModel will handle the actual session creation later.

---

## 13. Create `MeasurementScreen.kt`

Create:

```text
app/src/main/java/com/example/researchapp/ui/MeasurementScreen.kt
```

The Measurement Screen should show:

```text
session ID
device status
acquisition status
latest raw value
latest processed value
latest status
measurement count
buttons for connect/start/stop/inference/result
```

Code:

```kotlin
package com.example.researchapp.ui

import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.width
import androidx.compose.material3.Button
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun MeasurementScreen(
    sessionId: Long,
    deviceStatusText: String,
    acquisitionStatusText: String,
    latestRawValue: Double?,
    latestProcessedValue: Double?,
    latestStatus: String,
    measurementCount: Int,
    canConnectDevice: Boolean,
    canDisconnectDevice: Boolean,
    canStartAcquisition: Boolean,
    canStopAcquisition: Boolean,
    canRunInference: Boolean,
    onConnectDeviceClick: () -> Unit,
    onDisconnectDeviceClick: () -> Unit,
    onStartAcquisitionClick: () -> Unit,
    onStopAcquisitionClick: () -> Unit,
    onRunInferenceClick: () -> Unit,
    onViewResultClick: () -> Unit,
    onBackClick: () -> Unit
) {
    val rawText = latestRawValue?.let {
        "%.3f".format(it)
    } ?: "No data"

    val processedText = latestProcessedValue?.let {
        "%.3f".format(it)
    } ?: "No data"

    Column(
        modifier = Modifier.padding(16.dp)
    ) {
        Button(
            onClick = onBackClick
        ) {
            Text("Back")
        }

        Spacer(modifier = Modifier.height(16.dp))

        Text("Measurement Screen")
        Text("Session ID: $sessionId")

        Spacer(modifier = Modifier.height(16.dp))

        Text("Device status: $deviceStatusText")
        Text("Acquisition status: $acquisitionStatusText")

        Spacer(modifier = Modifier.height(16.dp))

        Text("Latest raw value: $rawText")
        Text("Latest processed value: $processedText")
        Text("Latest status: $latestStatus")
        Text("Measurement count: $measurementCount")

        Spacer(modifier = Modifier.height(24.dp))

        Row {
            Button(
                onClick = onConnectDeviceClick,
                enabled = canConnectDevice
            ) {
                Text("Connect")
            }

            Spacer(modifier = Modifier.width(8.dp))

            Button(
                onClick = onDisconnectDeviceClick,
                enabled = canDisconnectDevice
            ) {
                Text("Disconnect")
            }
        }

        Spacer(modifier = Modifier.height(12.dp))

        Row {
            Button(
                onClick = onStartAcquisitionClick,
                enabled = canStartAcquisition
            ) {
                Text("Start")
            }

            Spacer(modifier = Modifier.width(8.dp))

            Button(
                onClick = onStopAcquisitionClick,
                enabled = canStopAcquisition
            ) {
                Text("Stop")
            }
        }

        Spacer(modifier = Modifier.height(12.dp))

        Button(
            onClick = onRunInferenceClick,
            enabled = canRunInference
        ) {
            Text("Run Inference")
        }

        Spacer(modifier = Modifier.height(12.dp))

        Button(
            onClick = onViewResultClick
        ) {
            Text("View Result")
        }
    }
}
```

This is the most important screen for data acquisition.

---

## 14. Why pass `canStartAcquisition` instead of deciding inside screen?

The screen receives:

```kotlin
canStartAcquisition: Boolean
```

rather than calculating everything itself.

Why?

Because the decision may depend on app state:

```text
Is there an active session?
Is the device connected?
Is acquisition already recording?
Are permissions granted?
```

Those rules belong to the ViewModel.

The screen should only use the result:

```kotlin
enabled = canStartAcquisition
```

This keeps UI code simple.

---

## 15. Measurement screen mental model

The Measurement Screen is responsible for displaying acquisition controls.

It does not directly:

```text
connect to fake device
read values
insert measurements into Room
run model inference
```

It only calls callbacks:

```text
onConnectDeviceClick()
onStartAcquisitionClick()
onRunInferenceClick()
```

The ViewModel will connect those callbacks to repository functions later.

---

## 16. Create `ResultScreen.kt`

Create:

```text
app/src/main/java/com/example/researchapp/ui/ResultScreen.kt
```

The Result Screen should show:

```text
session ID
prediction label
confidence
measurement count
export button later
```

Code:

```kotlin
package com.example.researchapp.ui

import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Button
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun ResultScreen(
    sessionId: Long,
    predictionLabel: String?,
    predictionConfidence: Double?,
    measurementCount: Int,
    message: String,
    onExportClick: () -> Unit,
    onBackClick: () -> Unit
) {
    val labelText = predictionLabel ?: "No prediction"

    val confidenceText = predictionConfidence?.let {
        "%.2f".format(it)
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

        Text("Result Screen")
        Text("Session ID: $sessionId")

        Spacer(modifier = Modifier.height(16.dp))

        Text("Prediction: $labelText")
        Text("Confidence: $confidenceText")
        Text("Measurement count: $measurementCount")

        Spacer(modifier = Modifier.height(24.dp))

        Button(
            onClick = onExportClick
        ) {
            Text("Export Session CSV")
        }

        Spacer(modifier = Modifier.height(16.dp))

        if (message.isNotBlank()) {
            Text(message)
        }
    }
}
```

This screen is simple for now.

Export will be connected later.

---

## 17. Result screen mental model

The Result Screen should not run the model itself.

Avoid this:

```text
ResultScreen
 └── load measurements
 └── extract features
 └── run model
 └── save result
```

That should happen through:

```text
ViewModel
 ↓
Repository
 ↓
ModelRunner
```

The Result Screen only displays:

```text
prediction label
confidence
measurement count
message
```

---

## 18. What about `LazyColumn`?

For patient and session lists, we used:

```kotlin
patientItems.forEach { ... }
```

This is okay for small beginner examples.

But if the list becomes long, use:

```kotlin
LazyColumn
```

For example:

```kotlin
LazyColumn {
    items(patientItems) { patient ->
        PatientListRow(
            patient = patient,
            onClick = {
                onPatientClick(patient.id)
            }
        )
    }
}
```

For this lesson, I used `forEach` to keep the code simple.

Later, if the patient/session list grows, switch to `LazyColumn`.

---

## 19. Should UI display raw timestamps?

Right now, some UI models contain:

```text
createdAt: Long
startedAt: Long
endedAt: Long?
```

But in the screen, we did not format them nicely yet.

That is intentional.

Proper date/time formatting can be added later.

For now, focus on:

```text
screen structure
data flow
callbacks
```

A future improvement can show:

```text
Started: 2026-08-26 14:30
Ended: 2026-08-26 14:35
```

But that is not necessary for Lesson 30.

---

## 20. Create simple preview data later

In Android Studio, you may later add `@Preview` functions to see screens without running the full app.

For example:

```kotlin
@Preview(showBackground = true)
@Composable
fun PatientListScreenPreview() {
    PatientListScreen(
        patientCode = "P001",
        onPatientCodeChange = {},
        patientItems = listOf(
            PatientListItem(
                id = 1,
                patientCode = "P001",
                createdAt = 0L
            )
        ),
        onCreatePatientClick = {},
        onPatientClick = {}
    )
}
```

This is useful for UI design.

But we do not need to add previews to every screen yet.

The main lesson is screen structure.

---

## 21. Why screens do not create ViewModel directly yet

You may expect to see:

```kotlin
val viewModel: ResearchViewModel = viewModel()
```

inside the screens.

But in this lesson, I am avoiding that.

Why?

Because first we want to write screens as reusable UI components.

The screen receives:

```text
state values
callback functions
```

Then later, a parent composable can connect those values to the real ViewModel.

This pattern is cleaner:

```text
Screen composable
 ↓
stateless / mostly stateless UI

Parent composable
 ↓
gets ViewModel state
passes state and callbacks into screen
```

This makes the app easier to maintain.

---

## 22. Current UI files after Lesson 30

After this lesson, your `ui` folder should contain:

```text
ui
 ├── ResearchApp.kt
 ├── PatientListScreen.kt
 ├── PatientDetailScreen.kt
 ├── MeasurementScreen.kt
 └── ResultScreen.kt
```

The screens are not fully connected yet, but their shape is ready.

The app now has:

```text
data layer
fake device layer
processing layer
fake ML layer
screen layer
```

The missing bridge is:

```text
ViewModel + navigation connection
```

That comes next.

---

## 23. Current architecture after Lesson 30

The architecture now looks like this:

```text
UI screens
 ├── PatientListScreen
 ├── PatientDetailScreen
 ├── MeasurementScreen
 └── ResultScreen

ViewModel layer
 └── not fully connected yet

Repository layer
 └── MeasurementRepository

Backend layers
 ├── Room database
 ├── FakeDeviceDataSource
 ├── SignalProcessor
 └── FakeModelRunner
```

The screens currently define:

```text
what the user sees
what actions the user can take
```

The repository currently defines:

```text
what the app can do with data
```

Lesson 31 will connect them.

---

## 24. Common mistake: screens doing too much

Screens should stay simple.

Avoid putting this inside screens:

```text
Room queries
repository creation
device connection loops
model inference
CSV export construction
```

The screen should mostly contain:

```text
Text
Button
OutlinedTextField
Card
Column
Row
callbacks
```

This is the correct beginner-friendly Compose structure.

---

## 25. Common mistake: no callbacks

A screen without callbacks may look nice, but it cannot interact with the app.

For example:

```kotlin
Button(
    onClick = {}
) {
    Text("Create Patient")
}
```

This button does nothing.

A better button is:

```kotlin
Button(
    onClick = onCreatePatientClick
) {
    Text("Create Patient")
}
```

Now the parent composable can decide what happens.

So each important user action should have a callback.

---

## 26. What you learned in Lesson 30

You created the main Compose screen files:

```text
PatientListScreen
PatientDetailScreen
MeasurementScreen
ResultScreen
```

You learned that screens should receive:

```text
data to display
callbacks for user actions
```

You learned that screens should not directly contain:

```text
database operations
device communication
ML inference
export construction
```

The most important mental model is:

```text
Screens show state and report user actions.
They should not own the whole app logic.
```

The UI workflow is now:

```text
PatientListScreen
 ↓
PatientDetailScreen
 ↓
MeasurementScreen
 ↓
ResultScreen
```

This matches the research workflow:

```text
choose patient
 ↓
choose/create session
 ↓
collect measurements
 ↓
view result
```

---

## 27. Lesson 31 preview

In Lesson 31, we will connect the screens to navigation and ViewModel state.

We will work on:

```text
ResearchUiState
ResearchViewModel
ResearchApp navigation
passing callbacks into screens
loading patients
creating patients and sessions
moving between screens
```

This will start turning the separate screen files into a connected fake-data research app.