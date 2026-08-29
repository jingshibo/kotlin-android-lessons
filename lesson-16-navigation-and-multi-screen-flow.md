# Lesson 16 — Multi-Screen App Navigation

In Lesson 15, we changed the app from a simple measurement logger into a more realistic research data model:

```text
Patient
 ↓
Session
 ↓
Measurement
 ↓
Result
```

That was important because a research app should not only save values. It should also know:

```text
who/what was measured
which session the measurement belongs to
when the session started
what result was produced
```

But now we have a new problem.

One screen is no longer enough.

So in Lesson 16, we move from:

```text
one large ResearchScreen
```

to:

```text
multiple screens
```

The goal is to make the app feel more like a real tablet research application.

---

## 1. Why we need multiple screens

So far, our app screen contains many things:

```text
patient/sample input
session input
device connection status
start/stop acquisition
latest value
measurement history
database saving
result display
```

This is okay for learning, but if we keep adding features, the screen will become too crowded.

A real research app is usually divided into screens such as:

```text
Patient List Screen
Patient Detail Screen
Session / Measurement Screen
Result Screen
Settings Screen
Export Screen
```

Each screen has one main responsibility.

This makes the app easier to use and easier to develop.

---

## 2. The screen structure we want

For our current learning app, we can use four main screens:

```text
Patient List Screen
 ↓
Patient Detail Screen
 ↓
Session Measurement Screen
 ↓
Result Screen
```

The flow is:

```text
Open app
 ↓
See all patients/samples
 ↓
Select one patient/sample
 ↓
See that patient's sessions
 ↓
Start or open a session
 ↓
Collect measurements
 ↓
View result
```

This matches the data model from Lesson 15.

---

## 3. What is navigation?

Navigation simply means:

```text
moving from one screen to another
```

For example:

```text
Patient List
 ↓ click patient
Patient Detail
 ↓ click session
Measurement Screen
 ↓ click view result
Result Screen
```

In Jetpack Compose apps, Android provides **Navigation Compose** for navigating between composable screens. The official Android documentation says that if an app is built entirely with Jetpack Compose, Navigation Compose is the appropriate navigation option. citeturn859815search14

The main tools are:

```text
NavController
NavHost
routes
composable destinations
```

Do not worry if these terms are new. We will introduce them one by one.

---

## 4. Basic mental model

Think of navigation like this:

```text
NavController
 ↓
the object that tells the app where to go

NavHost
 ↓
the container that displays the current screen

Route
 ↓
the name/address of a screen
```

For example:

```text
Route: patient_list
 ↓
show PatientListScreen

Route: patient_detail/3
 ↓
show PatientDetailScreen for patient ID 3

Route: measurement/10
 ↓
show MeasurementScreen for session ID 10
```

The Android documentation describes `rememberNavController()` as the Compose way to create a `NavController`, and `NavHost` as the composable that defines the navigation graph. citeturn859815search4turn859815search2

---

## 5. Add Navigation dependency

To use Navigation Compose, your app needs the Navigation Compose dependency.

In `build.gradle.kts`, you will commonly need something like:

```kotlin
implementation("androidx.navigation:navigation-compose:<latest-version>")
```

The exact version may change, so in a real project you should check Android Studio’s suggestions or the official AndroidX Navigation release notes. The important idea is:

```text
Navigation Compose allows the app to move between composable screens.
```

For this lesson, focus on the app structure first.

---

## 6. Create screen routes

For beginner learning, we can start with simple string routes.

Create an object:

```kotlin
object Routes {
    const val PATIENT_LIST = "patient_list"
    const val PATIENT_DETAIL = "patient_detail"
    const val MEASUREMENT = "measurement"
    const val RESULT = "result"
}
```

This avoids writing raw strings everywhere.

Instead of:

```kotlin
navController.navigate("patient_list")
```

we can write:

```kotlin
navController.navigate(Routes.PATIENT_LIST)
```

This is safer because the route names are stored in one place.

Later, we can move toward modern type-safe routes. Android’s current navigation guidance supports type-safe routes using serializable objects or classes, which can reduce runtime mistakes from route typos or wrong argument types. citeturn859815search0turn859815search1

But for this lesson, string routes are easier for understanding the basic idea.

---

## 7. Create the app navigation host

Create a new composable:

