# Lesson 16 - Multi-Screen App Navigation

In Lesson 15, we changed the app from a simple measurement logger into a more realistic research data model:

```text
Patient
 -> Session
 -> Measurement
 -> Result
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

Extra note:

```text
If you want the broader picture of Surface, Scaffold, NavHost,
bottom navigation, snackbar, toast, dialog, and full-screen layout,
read lesson-16-notes-screen-layout-architecture.md.
```

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
 -> Patient Detail Screen
 -> Session Measurement Screen
 -> Result Screen
```

The flow is:

```text
Open app
 -> See all patients/samples
 -> Select one patient/sample
 -> See that patient's sessions
 -> Start or open a session
 -> Collect measurements
 -> View result
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
 -> click patient
Patient Detail
 -> click session
Measurement Screen
 -> click view result
Result Screen
```

In Jetpack Compose apps, Android provides **Navigation Compose** for navigating between composable screens. The official Android documentation says that if an app is built entirely with Jetpack Compose, Navigation Compose is the appropriate navigation option.

The main tools are:

```text
NavController
NavHost
routes
composable destinations
```

Do not worry if these terms are new. We will introduce them one by one.

---

## 4. Two common navigation styles

When you first learn multi-screen apps, it helps to know that navigation usually appears in two common styles.

The first style is **workflow or detail navigation**.

This happens when the user clicks something specific and the app opens a screen for that specific thing.

For example:

```text
Click Patient P001
 -> open PatientDetailScreen for patientId = 1

Click Start Session
 -> open MeasurementScreen for sessionId = 10
```

This kind of navigation often passes an ID in the route:

```kotlin
navController.navigate("${Routes.PATIENT_DETAIL}/$patientId")
```

The second style is **top-level section navigation**.

This happens when the user switches between main app areas, often using a bottom navigation bar.

For example:

```text
Click Patients
 -> open the Patients section

Click Settings
 -> open the Settings section
```

This kind of navigation often uses a fixed route:

```kotlin
navController.navigate(Routes.SETTINGS)
```

Both styles still use the same basic Navigation Compose idea:

```text
navController.navigate(...)
    sends a route request

NavHost
    finds the matching route and displays the target screen
```

In this lesson, we mostly build the first style: moving from a list, to a detail screen, to measurement, to result.

The broader layout note explains how bottom navigation fits into a larger app structure.

---

## 5. Basic mental model

Think of navigation like this:

```text
NavController
 -> the object that controls the navigation between screens

NavHost
 -> the container that displays the current screen

Route
 -> the name/address of the screen to go
```

For example:

```text
Route: patient_list
 -> show PatientListScreen

Route: patient_detail/3
 -> show PatientDetailScreen for patient ID 3

Route: measurement/10
 -> show MeasurementScreen for session ID 10
```

The Android documentation describes `rememberNavController()` as the Compose way to create a `NavController`, and `NavHost` as the composable that defines the navigation graph.

---

## 6. Add Navigation dependency

To use Navigation Compose, your app needs the Navigation Compose dependency.

In `build.gradle.kts`, you will commonly need something like:

```kotlin
implementation("androidx.navigation:navigation-compose:<latest-version>")
```

The exact version may change, so in a real project you should check Android Studio's suggestions or the official AndroidX Navigation release notes. The important idea is:

```text
Navigation Compose allows the app to move between composable screens.
```

For this lesson, focus on the app structure first.

---

## 7. Create screen routes

For beginner learning, we can start with simple string routes.

Each full screen should have a route name.

A route is the screen's address.

For example:

```text
patient_list:
    address for the Patient List screen

patient_detail:
    base address for the Patient Detail screen

settings:
    address for the Settings screen
```

Create one `Routes` object to keep these screen addresses in one place:

```kotlin
object Routes {
    const val PATIENT_LIST = "patient_list"
    const val PATIENT_DETAIL = "patient_detail"
    const val MEASUREMENT = "measurement"
    const val RESULT = "result"
    const val ADD_PATIENT = "add_patient"
    const val SETTINGS = "settings"
}
```

