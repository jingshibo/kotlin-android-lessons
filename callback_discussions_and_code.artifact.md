# 📚 Complete Discussions & Source Code: Callbacks, Lambdas & State Hoisting

This document contains all the Q&A discussions, technical explanations, and complete source code snippets regarding **Callback Functions, Lambdas, and Event Hoisting** in the ResearchDeviceUi codebase.

---

## 💬 Part 1: Q&A Discussions & Technical Explanations

### Q1: What does `TabletScreen.valueOf(currentScreenName)` mean?
**Explanation**:
In `MainActivity.kt`, top-level navigation position is stored as a `String` (`var currentScreenName by rememberSaveable { mutableStateOf("Device") }`) so Android's `rememberSaveable` can easily save it during screen rotations.
`TabletScreen.valueOf(currentScreenName)` converts that `String` name back into the strongly-typed `TabletScreen` enum object (e.g. `TabletScreen.Device` or `TabletScreen.Data`), allowing type-safe `when` branching and property access (`navLabel`, `shortcut`).

---

### Q2: How does `when (currentScreen)` in `TabletShell` connect to the `content` parameter?
**Explanation**:
In Kotlin, if the last parameter of a function is a lambda function (like `content: @Composable () -> Unit`), Kotlin allows putting the curly braces `{ ... }` outside the function's parentheses:
```kotlin
TabletShell(
    activeScreen = activeNavScreen,
    onNavigate = { currentScreenName = it.name }
) { // 👈 THIS ENTIRE BLOCK IS 'content'!
    when (currentScreen) {
        TabletScreen.Device -> DeviceScreen(...)
        TabletScreen.Data -> DataScreen(...)
        ...
    }
}
```
Inside `TabletShell`, the `Box` layout calls `content()`, which executes the `when (currentScreen)` block and displays the active screen!

---

### Q3: What do `activeScreen = activeNavScreen` and `onNavigate = { currentScreenName = it.name }` mean?
**Explanation**:
* **`activeScreen = activeNavScreen`**: Tells `TabletShell` which side rail tab to visually highlight in green/teal. When opening full-screen modals like `AddPatient`, `activeNavScreen` keeps pointing to the parent tab (`Data` or `Repository`) so the side rail highlight stays on the parent section!
* **`onNavigate = { currentScreenName = it.name }`**: Defines the callback lambda passed into `TabletShell`. When a tab is tapped, `onNavigate` receives the clicked `TabletScreen` enum (`it`) and updates `currentScreenName = it.name`, triggering Compose to switch screens.

---

### Q4: What does `onDeviceSelect: ((String) -> Unit)? = null` mean?
**Explanation**:
* **`(String) -> Unit`**: A function signature taking 1 `String` parameter and returning no value (`Unit` = void).
* **`?`**: Makes the function parameter nullable so callers can pass `null` (e.g. in Composable previews).
* **`= null`**: Default parameter value.
* **`onDeviceSelect?.invoke("BT-Sensor-01")`**: The safe-call `.invoke(...)` method executes the function safely if not null.

---

### Q5: Demystifying `onNavigate = { currentScreenName = it.name }` vs `onClick = { onNavigate(item) }`
**Explanation**:
* **`onClick = { onNavigate(item) }`** (inside `SideNavBar`): Executes when a specific tab row is clicked, capturing that row's `item` enum (e.g. `TabletScreen.Data`) and passing it up to `onNavigate`.
* **`onNavigate = { currentScreenName = it.name }`** (inside `MainActivity.kt`): The listener defined at the top. `it` represents the `TabletScreen` object passed up. It extracts `it.name` (`"Data"`) and updates `currentScreenName`.

---

### Q6: Why is `AddPatient` treated as a top-level route in `MainActivity.kt`, but Repository sub-tabs are not?
**Explanation**:
* **`AddPatient`**: A cross-screen modal flow that can be opened from multiple screens (`Data` or `Repository`). It needs top-level back-stack memory (`previousScreenName`) to return the user to whichever screen opened it.
* **Repository Sub-Tabs (`Sessions`, `Patients`, `Devices`)**: Local sub-views that belong exclusively to the Repository domain, sharing the same top header banner, card layout, and side rail highlight.

---

### Q7: Passing Functions vs. Executing Functions (`.clickable(enabled, onClick)`)
**Explanation**:
Neither `SideNavBar(onNavigate = onNavigate)` nor `.clickable(onClick = onClick)` executes the function immediately when the screen is drawn. Both lines are merely **passing function references down**.
Execution **ONLY happens later when a physical finger touches the glass on the tablet**, causing Compose's touch listener to execute `onClick()`.

