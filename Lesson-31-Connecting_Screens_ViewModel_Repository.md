# Lesson 31 — Connecting Navigation and ViewModel

In Lesson 30, we created the main Compose screens:

```text
PatientListScreen
PatientDetailScreen
MeasurementScreen
ResultScreen
```

But they were still mostly separate UI components.

They received data and callbacks, but they were not fully connected to:

```text
ResearchViewModel
MeasurementRepository
Room database
Navigation
```

Now we connect them.

This corresponds to the next part of **Direction A**:

```text
Connect screens to navigation and ViewModel state
```

After this lesson, the app structure becomes:

```text
ResearchApp
 ↓
NavHost
 ↓
Screens
 ↓
ResearchViewModel
 ↓
MeasurementRepository
 ↓
Room / FakeDevice / SignalProcessor / FakeModelRunner
```

---

## 1. What Lesson 31 is trying to solve

Right now, our screens can show UI, but they do not know where their data comes from.

For example, `PatientListScreen` expects:

```kotlin
patientItems: List<PatientListItem>
```

but we have not yet loaded real patients from Room.

`MeasurementScreen` expects:

```kotlin
latestRawValue: Double?
latestProcessedValue: Double?
measurementCount: Int
```

but we have not yet connected it to actual acquisition logic.

So Lesson 31 connects:

```text
UI state
 ↓
screens

user actions
 ↓
ViewModel functions

ViewModel functions
 ↓
Repository
```

---

## 2. The main idea

The screen should not directly talk to the repository.

Avoid this:

```text
PatientListScreen
 ↓
MeasurementRepository
```

Use this:

```text
PatientListScreen
 ↓
ResearchViewModel
 ↓
MeasurementRepository
```

So the UI flow becomes:

```text
User clicks button
 ↓
Screen calls callback
 ↓
ResearchApp passes callback to ViewModel
 ↓
ViewModel calls Repository
 ↓
Repository updates database or data source
 ↓
ViewModel updates uiState
 ↓
Screen updates automatically
```

This is the Compose + ViewModel pattern we are using.

---

## 3. Add state enums

Create or update a file such as:

```text
viewmodel/AppStateEnums.kt
```

Add:

```kotlin
package com.example.researchapp.viewmodel

enum class DeviceConnectionState {
    DISCONNECTED,
    CONNECTING,
    CONNECTED,
    ERROR
}

enum class AcquisitionState {
    IDLE,
    RECORDING,
    STOPPED,
    ERROR
}
```

These states help the app decide:

```text
Can the user connect?
Can the user start acquisition?
Can the user stop acquisition?
Can the user run inference?
```

This is better than using many separate Boolean variables.

---

## 4. Update `ResearchUiState`

Create or update:

```text
viewmodel/ResearchUiState.kt
```

Code:

```kotlin
package com.example.researchapp.viewmodel

import com.example.researchapp.data.entity.MeasurementEntity
import com.example.researchapp.ml.PredictionResult
import com.example.researchapp.ui.PatientListItem
import com.example.researchapp.ui.SessionListItem

data class ResearchUiState(
    val patientCode: String = "",
    val sessionName: String = "",

    val currentPatientId: Long? = null,
    val currentPatientCode: String = "",
    val currentSessionId: Long? = null,

    val patientItems: List<PatientListItem> = emptyList(),
    val sessionItems: List<SessionListItem> = emptyList(),

    val deviceConnectionState: DeviceConnectionState = DeviceConnectionState.DISCONNECTED,
    val acquisitionState: AcquisitionState = AcquisitionState.IDLE,

    val measurements: List<MeasurementEntity> = emptyList(),

    val latestRawValue: Double? = null,
    val latestProcessedValue: Double? = null,
    val latestStatus: String = "",

    val latestPrediction: PredictionResult? = null,

    val isLoading: Boolean = false,
    val message: String = ""
)
```

This state now contains the main data needed by the screens.

For example:

```text
PatientListScreen uses:
patientCode
patientItems

PatientDetailScreen uses:
currentPatientId
currentPatientCode
sessionName
sessionItems

MeasurementScreen uses:
deviceConnectionState
acquisitionState
latestRawValue
latestProcessedValue
measurements

ResultScreen uses:
latestPrediction
measurements
message
```