This avoids writing raw strings everywhere.

Think of `Routes` as the app's screen address book:

```text
PATIENT_LIST:
    PatientListScreen

PATIENT_DETAIL:
    PatientDetailScreen

MEASUREMENT:
    MeasurementScreen

RESULT:
    ResultScreen

ADD_PATIENT:
    AddPatientScreen

SETTINGS:
    SettingsScreen
```

Instead of:

```kotlin
navController.navigate("patient_list")
```

we can write:

```kotlin
navController.navigate(Routes.PATIENT_LIST)
```

This is safer because the route names are stored in one place.

Later, we can move toward modern type-safe routes. Android's current navigation guidance supports type-safe routes using serializable objects or classes, which can reduce runtime mistakes from route typos or wrong argument types.
But for this lesson, string routes are easier for understanding the basic idea.

---

## 8. Create the app navigation host

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

Important placement idea:

```text
ResearchApp() contains one NavHost.
The NavHost contains one composable(...) block for each screen route.
```

As the lesson adds more screens, you keep adding more `composable(...)` blocks inside the same `NavHost`:

```text
NavHost(...) {
    composable(Routes.PATIENT_LIST) {
        ...
    }

    composable(patient detail route) {
        ...
    }

    composable(measurement route) {
        ...
    }
}
```

Let us break it down.

---

## 9. `rememberNavController()`

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

The same `navController.navigate(...)` function can be used for different navigation styles.

For a predefined screen:

```kotlin
navController.navigate(Routes.PATIENT_LIST)
```

This means:

```text
Go to the patient list screen.
```

For a specific detail screen:

```kotlin
navController.navigate("${Routes.PATIENT_DETAIL}/$patientId")
```

This means:

```text
Go to the patient detail screen for this specific patient.
```

So the controller is the same:

```text
navController.navigate(...)
```

The route input is different:

```text
patient_list
    predefined screen

patient_detail/1
    detail screen with an ID
```

---

## 10. `NavHost`

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

No matter which navigation style you use, the target screen must be registered inside `NavHost`.

Think of it like this:

```text
NavController:
    sends a route request

NavHost:
    owns the map of routes to screens

composable(...):
    one route entry in that map
```

For example:

```text
Detail navigation:
    navController.navigate("patient_detail/1")
    NavHost matches patient_detail/{patientId}
    PatientDetailScreen is shown

Bottom navigation:
    navController.navigate("settings")
    NavHost matches settings
    SettingsScreen is shown
```

So `NavHost` is not only for one navigation style.

It is the place where Navigation Compose knows what screens exist, and which route opens each screen.

---

## 11. First screen: Patient List Screen

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

This callback is important, so slow down and read it carefully.

### What `onPatientClick` Is

```kotlin
fun PatientListScreen(
    onPatientClick: (Long) -> Unit
)
```

`onPatientClick` is a custom callback parameter.

It is not from Android.

It is not from Navigation Compose.

It is a name we choose when we design `PatientListScreen`.

It means:

```text
PatientListScreen does not know how to navigate.
PatientListScreen needs someone else to tell it what to do when a patient is clicked.
```

The type explains the shape of the callback:

```text
(Long) -> Unit
```

Read it as:

```text
Input:
    one Long value, the patient ID

Output:
    Unit, meaning it does not return a value
```

So this parameter means:

```text
Give PatientListScreen a function.
That function must accept one patient ID.
PatientListScreen will call it when a patient is clicked.
```

### Where `onPatientClick` Comes From

The name is created here:

```kotlin
fun PatientListScreen(
    onPatientClick: (Long) -> Unit
)
```

At this point, `PatientListScreen` only creates a slot:

```text
I need a function named onPatientClick.
I know what kind of function it must be.
But I do not know what it will actually do.
```

The real behavior is supplied later, when `ResearchApp` calls `PatientListScreen`:

```kotlin
PatientListScreen(
    onPatientClick = { patientId ->
        navController.navigate(
            "${Routes.PATIENT_DETAIL}/$patientId"
        )
    }
)
```