---

### Q8: Why pass `{ dataViewModel.clearDataScreen() }` instead of passing `dataViewModel` directly into `DeviceScreen`?
**Explanation**:
* **Loose Coupling / Modular Architecture**: `DeviceScreen` is for managing Bluetooth sensors. It should remain 100% independent and not know `DataViewModel` exists.
* **Previews & Testing**: Passing callback lambdas (`onDeviceSelect: () -> Unit`) allows rendering `@Preview` composables effortlessly without mocking ViewModel instances.
* **Mediator Pattern**: `MainActivity.kt` acts as the coordinator/mediator between decoupled screen ViewModels.

---

### Q9: Who provides the String parameter in `onDeviceSelect?.invoke("BT-Sensor-01")`?
**Explanation**:
* **`DeviceScreen` is the PRODUCER**: `DeviceScreen` knows which device was tapped by the user, so `DeviceScreen` **provides the string** when invoking `onDeviceSelect?.invoke("BT-Sensor-01")`.
* **`MainActivity` is the CONSUMER**: `MainActivity` receives the event. It can choose to read the string or ignore it (`onDeviceSelect = { dataViewModel.clearDataScreen() }`) because it only cares that the selection event occurred.

---

## 💻 Part 2: Complete Source Code Snippets

### 1. `MainActivity.kt`
```kotlin
package com.example.researchdeviceui

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.Surface
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.saveable.rememberSaveable
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.tooling.preview.Preview
import androidx.lifecycle.viewmodel.compose.viewModel
import com.example.researchdeviceui.ui.components.TabletShell
import com.example.researchdeviceui.ui.screens.data.DataScreen
import com.example.researchdeviceui.ui.screens.device.DeviceScreen
import com.example.researchdeviceui.ui.screens.repository.RepositoryScreen
import com.example.researchdeviceui.ui.screens.repository.patients.AddNewPatientScreen
import com.example.researchdeviceui.ui.screens.settings.SettingsScreen
import com.example.researchdeviceui.ui.theme.AppBackground
import com.example.researchdeviceui.ui.theme.ResearchDeviceTheme
import com.example.researchdeviceui.viewmodel.common.TabletScreen
import com.example.researchdeviceui.viewmodel.data.DataViewModel
import com.example.researchdeviceui.viewmodel.device.DeviceViewModel
import com.example.researchdeviceui.viewmodel.repository.DevicesViewModel
import com.example.researchdeviceui.viewmodel.repository.HistoryViewModel
import com.example.researchdeviceui.viewmodel.repository.PatientsViewModel
import com.example.researchdeviceui.viewmodel.settings.SettingsViewModel

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            ResearchDeviceTheme {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = AppBackground
                ) {
                    ResearchTabletApp()
                }
            }
        }
    }
}

/**
 * Top-level screen navigation control & state coordinator.
 */
@Composable
private fun ResearchTabletApp(
    // SECTION 1: ViewModels & State Holder Instantiation
    deviceViewModel: DeviceViewModel = viewModel(),
    devicesViewModel: DevicesViewModel = viewModel(),
    dataViewModel: DataViewModel = viewModel(),
    historyViewModel: HistoryViewModel = viewModel(),
    patientsViewModel: PatientsViewModel = viewModel(),
    settingsViewModel: SettingsViewModel = viewModel()
) {
    // SECTION 2: Navigation Position State
    var currentScreenName by rememberSaveable { mutableStateOf(TabletScreen.Device.name) }
    var previousScreenName by rememberSaveable { mutableStateOf(TabletScreen.Data.name) }

    // SECTION 3: Active Screen Resolution & State Observation
    // Converts saved String screen name back to type-safe TabletScreen enum object
    val currentScreen = TabletScreen.valueOf(currentScreenName)
    // When in full-screen modal (AddPatient), keep the active rail nav highlight on the parent tab
    val activeNavScreen = if (currentScreen == TabletScreen.AddPatient) {
        TabletScreen.valueOf(previousScreenName)
    } else {currentScreen}
    // For controlling the lock of onNavigate tabs
    val dataState by dataViewModel.uiState.collectAsStateWithLifecycle()

    // SECTION 4: Navigation Shell & Cross-Screen Event Callbacks
    TabletShell(
        // Highlights active side rail tab (preserves parent tab during AddPatient modal)
        activeScreen = activeNavScreen,
        isNavLocked = dataState.isTransferring || dataState.isAnalyzing,
        // onNavigate tells it what to do when the user taps another tab.
        // When a tab is tapped, update currentScreenName to that tab's name.
        onNavigate = { currentScreenName = it.name } // Define the Lambda function and use as the onNavigate input. 'it' refers to selectedScreen: TabletScreen as the function's input argument
    ) {
        when (currentScreen) {
            // 1. Bluetooth Sensor Management Tab
            TabletScreen.Device -> DeviceScreen(
                viewModel = deviceViewModel, // each screen has its own viewModel
                onDeviceSelect = { // do not pass dataViewModel as the input for better decoupling/modular
                    dataViewModel.clearDataScreen() //  onDeviceSelect defines an input which should be provided when it is called.
                }
            )

            // 2. Dataset Transfer & AI Inference Tab
            TabletScreen.Data -> DataScreen(
                viewModel = dataViewModel,
                onAddPatient = {
                    previousScreenName = TabletScreen.Data.name
                    currentScreenName = TabletScreen.AddPatient.name
                }
            )

            // 3. Repository & Data Audit Tab (Sessions, Patients, Devices)
            TabletScreen.Repository -> RepositoryScreen(
                viewModel = historyViewModel,
                patientsViewModel = patientsViewModel,
                devicesViewModel = devicesViewModel,
                onSelectSessionForData = { session ->
                    dataViewModel.selectPatient(session.patientId)
                    if (session.arm != "--" && session.arm.isNotBlank()) {
                        dataViewModel.selectArm(session.arm)
                    }
                    try {
                        val parsedDate = java.time.LocalDate.parse(session.recordingDate)
                        dataViewModel.selectDate(parsedDate.year, parsedDate.monthValue, parsedDate.dayOfMonth)
                    } catch (_: Exception) {}
                    currentScreenName = TabletScreen.Data.name
                },
                onAddPatient = {
                    historyViewModel.selectTab("Patients")
                    previousScreenName = TabletScreen.Repository.name
                    currentScreenName = TabletScreen.AddPatient.name
                }
            )

            // 4. System Settings & Diagnostics Tab
            TabletScreen.Settings -> SettingsScreen(
                viewModel = settingsViewModel
            )

            // 5. Add New Participant Modal Form
            // It can be opened from multiple screens and needs previousScreenName to navigate back, so we treat it as a separate screen
            TabletScreen.AddPatient -> AddNewPatientScreen(
                patientsViewModel = patientsViewModel,
                openedFromScreen = if (previousScreenName == TabletScreen.Data.name) "Data" else "Repository",
                onCancel = {
                    patientsViewModel.resetNewPatientForm()
                    currentScreenName = previousScreenName
                },
                onSubmitPatient = {
                    patientsViewModel.submitNewPatient { patientCode ->
                        val cleanCode = patientCode.trim()
                        dataViewModel.selectPatient(cleanCode)
                        currentScreenName = previousScreenName
                    }
                }
            )
        }
    }
}

@Preview(widthDp = 1280, heightDp = 800, showBackground = true)
@Composable
private fun ResearchTabletAppPreview() {
    ResearchDeviceTheme {
        ResearchTabletApp()
    }
}
```