---

## 5. A note about UI item classes

In Lesson 30, we created:

```kotlin
data class PatientListItem(...)
data class SessionListItem(...)
```

inside UI files.

For a small beginner project, this is acceptable.

But if you want a cleaner structure, you can move them to:

```text
ui/model/PatientListItem.kt
ui/model/SessionListItem.kt
```

or:

```text
viewmodel/model
```

For now, we keep them as they are to avoid too much restructuring.

---

## 6. Create `ResearchViewModel`

Create or update:

```text
viewmodel/ResearchViewModel.kt
```

Because the repository needs `Context`, we can use `AndroidViewModel` for this beginner version.

```kotlin
package com.example.researchapp.viewmodel

import android.app.Application
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.setValue
import androidx.lifecycle.AndroidViewModel
import androidx.lifecycle.viewModelScope
import com.example.researchapp.data.MeasurementRepository
import com.example.researchapp.ui.PatientListItem
import com.example.researchapp.ui.SessionListItem
import kotlinx.coroutines.launch

class ResearchViewModel(
    application: Application
) : AndroidViewModel(application) {

    private val measurementRepository =
        MeasurementRepository(application)

    var uiState by mutableStateOf(
        ResearchUiState()
    )
        private set
}
```

This gives the ViewModel access to:

```text
MeasurementRepository
```

and gives the UI access to:

```text
uiState
```

---

## 7. Add patient input functions

Inside `ResearchViewModel`, add:

```kotlin
fun updatePatientCode(
    newPatientCode: String
) {
    uiState = uiState.copy(
        patientCode = newPatientCode
    )
}

fun updateSessionName(
    newSessionName: String
) {
    uiState = uiState.copy(
        sessionName = newSessionName
    )
}
```

These functions are called when the user types into text fields.

For example, `PatientListScreen` will call:

```kotlin
onPatientCodeChange = viewModel::updatePatientCode
```

This is the connection:

```text
TextField changes
 ↓
ViewModel updates state
 ↓
UI recomposes
```

---

## 8. Load patients from Room

Add this function:

```kotlin
fun loadPatients() {
    viewModelScope.launch {
        uiState = uiState.copy(
            isLoading = true,
            message = "Loading patients..."
        )

        try {
            val patients =
                measurementRepository.getAllPatients()

            val patientItems = patients.map { patient ->
                PatientListItem(
                    id = patient.id,
                    patientCode = patient.patientCode,
                    createdAt = patient.createdAt
                )
            }

            uiState = uiState.copy(
                patientItems = patientItems,
                isLoading = false,
                message = ""
            )
        } catch (e: Exception) {
            uiState = uiState.copy(
                isLoading = false,
                message = "Could not load patients"
            )
        }
    }
}
```

This function does:

```text
ask repository for patients
 ↓
convert PatientEntity to PatientListItem
 ↓
update uiState
```

The screen does not load patients itself.

---

## 9. Create patient

Add:

```kotlin
fun createPatient() {
    val code = uiState.patientCode.trim()

    if (code.isBlank()) {
        uiState = uiState.copy(
            message = "Enter a patient or sample code"
        )
        return
    }

    viewModelScope.launch {
        try {
            measurementRepository.createPatient(
                patientCode = code
            )

            uiState = uiState.copy(
                patientCode = "",
                message = "Patient created"
            )

            loadPatients()
        } catch (e: Exception) {
            uiState = uiState.copy(
                message = "Could not create patient"
            )
        }
    }
}
```

The flow is:

```text
user enters code
 ↓
clicks Create Patient
 ↓
ViewModel calls repository
 ↓
Room inserts patient
 ↓
ViewModel reloads patient list
```

This connects the patient list screen to the database.

---

## 10. Load one patient and sessions

When the user opens a patient, the app needs to load:

```text
patient information
sessions for that patient
```

Add:

```kotlin
fun loadPatientDetail(
    patientId: Long
) {
    viewModelScope.launch {
        uiState = uiState.copy(
            isLoading = true,
            message = "Loading patient..."
        )

        try {
            val patient =
                measurementRepository.getPatientById(
                    patientId = patientId
                )

            if (patient == null) {
                uiState = uiState.copy(
                    isLoading = false,
                    message = "Patient not found"
                )
                return@launch
            }

            val sessions =
                measurementRepository.getSessionsForPatient(
                    patientId = patientId
                )

            val sessionItems = sessions.map { session ->
                SessionListItem(
                    id = session.id,
                    sessionName = session.sessionName,
                    startedAt = session.startedAt,
                    endedAt = session.endedAt
                )
            }

            uiState = uiState.copy(
                currentPatientId = patient.id,
                currentPatientCode = patient.patientCode,
                sessionItems = sessionItems,
                isLoading = false,
                message = ""
            )
        } catch (e: Exception) {
            uiState = uiState.copy(
                isLoading = false,
                message = "Could not load patient detail"
            )
        }
    }
}
```

This function prepares the data for `PatientDetailScreen`.

---

## 11. Create session for current patient

Add:

```kotlin
fun createSessionForCurrentPatient() {
    val patientId = uiState.currentPatientId
    val name = uiState.sessionName.trim()

    if (patientId == null) {
        uiState = uiState.copy(
            message = "No patient selected"
        )
        return
    }

    if (name.isBlank()) {
        uiState = uiState.copy(
            message = "Enter a session name"
        )
        return
    }

    viewModelScope.launch {
        try {
            val sessionId =
                measurementRepository.createSession(
                    patientId = patientId,
                    sessionName = name
                )

            uiState = uiState.copy(
                currentSessionId = sessionId,
                sessionName = "",
                message = "Session created"
            )

            loadPatientDetail(patientId)
        } catch (e: Exception) {
            uiState = uiState.copy(
                message = "Could not create session"
            )
        }
    }
}
```

This creates a new session under the selected patient.

Later, navigation can move to:

```text
MeasurementScreen(sessionId)
```

---

## 12. Load session measurements

When opening a measurement screen, load the session’s measurements.

Add:

```kotlin
fun loadSession(
    sessionId: Long
) {
    viewModelScope.launch {
        uiState = uiState.copy(
            currentSessionId = sessionId,
            isLoading = true,
            message = "Loading session..."
        )

        try {
            val measurements =
                measurementRepository.getMeasurementsForSession(
                    sessionId = sessionId
                )

            val latest = measurements.lastOrNull()

            val latestResult =
                measurementRepository.getLatestResultForSession(
                    sessionId = sessionId
                )

            uiState = uiState.copy(
                measurements = measurements,
                latestRawValue = latest?.rawValue,
                latestProcessedValue = latest?.processedValue,
                latestStatus = latest?.status ?: "",
                latestPrediction = latestResult?.let {
                    com.example.researchapp.ml.PredictionResult(
                        label = it.label,
                        confidence = it.confidence
                    )
                },
                isLoading = false,
                message = ""
            )
        } catch (e: Exception) {
            uiState = uiState.copy(
                isLoading = false,
                message = "Could not load session"
            )
        }
    }
}
```

This lets the measurement and result screens display existing session data.

---

## 13. Add display helper functions

The UI needs readable status text.

Add these helper functions, either in the ViewModel file or a separate helper file:

```kotlin
fun getDeviceStatusText(
    state: DeviceConnectionState
): String {
    return when (state) {
        DeviceConnectionState.DISCONNECTED -> "Disconnected"
        DeviceConnectionState.CONNECTING -> "Connecting"
        DeviceConnectionState.CONNECTED -> "Connected"
        DeviceConnectionState.ERROR -> "Error"
    }
}

fun getAcquisitionStatusText(
    state: AcquisitionState
): String {
    return when (state) {
        AcquisitionState.IDLE -> "Idle"
        AcquisitionState.RECORDING -> "Recording"
        AcquisitionState.STOPPED -> "Stopped"
        AcquisitionState.ERROR -> "Error"
    }
}
```

These convert enum values into readable text.

---

## 14. Add button state helper functions

Inside `ResearchViewModel`, add:

```kotlin
fun canConnectDevice(): Boolean {
    return uiState.deviceConnectionState == DeviceConnectionState.DISCONNECTED
}

fun canDisconnectDevice(): Boolean {
    return uiState.deviceConnectionState == DeviceConnectionState.CONNECTED
}

fun canStartAcquisition(): Boolean {
    return uiState.currentSessionId != null &&
        uiState.deviceConnectionState == DeviceConnectionState.CONNECTED &&
        uiState.acquisitionState != AcquisitionState.RECORDING
}

fun canStopAcquisition(): Boolean {
    return uiState.acquisitionState == AcquisitionState.RECORDING
}

fun canRunInference(): Boolean {
    return uiState.currentSessionId != null &&
        uiState.measurements.isNotEmpty() &&
        uiState.acquisitionState != AcquisitionState.RECORDING
}
```

These functions help the UI decide whether buttons should be enabled.

For example:

```text
Start Acquisition button
 ↓
enabled only when device is connected and session exists
```

---

## 15. Connect and disconnect fake device

Add:

```kotlin
fun connectDevice() {
    if (!canConnectDevice()) {
        return
    }

    uiState = uiState.copy(
        deviceConnectionState = DeviceConnectionState.CONNECTING,
        message = "Connecting device..."
    )

    viewModelScope.launch {
        try {
            measurementRepository.connectDevice()

            uiState = uiState.copy(
                deviceConnectionState = DeviceConnectionState.CONNECTED,
                message = "Device connected"
            )
        } catch (e: Exception) {
            uiState = uiState.copy(
                deviceConnectionState = DeviceConnectionState.ERROR,
                message = "Device connection failed"
            )
        }
    }
}

fun disconnectDevice() {
    if (uiState.acquisitionState == AcquisitionState.RECORDING) {
        stopAcquisition()
    }

    viewModelScope.launch {
        try {
            measurementRepository.disconnectDevice()

            uiState = uiState.copy(
                deviceConnectionState = DeviceConnectionState.DISCONNECTED,
                acquisitionState = AcquisitionState.IDLE,
                message = "Device disconnected"
            )
        } catch (e: Exception) {
            uiState = uiState.copy(
                deviceConnectionState = DeviceConnectionState.ERROR,
                message = "Device disconnection failed"
            )
        }
    }
}
```

This connects the fake device source to the UI.

The UI will call:

```kotlin
viewModel.connectDevice()
viewModel.disconnectDevice()
```

---

## 16. Start acquisition

Add:

```kotlin
fun startAcquisition() {
    val sessionId = uiState.currentSessionId

    if (sessionId == null) {
        uiState = uiState.copy(
            message = "No active session"
        )
        return
    }

    if (!canStartAcquisition()) {
        uiState = uiState.copy(
            message = "Cannot start acquisition"
        )
        return
    }

    uiState = uiState.copy(
        acquisitionState = AcquisitionState.RECORDING,
        message = "Acquisition started"
    )

    viewModelScope.launch {
        while (uiState.acquisitionState == AcquisitionState.RECORDING) {
            try {
                val repetition =
                    uiState.measurements.size + 1

                val measurement =
                    measurementRepository.readAndSaveMeasurement(
                        sessionId = sessionId,
                        repetition = repetition
                    )

                val updatedMeasurements =
                    measurementRepository.getMeasurementsForSession(
                        sessionId = sessionId
                    )

                uiState = uiState.copy(
                    measurements = updatedMeasurements,
                    latestRawValue = measurement.rawValue,
                    latestProcessedValue = measurement.processedValue,
                    latestStatus = measurement.status,
                    message = "Measurement saved"
                )
            } catch (e: Exception) {
                uiState = uiState.copy(
                    acquisitionState = AcquisitionState.ERROR,
                    message = "Measurement acquisition failed"
                )
            }
        }
    }
}
```

This is the first connected fake acquisition loop.

The flow is:

```text
Start button
 ↓
ViewModel.startAcquisition()
 ↓
Repository.readAndSaveMeasurement()
 ↓
FakeDeviceDataSource.readValue()
 ↓
SignalProcessor
 ↓
Room insert
 ↓
uiState update
```

---

## 17. Stop acquisition

Add:

```kotlin
fun stopAcquisition() {
    if (uiState.acquisitionState != AcquisitionState.RECORDING) {
        return
    }

    val sessionId = uiState.currentSessionId

    uiState = uiState.copy(
        acquisitionState = AcquisitionState.STOPPED,
        message = "Acquisition stopped"
    )

    if (sessionId != null) {
        viewModelScope.launch {
            try {
                measurementRepository.endSession(
                    sessionId = sessionId
                )
            } catch (e: Exception) {
                uiState = uiState.copy(
                    message = "Could not mark session as ended"
                )
            }
        }
    }
}
```

When `acquisitionState` becomes `STOPPED`, the loop condition becomes false:

```kotlin
while (uiState.acquisitionState == AcquisitionState.RECORDING)
```

So acquisition stops.

---

## 18. Run fake inference

Add:

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

This connects the fake ML layer.

The flow is:

```text
Run Inference button
 ↓
ViewModel
 ↓
Repository
 ↓
SignalProcessor.extractFeatures()
 ↓
FakeModelRunner
 ↓
ResultEntity saved to Room
 ↓
PredictionResult shown in UI
```

---

## 19. Create `ResearchApp.kt`

Now connect screens with navigation.

Create or update:

```text
ui/ResearchApp.kt
```

Start with:

```kotlin
package com.example.researchapp.ui

import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import com.example.researchapp.viewmodel.ResearchViewModel
import com.example.researchapp.viewmodel.getAcquisitionStatusText
import com.example.researchapp.viewmodel.getDeviceStatusText

object Routes {
    const val PATIENT_LIST = "patient_list"
    const val PATIENT_DETAIL = "patient_detail"
    const val MEASUREMENT = "measurement"
    const val RESULT = "result"
}
```

Then create:

```kotlin
@Composable
fun ResearchApp(
    viewModel: ResearchViewModel = viewModel()
) {
    val navController = rememberNavController()
    val uiState = viewModel.uiState

    NavHost(
        navController = navController,
        startDestination = Routes.PATIENT_LIST
    ) {
        composable(Routes.PATIENT_LIST) {
            LaunchedEffect(Unit) {
                viewModel.loadPatients()
            }

            PatientListScreen(
                patientCode = uiState.patientCode,
                onPatientCodeChange = viewModel::updatePatientCode,
                patientItems = uiState.patientItems,
                onCreatePatientClick = viewModel::createPatient,
                onPatientClick = { patientId ->
                    navController.navigate(
                        "${Routes.PATIENT_DETAIL}/$patientId"
                    )
                }
            )
        }
    }
}
```

This connects the first screen.

---

## 20. Add Patient Detail route

Inside `NavHost`, add:

```kotlin
composable(
    route = "${Routes.PATIENT_DETAIL}/{patientId}"
) { backStackEntry ->
    val patientId = backStackEntry.arguments
        ?.getString("patientId")
        ?.toLongOrNull()

    if (patientId != null) {
        LaunchedEffect(patientId) {
            viewModel.loadPatientDetail(
                patientId = patientId
            )
        }

        PatientDetailScreen(
            patientId = patientId,
            patientCode = uiState.currentPatientCode,
            sessionName = uiState.sessionName,
            onSessionNameChange = viewModel::updateSessionName,
            sessionItems = uiState.sessionItems,
            onCreateSessionClick = viewModel::createSessionForCurrentPatient,
            onSessionClick = { sessionId ->
                navController.navigate(
                    "${Routes.MEASUREMENT}/$sessionId"
                )
            },
            onBackClick = {
                navController.popBackStack()
            }
        )
    }
}
```

This means:

```text
open patient detail
 ↓
load patient and sessions
 ↓
display PatientDetailScreen
```

---

## 21. Add Measurement route

Inside `NavHost`, add:

```kotlin
composable(
    route = "${Routes.MEASUREMENT}/{sessionId}"
) { backStackEntry ->
    val sessionId = backStackEntry.arguments
        ?.getString("sessionId")
        ?.toLongOrNull()

    if (sessionId != null) {
        LaunchedEffect(sessionId) {
            viewModel.loadSession(
                sessionId = sessionId
            )
        }

        MeasurementScreen(
            sessionId = sessionId,
            deviceStatusText = getDeviceStatusText(
                uiState.deviceConnectionState
            ),
            acquisitionStatusText = getAcquisitionStatusText(
                uiState.acquisitionState
            ),
            latestRawValue = uiState.latestRawValue,
            latestProcessedValue = uiState.latestProcessedValue,
            latestStatus = uiState.latestStatus,
            measurementCount = uiState.measurements.size,
            canConnectDevice = viewModel.canConnectDevice(),
            canDisconnectDevice = viewModel.canDisconnectDevice(),
            canStartAcquisition = viewModel.canStartAcquisition(),
            canStopAcquisition = viewModel.canStopAcquisition(),
            canRunInference = viewModel.canRunInference(),
            onConnectDeviceClick = viewModel::connectDevice,
            onDisconnectDeviceClick = viewModel::disconnectDevice,
            onStartAcquisitionClick = viewModel::startAcquisition,
            onStopAcquisitionClick = viewModel::stopAcquisition,
            onRunInferenceClick = viewModel::runInferenceForCurrentSession,
            onViewResultClick = {
                navController.navigate(
                    "${Routes.RESULT}/$sessionId"
                )
            },
            onBackClick = {
                navController.popBackStack()
            }
        )
    }
}
```

This connects the measurement screen to:

```text
device connection
acquisition
inference
navigation to result
```

---

## 22. Add Result route

Inside `NavHost`, add:

```kotlin
composable(
    route = "${Routes.RESULT}/{sessionId}"
) { backStackEntry ->
    val sessionId = backStackEntry.arguments
        ?.getString("sessionId")
        ?.toLongOrNull()

    if (sessionId != null) {
        LaunchedEffect(sessionId) {
            viewModel.loadSession(
                sessionId = sessionId
            )
        }

        ResultScreen(
            sessionId = sessionId,
            predictionLabel = uiState.latestPrediction?.label,
            predictionConfidence = uiState.latestPrediction?.confidence,
            measurementCount = uiState.measurements.size,
            message = uiState.message,
            onExportClick = {
                // Export will be connected in a later lesson.
            },
            onBackClick = {
                navController.popBackStack()
            }
        )
    }
}
```

Now the Result screen can display the latest prediction.

Export is not connected yet.

That will come in a later lesson.

---

## 23. Full navigation flow after Lesson 31

After connecting these routes, the app flow is:

```text
PatientListScreen
 ↓ click patient
PatientDetailScreen
 ↓ click session
MeasurementScreen
 ↓ click View Result
ResultScreen
```

And the data/action flow is:

```text
Screen button
 ↓
ViewModel function
 ↓
Repository function
 ↓
Room / FakeDevice / SignalProcessor / FakeModel
 ↓
uiState update
 ↓
Screen updates
```

This is now a connected app skeleton.

---

## 24. Important limitation

At this stage, there may still be practical Android Studio setup issues, such as:

```text
Room Gradle dependencies
Navigation Compose dependency
KSP setup
package imports
Room 3 vs Room 2 import differences
```

This lesson focuses on how the app should be connected structurally.

If the project does not compile immediately, the next step would be to fix dependencies and imports.

That is normal.

---

## 25. What you learned in Lesson 31

You connected:

```text
ResearchUiState
ResearchViewModel
ResearchApp navigation
PatientListScreen
PatientDetailScreen
MeasurementScreen
ResultScreen
```

You added ViewModel functions to:

```text
load patients
create patients
load patient detail
create sessions
load session measurements
connect fake device
start acquisition
stop acquisition
run fake inference
```

The most important mental model is:

```text
Screens display state and call callbacks.
ViewModel handles app state and user actions.
Repository performs data/device/processing/ML work.
```

The connected structure is now:

```text
ResearchApp
 ↓
Screens
 ↓
ResearchViewModel
 ↓
MeasurementRepository
 ↓
Room / FakeDevice / SignalProcessor / FakeModel
```

This is the first connected version of the fake-data research app.

---

## 26. Lesson 32 preview

In Lesson 32, we will add session CSV export.

We will implement:

```text
ExportFormatter
buildSessionCsvExport()
CreateDocument save flow
Export button on ResultScreen
```

Then the app will support the complete fake workflow:

```text
create patient
create session
collect fake processed data
run fake inference
export session CSV
```