This part is the body of `onPatientClick`:

```kotlin
{ patientId ->
    navController.navigate(
        "${Routes.PATIENT_DETAIL}/$patientId"
    )
}
```

Read each part:

```text
patientId ->
    The callback receives a patient ID.
    The ID will come from PatientListScreen.

navController.navigate(...)
    Tell Navigation Compose to move to another screen.

"${Routes.PATIENT_DETAIL}/$patientId"
    Build the destination route using the selected patient ID.
```

For example, if `patientId` is `1`, the route becomes:

```text
patient_detail/1
```

### How `onPatientClick` Is Used

Inside `PatientListScreen`, the buttons call the callback.

For Patient P001:

```kotlin
onPatientClick(1L)
```

This means:

```text
The user clicked Patient P001.
Send patient ID 1 to the outside callback.
```

For Patient P002:

```kotlin
onPatientClick(2L)
```

This means:

```text
The user clicked Patient P002.
Send patient ID 2 to the outside callback.
```

So the flow is:

```text
ResearchApp gives PatientListScreen a callback function called onPatientClick.

1. The user clicks Patient P001.

2. PatientListScreen calls onPatientClick(1L).
       This means: Patient 1 was clicked.

3. Calling onPatientClick(1L) runs the lambda that ResearchApp supplied.

4. Inside that lambda, patientId becomes 1L.

5. The lambda uses navController.navigate(...).
       The app navigates to patient_detail/1.

Same idea for Patient P002:
    PatientListScreen calls onPatientClick(2L).
    The supplied lambda runs with patientId = 2L.
    The app navigates to patient_detail/2.
```

Simple mental model:

```text
onPatientClick parameter:
    Tell me what to do when a patient is clicked.

onPatientClick(1L):
    Patient 1 was clicked.

onPatientClick(2L):
    Patient 2 was clicked.
```

---

## 12. Add patient detail route with argument

When it navigates to the Patient Detail Screen, it needs to know:

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

Remember, `Routes.PATIENT_DETAIL` stores the base route name:

```kotlin
const val PATIENT_DETAIL = "patient_detail"
```

For a detail screen, we combine that base route with an argument:

```kotlin
"${Routes.PATIENT_DETAIL}/{patientId}"
```

Then inside `NavHost`, add:

```text
ResearchApp()
    remember navController
    NavHost(...) {
        composable(Routes.PATIENT_LIST) {
            ...
        }

        put the patient detail composable route here
    }
```

Here "inside `NavHost`" means:

It goes inside the navigation graph, where all `composable(...)` route definitions live.

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

Here, `backStackEntry` means:

```text
the navigation record for the screen that is currently being opened
```

When `NavHost` matches this route:

```text
patient_detail/{patientId}
```

with an actual route like this:

```text
patient_detail/1
```

it stores the route information in `backStackEntry`.

So `backStackEntry.arguments` contains:

```text
patientId = "1"
```

Then the screen can read that value and use it.

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

One `backStackEntry` can also contain more than one route argument.

For example:

```kotlin
composable(
    route = "measurement/{patientId}/{sessionId}"
) { backStackEntry ->
    val patientId = backStackEntry.arguments
        ?.getString("patientId")
        ?.toLongOrNull()

    val sessionId = backStackEntry.arguments
        ?.getString("sessionId")
        ?.toLongOrNull()
}
```

If the app navigates to:

```kotlin
navController.navigate("measurement/1/10")
```

then `backStackEntry.arguments` can contain:

```text
patientId = "1"
sessionId = "10"
```

Important wording:

```text
The values are not returned to backStackEntry.

Instead:
    navController.navigate(...) sends route values
    NavHost matches the route pattern
    backStackEntry stores the values for the current destination
    the destination reads them from backStackEntry.arguments
```

---

## 13. Patient Detail Screen

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
 -> get generated sessionId
 -> navigate to MeasurementScreen(sessionId)