---

### 2. `NavigationComponents.kt`
```kotlin
package com.example.researchdeviceui.ui.components

import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxHeight
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.width
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import com.example.researchdeviceui.viewmodel.common.TabletScreen
import com.example.researchdeviceui.ui.theme.AppBackground
import com.example.researchdeviceui.ui.theme.Border
import com.example.researchdeviceui.ui.theme.Primary
import com.example.researchdeviceui.ui.theme.PrimaryDark
import com.example.researchdeviceui.ui.theme.PrimarySoft
import com.example.researchdeviceui.ui.theme.SideNavigation
import com.example.researchdeviceui.ui.theme.TextMuted
import com.example.researchdeviceui.ui.theme.TextPrimary

/**
 * Root tablet shell wrapper providing the top bar, side navigation rail, and main screen area.
 */
@Composable
fun TabletShell(
    activeScreen: TabletScreen,
    isNavLocked: Boolean = false,
    onNavigate: (TabletScreen) -> Unit, // This input parameter is a function named onNavigate that MUST accept a TabletScreen data as its first argument.
    content: @Composable () -> Unit
) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .background(AppBackground)
    ) {
        TopTabletBar()

        Row(modifier = Modifier.fillMaxSize()) {
            // Passes the active tab highlight and onNavigate callback down to SideNavBar
            SideNavBar(
                activeScreen = activeScreen,
                isNavLocked = isNavLocked,
                onNavigate = onNavigate
            )

            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .padding(start = 34.dp, top = 28.dp, end = 34.dp, bottom = 28.dp)
            ) {
                content()
            }
        }
    }
}

/**
 * Top app banner displaying system endpoint title and branding.
 */
@Composable
fun TopTabletBar() {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .height(58.dp)
            .border(width = 1.dp, color = Border)
            .padding(horizontal = 32.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        Text(
            text = "Endpoint",
            color = TextMuted,
            fontWeight = FontWeight.Bold,
            fontSize = 14.sp
        )

        Spacer(modifier = Modifier.weight(1f))

        Text(
            text = "Research data collection tablet",
            color = TextMuted,
            fontSize = 14.sp
        )

        Spacer(modifier = Modifier.weight(1f))

        Text(
            text = "iiTech",
            fontWeight = FontWeight.Bold,
            color = TextMuted,
            fontSize = 14.sp
        )
    }
}

/**
 * Left vertical navigation rail holding main screen tab items.
 */
@Composable
fun SideNavBar(
    activeScreen: TabletScreen,
    isNavLocked: Boolean = false,
    onNavigate: (TabletScreen) -> Unit // onNavigate callback function requires a TabletScreen datatype as input argument
) {
    val items = listOf(
        TabletScreen.Device,
        TabletScreen.Data,
        TabletScreen.Repository,
        TabletScreen.Settings
    )

    Column(
        modifier = Modifier
            .width(192.dp)
            .fillMaxHeight()
            .background(SideNavigation)
            .padding(top = 36.dp, start = 12.dp, end = 12.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        Text(
            text = "Research App",
            color = Primary,
            fontWeight = FontWeight.Bold,
            fontSize = 17.sp,
            modifier = Modifier.padding(start = 10.dp, bottom = 28.dp)
        )

        items.forEach { item ->
            // Captures the tapped TabletScreen 'item' and passes it up via onNavigate(item)
            NavigationItem(
                letter = item.shortcut,
                label = item.navLabel,
                selected = item == activeScreen, // Checks if this tab is the currently active screen on the tablet.
                enabled = !isNavLocked,
                onClick = { if (!isNavLocked) onNavigate(item) } // 2. Pass the obtained 'item' (selected screen name) parameter to the onNavigation function for later NavigationItem execution
            )
        }
    }
}

/**
 * Individual navigation tab item in the side rail.
 */
@Composable
fun NavigationItem(
    letter: String,
    label: String,
    selected: Boolean,
    enabled: Boolean = true,
    onClick: () -> Unit // a callback Lambda function
) {
    val itemAlpha = if (enabled || selected) 1.0f else 0.4f
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .height(54.dp)
            .clip(RoundedCornerShape(16.dp))
            .background(if (selected) PrimarySoft else Color.Transparent)
            // Assign the onClick callback function to onClick. It will only be executed when user click
            .clickable(enabled = enabled, onClick = onClick) // when clicking, execute the { if (!isNavLocked) onNavigate(item) } function
            .padding(horizontal = 14.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        Box(
            modifier = Modifier
                .size(26.dp)
                .clip(RoundedCornerShape(8.dp))
                .background(if (selected) Primary else Color.Transparent),
            contentAlignment = Alignment.Center
        ) {
            Text(
                text = letter,
                color = (if (selected) Color.White else TextMuted).copy(alpha = itemAlpha),
                fontWeight = FontWeight.Bold,
                fontSize = 12.sp
            )
        }

        Spacer(modifier = Modifier.width(14.dp))

        Text(
            text = label,
            color = (if (selected) PrimaryDark else TextMuted).copy(alpha = itemAlpha),
            fontWeight = FontWeight.Bold,
            fontSize = 14.sp
        )
    }
}
```