```kotlin
@Composable
fun ResearchApp() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = Routes.PATIENT_LIST
    ) {
        composable(Routes.PATIENT_LIST) {
            PatientListScreen(
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

This is the beginning of our navigation system.

Let us break it down.

---

## 8. `rememberNavController()`

This line:

```kotlin
val navController = rememberNavController()
```

creates the navigation controller.

The `navController` is responsible for moving between screens.

For example:

```kotlin
navController.navigate("patient_list")
```

means:

```text
go to the patient list screen
```

And:

```kotlin
navController.popBackStack()
```

means:

```text
go back to the previous screen
```

---

## 9. `NavHost`

This part:

```kotlin
NavHost(
    navController = navController,
    startDestination = Routes.PATIENT_LIST
) {
    ...
}
```

means:

```text
Use this navController.
Start the app at the patient list screen.
Display whichever screen matches the current route.
```

So the `NavHost` is like the screen container.

It decides which screen should currently be visible.

---

## 10. First screen: Patient List Screen

The Patient List Screen should show all patients/samples.

For now, we can make a simple version with fake data:

```kotlin
@Composable
fun PatientListScreen(
    onPatientClick: (Long) -> Unit
) {
    Column(
        modifier = Modifier.padding(16.dp)
    ) {
        Text("Patients")

        Spacer(modifier = Modifier.height(16.dp))

        Button(
            onClick = {
                onPatientClick(1L)
            }
        ) {
            Text("Open Patient P001")
        }

        Button(
            onClick = {
                onPatientClick(2L)
            }
        ) {
            Text("Open Patient P002")
        }
    }
}
```

Notice this parameter:

```kotlin
onPatientClick: (Long) -> Unit
```

This means:

```text
When the user clicks a patient,
send the patient ID back to the navigation system.
```

The screen itself does not directly control navigation.

It only says:

```text
A patient was clicked.
```

This is a good Compose habit.

---

## 11. Add patient detail route with argument

The Patient Detail Screen needs to know:

```text
which patient was selected
```

So we need to pass:

```text
patientId
```

A route with an argument can look like this:

```text
patient_detail/{patientId}
```

Add this route pattern:

```kotlin
object Routes {
    const val PATIENT_LIST = "patient_list"
    const val PATIENT_DETAIL = "patient_detail"
    const val MEASUREMENT = "measurement"
    const val RESULT = "result"
}
```

Then inside `NavHost`, add:

```kotlin
composable(
    route = "${Routes.PATIENT_DETAIL}/{patientId}"
) { backStackEntry ->
    val patientId = backStackEntry.arguments
        ?.getString("patientId")
        ?.toLongOrNull()

    if (patientId != null) {
        PatientDetailScreen(
            patientId = patientId,
            onStartSessionClick = { sessionId ->
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

This part:

```kotlin
backStackEntry.arguments
    ?.getString("patientId")
    ?.toLongOrNull()
```

means:

```text
Read patientId from the route.
Convert it from String to Long.
If conversion fails, return null.
```

Again, this uses Kotlin null safety.

---

## 12. Patient Detail Screen

The Patient Detail Screen should show information for one patient and allow the user to start or open a session.

A simple version:

```kotlin
@Composable
fun PatientDetailScreen(
    patientId: Long,
    onStartSessionClick: (Long) -> Unit,
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

        Spacer(modifier = Modifier.height(16.dp))

        Button(
            onClick = {
                onStartSessionClick(10L)
            }
        ) {
            Text("Start Session")
        }
    }
}
```

For now, we use a fake session ID:

```kotlin
10L
```

Later, this should come from Room:

```text
create session in database
 ↓
get generated sessionId
 ↓
navigate to MeasurementScreen(sessionId)
```

---

## 13. Add Measurement Screen route

Now add a route for the Measurement Screen:

```kotlin
composable(
    route = "${Routes.MEASUREMENT}/{sessionId}"
) { backStackEntry ->
    val sessionId = backStackEntry.arguments
        ?.getString("sessionId")
        ?.toLongOrNull()

    if (sessionId != null) {
        MeasurementScreen(
            sessionId = sessionId,
            onResultClick = {
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

The important idea:

```text
MeasurementScreen needs sessionId.
```

Why?

Because in Lesson 15, we decided that measurements should belong to a session.

So the measurement screen must know:

```text
which session it is recording for
```

---

## 14. Measurement Screen

This screen is where acquisition happens.

A simple version:

```kotlin
@Composable
fun MeasurementScreen(
    sessionId: Long,
    onResultClick: () -> Unit,
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

        Text("Measurement Screen")
        Text("Session ID: $sessionId")

        Spacer(modifier = Modifier.height(16.dp))

        Text("Device status: Connected")
        Text("Acquisition status: Stopped")
        Text("Latest value: --")
        Text("Measurements: 0")

        Spacer(modifier = Modifier.height(16.dp))

        Row(
            horizontalArrangement = Arrangement.spacedBy(12.dp)
        ) {
            Button(
                onClick = {
                    // later: viewModel.startAcquisition(sessionId)
                }
            ) {
                Text("Start")
            }

            Button(
                onClick = {
                    // later: viewModel.stopAcquisition()
                }
            ) {
                Text("Stop")
            }
        }

        Spacer(modifier = Modifier.height(16.dp))

        Button(
            onClick = onResultClick
        ) {
            Text("View Result")
        }
    }
}
```

For now, this is a simple navigation version.

Later, we will connect it to the real `ResearchViewModel`.

---

## 15. Add Result Screen route

The Result Screen also needs the session ID because the result belongs to a session.

Add:

```kotlin
composable(
    route = "${Routes.RESULT}/{sessionId}"
) { backStackEntry ->
    val sessionId = backStackEntry.arguments
        ?.getString("sessionId")
        ?.toLongOrNull()

    if (sessionId != null) {
        ResultScreen(
            sessionId = sessionId,
            onBackClick = {
                navController.popBackStack()
            }
        )
    }
}
```

---

## 16. Result Screen

A simple version:

```kotlin
@Composable
fun ResultScreen(
    sessionId: Long,
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

        Text("Result Screen")
        Text("Session ID: $sessionId")

        Spacer(modifier = Modifier.height(16.dp))

        Text("Prediction: Not available yet")
        Text("Confidence: --")
    }
}
```

Later, this screen will show:

```text
classification label
confidence score
summary statistics
export button
session metadata
```

But for Lesson 16, the goal is navigation.

---

## 17. Full navigation structure

Now the navigation structure looks like this:

```kotlin
@Composable
fun ResearchApp() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = Routes.PATIENT_LIST
    ) {
        composable(Routes.PATIENT_LIST) {
            PatientListScreen(
                onPatientClick = { patientId ->
                    navController.navigate(
                        "${Routes.PATIENT_DETAIL}/$patientId"
                    )
                }
            )
        }

        composable(
            route = "${Routes.PATIENT_DETAIL}/{patientId}"
        ) { backStackEntry ->
            val patientId = backStackEntry.arguments
                ?.getString("patientId")
                ?.toLongOrNull()

            if (patientId != null) {
                PatientDetailScreen(
                    patientId = patientId,
                    onStartSessionClick = { sessionId ->
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

        composable(
            route = "${Routes.MEASUREMENT}/{sessionId}"
        ) { backStackEntry ->
            val sessionId = backStackEntry.arguments
                ?.getString("sessionId")
                ?.toLongOrNull()

            if (sessionId != null) {
                MeasurementScreen(
                    sessionId = sessionId,
                    onResultClick = {
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

        composable(
            route = "${Routes.RESULT}/{sessionId}"
        ) { backStackEntry ->
            val sessionId = backStackEntry.arguments
                ?.getString("sessionId")
                ?.toLongOrNull()

            if (sessionId != null) {
                ResultScreen(
                    sessionId = sessionId,
                    onBackClick = {
                        navController.popBackStack()
                    }
                )
            }
        }
    }
}
```

This is the core of Lesson 16.

The app now has multiple screens.

---

## 18. Use `ResearchApp()` in `MainActivity`

In `MainActivity.kt`, instead of directly calling one screen:

```kotlin
setContent {
    ResearchScreen()
}
```

you now call:

```kotlin
setContent {
    ResearchApp()
}
```

So the app starts with the navigation system.

A simplified `MainActivity`:

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContent {
            ResearchApp()
        }
    }
}
```

---

## 19. Why screen functions receive callbacks

You may notice that screens do not directly call:

```kotlin
navController.navigate(...)
```

For example, `PatientListScreen` receives:

```kotlin
onPatientClick: (Long) -> Unit
```

rather than directly receiving `navController`.

This is intentional.

A cleaner pattern is:

```text
Screen says what happened.
Parent navigation function decides where to go.
```

For example:

```text
PatientListScreen:
Patient 1 was clicked.

ResearchApp:
Navigate to patient_detail/1.
```

This keeps the screen more reusable and easier to test.

---

## 20. Where should the ViewModel live?

This is an important question.

In our earlier lessons, we had one big screen and one ViewModel.

Now we have multiple screens.

A beginner-friendly approach is:

```text
PatientListScreen
 ↓
PatientListViewModel later

PatientDetailScreen
 ↓
PatientDetailViewModel later

MeasurementScreen
 ↓
MeasurementViewModel later

ResultScreen
 ↓
ResultViewModel later
```

But we do not need to split all ViewModels immediately.

For the next step, we can keep one `ResearchViewModel` while learning navigation.

Later, when screens become more complex, we can split it.

A practical path is:

```text
Lesson 16:
focus on navigation only

Later:
connect each screen to database/ViewModel properly
```

Do not try to solve everything at once.

---

## 21. Passing IDs between screens

The most important thing in this lesson is passing IDs.

From Patient List to Patient Detail:

```text
patientId
```

From Patient Detail to Measurement Screen:

```text
sessionId
```

From Measurement Screen to Result Screen:

```text
sessionId
```

Why IDs?

Because the database already stores relationships using IDs:

```text
PatientEntity.id
 ↓
SessionEntity.patientId

SessionEntity.id
 ↓
MeasurementEntity.sessionId
ResultEntity.sessionId
```

So navigation should pass IDs, not whole objects.

Good pattern:

```kotlin
navController.navigate("measurement/$sessionId")
```

Avoid passing a whole patient or measurement object through navigation.

The next screen can use the ID to load the required data from Room.

---

## 22. Current architecture after Lesson 16

After this lesson, the app structure becomes:

```text
MainActivity
 ↓
ResearchApp
 ↓
NavHost
 ├── PatientListScreen
 ├── PatientDetailScreen
 ├── MeasurementScreen
 └── ResultScreen
```

The data structure is still:

```text
Patient
 ↓
Session
 ↓
Measurement
 ↓
Result
```

The architecture is becoming:

```text
Screen
 ↓
ViewModel
 ↓
Repository
 ↓
Room database
```

But now there is more than one screen.

This is closer to a real Android research app.

---

## 23. What This Teaches You

This lesson is not mainly about making more UI pages.
It teaches this mental model:

- A real app is a flow of screens.
- Each screen has one main purpose.
- Navigation connects the screens.
- IDs connect screens to database records.

For our research app:

```text
Patient List Screen
 ↓
choose who/what to measure

Patient Detail Screen
 ↓
choose or create a session

Measurement Screen
 ↓
collect data for one session

Result Screen
 ↓
show analysis or ML output for that session
```

This is much better than putting everything into one very large screen.

## 24. What You Learned in Lesson 16

The key patterns are:

```kotlin
val navController = rememberNavController()
```

creates the navigation controller.

```kotlin
NavHost(
    navController = navController,
    startDestination = Routes.PATIENT_LIST
) {
    ...
}
```

defines the screen navigation graph.

```kotlin
composable(Routes.PATIENT_LIST) {
    PatientListScreen(...)
}
```

defines one screen destination.

```kotlin
navController.navigate("${Routes.PATIENT_DETAIL}/$patientId")
```

moves to another screen with an ID.

```kotlin
navController.popBackStack()
```

goes back to the previous screen.

The most important research-app idea is:

- Use navigation to move through the research workflow.
- Use IDs to connect screens to database records.

After Lesson 16, the app is no longer just one screen.
It now has the shape of a real research tablet app:

```text
Patient list
 ↓
Patient detail
 ↓
Measurement session
 ↓
Result
```

## Lesson 17 Preview

In Lesson 17, we should prepare for real device communication.
So far, the app still uses simulated data.
Next, we need to understand:

- Android permissions
- Bluetooth permission idea
- Wi-Fi/network permission idea
- runtime permission requests
- device connection flow
- where real device communication code belongs

The goal of Lesson 17 will not be to fully implement Bluetooth yet.
The goal will be to understand the permission and device-communication structure, so later we can replace simulated data with real device input safely.