```

---

## 14. Add Measurement Screen route

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

## 15. Measurement Screen

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

## 16. Add Result Screen route

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

## 17. Result Screen

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

## 18. Full navigation structure

Now the navigation structure looks like this:

This version also adds a simple `Scaffold` around the `NavHost`.

```kotlin
@Composable
fun ResearchApp() {
    val navController = rememberNavController()

    Scaffold(
        modifier = Modifier.fillMaxSize()
    ) { innerPadding ->
        NavHost(
            navController = navController,
            startDestination = Routes.PATIENT_LIST,
            modifier = Modifier.padding(innerPadding)
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
}
```

Here the `Scaffold` sits around the `NavHost`.

That gives the app a place for screen-level structure such as:

```text
top bar
bottom bar
floating action button
snackbar
content padding
```

For now, the `Scaffold` does not show a top bar or bottom bar yet.

But it already gives the `NavHost` the `innerPadding` value:

```kotlin
modifier = Modifier.padding(innerPadding)
```

That means the screen content is drawn inside the content area managed by `Scaffold`.

This is the core of Lesson 16.

The app now has multiple screens.

---

## 19. One Screen Can Open Different Screens

The examples above move in a simple chain:

```text
Patient List
 -> Patient Detail
 -> Measurement
 -> Result
```

But a real screen often has several buttons, and different buttons can open different screens.

For example, the Patient List screen might have buttons for:

```text
open one patient's detail screen
add a new patient
open settings
```

The rule is:

```text
Each target screen needs a route.
Each button calls a callback.
ResearchApp decides which route that callback opens.
NavHost contains the matching composable(...) destination.
```

In Section 7, we already gave each full screen a route name in `Routes`.

Now we can use those route names from different buttons.

Then the Patient List screen can receive several callbacks:

```kotlin
@Composable
fun PatientListScreen(
    onPatientClick: (Long) -> Unit,
    onAddPatientClick: () -> Unit,
    onSettingsClick: () -> Unit
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
            onClick = onAddPatientClick
        ) {
            Text("Add Patient")
        }

        Button(
            onClick = onSettingsClick
        ) {
            Text("Settings")
        }
    }
}
```

Inside the `NavHost` of `ResearchApp`, the Patient List route decides where each callback goes:

```kotlin
composable(Routes.PATIENT_LIST) {
    PatientListScreen(
        onPatientClick = { patientId ->
            navController.navigate(
                "${Routes.PATIENT_DETAIL}/$patientId"
            )
        },
        onAddPatientClick = {
            navController.navigate(Routes.ADD_PATIENT)
        },
        onSettingsClick = {
            navController.navigate(Routes.SETTINGS)
        }
    )
}
```

Because these callbacks navigate to `Routes.ADD_PATIENT` and `Routes.SETTINGS`, the same `NavHost` also needs matching destination routes for those screens:

```kotlin
composable(Routes.ADD_PATIENT) {
    AddPatientScreen(
        onSavePatientClick = { patientCode ->
            // Later: viewModel.addPatient(patientCode)
            navController.popBackStack()
        },
        onBackClick = {
            navController.popBackStack()
        }
    )
}

composable(Routes.SETTINGS) {
    SettingsScreen(
        onBackClick = {
            navController.popBackStack()
        }
    )
}
```

Here are two simple destination screens:

```kotlin
@Composable
fun AddPatientScreen(
    onSavePatientClick: (String) -> Unit,
    onBackClick: () -> Unit
) {
    Column(
        modifier = Modifier.padding(16.dp)
    ) {
        Text("Add Patient")

        Spacer(modifier = Modifier.height(16.dp))

        Button(
            onClick = {
                onSavePatientClick("PAT_1003")
            }
        ) {
            Text("Save Patient")
        }

        Button(
            onClick = onBackClick
        ) {
            Text("Cancel")
        }
    }
}

@Composable
fun SettingsScreen(
    onBackClick: () -> Unit
) {
    Column(
        modifier = Modifier.padding(16.dp)
    ) {
        Text("Settings")

        Spacer(modifier = Modifier.height(16.dp))

        Button(
            onClick = onBackClick
        ) {
            Text("Back")
        }
    }
}
```

The important idea is:

```text
PatientListScreen does not decide the route directly.
It only reports what the user clicked.