---

### 3. `DeviceScreen.kt`
```kotlin
package com.example.researchdeviceui.ui.screens.device

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxHeight
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.width
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.verticalScroll
import androidx.compose.material3.AlertDialog
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.lifecycle.viewmodel.compose.viewModel
import com.example.researchdeviceui.ui.components.GhostButton
import com.example.researchdeviceui.ui.components.ScreenHeader
import com.example.researchdeviceui.ui.components.SoftButton
import com.example.researchdeviceui.ui.theme.CardBackground
import com.example.researchdeviceui.ui.theme.TextMuted
import com.example.researchdeviceui.ui.theme.TextPrimary
import com.example.researchdeviceui.viewmodel.device.DeviceViewModel

@Composable
fun DeviceScreen(
    viewModel: DeviceViewModel = viewModel(),
    onDeviceSelect: ((String) -> Unit)? = null // A callback function as the Device Screen argument. It requires a string as the input.
) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()

    val isConnected = state.selectedDevice.isNotEmpty()
    val activeMeasuring = isConnected && state.isMeasuring
    val warningMessage = viewModel.getResourceWarningMessage()
    val devices = viewModel.getScannedDevices()

    if (state.showLowResourceWarningDialog && warningMessage != null) {
        AlertDialog(
            onDismissRequest = { viewModel.dismissWarningDialog() },
            title = {
                Text(
                    text = "Resource Warning",
                    fontWeight = FontWeight.Bold,
                    color = TextPrimary
                )
            },
            text = {
                Text(
                    text = warningMessage,
                    color = TextMuted
                )
            },
            confirmButton = {
                GhostButton(
                    text = "Continue Measurement",
                    danger = true,
                    onClick = { viewModel.confirmMeasurementWithWarning() }
                )
            },
            dismissButton = {
                SoftButton(
                    text = "Cancel",
                    onClick = { viewModel.dismissWarningDialog() }
                )
            },
            containerColor = CardBackground,
            titleContentColor = TextPrimary,
            textContentColor = TextMuted
        )
    }

    Column(modifier = Modifier.fillMaxSize()) {
        ScreenHeader(
            title = "Device",
            subtitle = "Connect, inspect, manage device data, and start measurement.",
            modeTitle = "Bluetooth mode",
            modeSubtitle = "Status auto-read on connection"
        )

        Spacer(modifier = Modifier.height(22.dp))

        Row(
            modifier = Modifier.fillMaxSize(),
            horizontalArrangement = Arrangement.spacedBy(24.dp)
        ) {
            BluetoothDeviceCard(
                devices = devices,
                selectedDevice = state.selectedDevice,
                onDisconnect = {
                    viewModel.disconnect()
                    onDeviceSelect?.invoke("") // .invoke is the built-in method of every Kotlin function, functioning same as running this function
                    // ? is a safe-call operator in case onDeviceSelect is null (for example, in a Composable Preview where no callback is passed)
                },
                onSelectDevice = { name ->
                    viewModel.selectDevice(name)
                    onDeviceSelect?.invoke(name) // name is the input argument required by the onDeviceSelect callback function to run (MainActivity ignored this)
                },
                modifier = Modifier
                    .width(420.dp)
                    .fillMaxHeight()
            )

            Column(
                modifier = Modifier
                    .weight(1f)
                    .fillMaxHeight()
                    .verticalScroll(rememberScrollState()),
                verticalArrangement = Arrangement.spacedBy(22.dp)
            ) {
                DeviceConditionCard(
                    selectedDevice = state.selectedDevice,
                    isConnected = isConnected,
                    isMeasuring = activeMeasuring,
                    lastChecked = if (isConnected) state.lastChecked else "-"
                )

                DeviceControlsCard(
                    isConnected = isConnected,
                    isMeasuring = activeMeasuring,
                    rebootEndTimeMillis = state.rebootEndTimeMillis,
                    onRebootDevice = { viewModel.rebootDevice() },
                    onToggleMeasurement = { viewModel.toggleMeasurement() },
                    modifier = Modifier.fillMaxWidth()
                )
            }
        }
    }
}
```

---

### 4. `NavigationState.kt`
```kotlin
package com.example.researchdeviceui.viewmodel.common

/**
 * Top-level navigation targets for the tablet application shell.
 *
 * @property navLabel Display label shown in the navigation side rail.
 * @property shortcut Single-character shortcut representation.
 */
enum class TabletScreen(
    val navLabel: String,
    val shortcut: String
) {
    Device("Device", "D"),
    Data("Data", "A"),
    Repository("Repository", "R"),
    Settings("Settings", "S"),
    AddPatient("Add Patient", "+")
}

/**
 * Table column sorting order direction enum.
 */
enum class SortOrder {
    ASCENDING, DESCENDING
}
```