ResearchApp receives that click event.
Then ResearchApp uses navController.navigate(...) to open the correct route.
```

---

## 20. Use `ResearchApp()` in `MainActivity`

In `MainActivity.kt`, instead of directly calling one screen:

```kotlin
setContent {
    ResearchScreen()
}
```

you now call `ResearchApp()` inside a full-screen `Surface`:

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContent {
            MaterialTheme {
                Surface(
                    modifier = Modifier
                        .fillMaxSize()
                        .systemBarsPadding(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    ResearchApp()
                }
            }
        }
    }
}
```

Read it as:

```text
MaterialTheme:
    use Material colors and typography

Surface:
    draw a full-screen app background

systemBarsPadding():
    keep the app content away from the status bar and navigation bar

ResearchApp():
    start the navigation system
```

So the outer structure is:

```text
MainActivity
    MaterialTheme
        Surface
            ResearchApp
                Scaffold
                    NavHost
                        current screen
```

So the app starts with the navigation system inside a proper screen container.

---

## 21. If `ResearchApp()` Looks Blank or Clipped

When you first test `ResearchApp()`, you might wonder:

```text
Why does it look blank?
Why is the top text hidden?
Why does the screen not seem to fill the app window?
```

Sometimes the navigation code is correct, but the screen is being drawn too close to the system bars.

On modern Android, apps often draw edge-to-edge.

That means your Compose content can start at the very top of the screen, behind the status bar or camera cutout area.

So this first text:

```kotlin
Text("Patients")
```

may be drawn at the top edge and become hidden or clipped.

A safer beginner structure is to use both `Surface` and `Scaffold`:

```text
Surface:
    gives the whole app a full-screen background and safe outer padding

Scaffold:
    gives the app screen structure and passes innerPadding to NavHost

NavHost:
    decides which screen is currently visible
```

The important beginner rule is:

```text
NavHost decides which screen to show.
Surface/Scaffold/padding decide where that screen is drawn.
```

Required imports:

```kotlin
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.systemBarsPadding
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Surface
import androidx.compose.ui.Modifier
```

---

## 22. Why screen functions receive callbacks

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

## 23. Where should the ViewModel live?

This is an important question.

In our earlier lessons, we had one big screen and one ViewModel.

Now we have multiple screens.

A beginner-friendly approach is:

```text
PatientListScreen
 -> PatientListViewModel later

PatientDetailScreen
 -> PatientDetailViewModel later

MeasurementScreen
 -> MeasurementViewModel later

ResultScreen
 -> ResultViewModel later
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

## 24. Passing IDs between screens

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
 -> SessionEntity.patientId

SessionEntity.id
 -> MeasurementEntity.sessionId
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

## 25. Current architecture after Lesson 16

After this lesson, the app structure becomes:

```text
MainActivity
 -> MaterialTheme
 -> Surface
 -> ResearchApp
 -> Scaffold
 -> NavHost
    -> PatientListScreen
    -> PatientDetailScreen
    -> MeasurementScreen
    -> ResultScreen
```

The data structure is still:

```text
Patient
 -> Session
 -> Measurement
 -> Result
```

The architecture is becoming:

```text
Screen
 -> ViewModel
 -> Repository
 -> Room database
```

But now there is more than one screen.

This is closer to a real Android research app.

---

## 26. What This Teaches You

This lesson is not mainly about making more UI pages.
It teaches this mental model:

- A real app is a flow of screens.
- Each screen has one main purpose.
- Navigation connects the screens.
- IDs connect screens to database records.

For our research app:

```text
Patient List Screen
 -> choose who/what to measure

Patient Detail Screen
 -> choose or create a session

Measurement Screen
 -> collect data for one session

Result Screen
 -> show analysis or ML output for that session
```

This is much better than putting everything into one very large screen.

## 27. What You Learned in Lesson 16

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
 -> Patient detail
 -> Measurement session
 -> Result
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


