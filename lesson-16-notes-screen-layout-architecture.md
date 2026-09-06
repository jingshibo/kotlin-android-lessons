# Lesson 16 Notes - Screen Layout Architecture

This note explains the bigger picture of Compose screen layout.

Earlier lessons we learned on smaller layout tools:

```text
Column
Row
Spacer
Card
Button
TextField
Modifier.padding(...)
```

Those tools arrange content inside one part of the screen.

Lesson 16 adds a higher-level question:

```text
How is the whole app screen organized when the app has multiple screens?
```

That is where these ideas become important:

```text
Surface
Scaffold
NavHost
top app bar
bottom navigation
snackbar
toast
dialog
screen callbacks
```

## 1. The Big Picture

Think of a Compose app as layers.

```text
MainActivity
    starts Compose with setContent

Theme
    gives colors, typography, and Material styling

Surface
    gives the app a full-screen visual container/background

ResearchApp
    owns the navigation controller and navigation graph

NavHost
    decides which screen is currently shown

Screen composables
    PatientListScreen, PatientDetailScreen, MeasurementScreen, ResultScreen

Scaffold
    gives a screen its major layout slots

Small layout composables
    Column, Row, Card, Button, Text, TextField
```

A simple mental model:

```text
Surface:
    What background/container is this content sitting on?

Scaffold:
    What major screen areas does this page need?

NavHost:
    Which screen route should be displayed now?

Column/Row/Card:
    How is the content inside one screen arranged?
```

### Two common navigation styles

Multi-screen apps usually use navigation in two common ways.

The first way is **workflow or detail navigation**.

This means the user clicks something specific, and the app opens a screen for that specific record or step.

Example:

```text
Click Patient P001
    -> open PatientDetailScreen for patientId = 1

Click Start Session
    -> open MeasurementScreen for sessionId = 10
```

Code idea:

```kotlin
navController.navigate("${Routes.PATIENT_DETAIL}/$patientId")
```

The route often carries an ID:

```text
patient_detail/1
```

The second way is **top-level section navigation**.

This means the user switches between main app sections, often with bottom navigation.

Example:

```text
Click Patients
    -> open Patients section

Click Settings
    -> open Settings section
```

Code idea:

```kotlin
navController.navigate(Routes.SETTINGS)
```

The route is usually a fixed screen name:

```text
settings
```

Both styles usually use the same Navigation Compose function:

```kotlin
navController.navigate(...)
```

The difference is the route you send.

Workflow/detail navigation often sends a route with an ID.

Top-level section navigation often sends a fixed route name.

In both cases, the target screen still needs to be registered inside `NavHost`.

Simple rule:

```text
Use detail navigation when the user opens a specific record.
Use bottom navigation when the user switches between main app areas.
```

## 2. Where Code Usually Goes

### MainActivity

`MainActivity` starts the app.

It usually contains `setContent`.

Example:

```kotlin
setContent {
    ResearchAppTheme {
        Surface(
            modifier = Modifier.fillMaxSize(),
            color = MaterialTheme.colorScheme.background
        ) {
            ResearchApp()
        }
    }
}
```

Read it as:

```text
Use the app theme.
Draw a full-screen background.
Start the app navigation.
```

### ResearchApp

`ResearchApp` usually owns navigation.

Example:

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
        }
    }
}
```

Read it as:

```text
Create the navigation controller.
Create a Scaffold for app-level screen structure.
Put the NavHost inside the Scaffold content area.
For each route, say which screen to show.
```

`innerPadding` comes from `Scaffold`.

Passing it to `NavHost` helps the displayed screen avoid areas reserved by the `Scaffold`.

### Screen Composables

Each screen composable focuses on one screen.

Example:

```kotlin
@Composable
fun PatientListScreen(
    onPatientClick: (Long) -> Unit
) {
    Column(
        modifier = Modifier.padding(16.dp)
    ) {
        Text("Patients")

        Button(
            onClick = {
                onPatientClick(1L)
            }
        ) {
            Text("Open Patient P001")
        }
    }
}
```

Read it as:

```text
Show this screen's content.
Report user actions through callbacks.
Do not directly own the whole navigation graph.
```

## 3. Surface

`Surface` is a visual container.

It can control:

```text
background color
content color
shape
border
elevation
size
```

For the whole app, `Surface` is useful as a full-screen background:

```kotlin
Surface(
    modifier = Modifier.fillMaxSize(),
    color = MaterialTheme.colorScheme.background
) {
    ResearchApp()
}
```

This means:

```text
Draw a Material background behind the whole app.
Make sure the app has a full-screen container.
```

`Surface` can also be used for a smaller visual area:

```kotlin
Surface(
    tonalElevation = 2.dp
) {
    Text("Session ready")
}
```

Beginner rule:

```text
Use Surface when you need a visual container.
```

## 4. Scaffold

`Scaffold` is a screen structure.

It gives you standard screen slots:

```text
topBar
bottomBar
floatingActionButton
snackbarHost
content area
```

Example:

```kotlin
Scaffold(
    topBar = {
        TopAppBar(
            title = {
                Text("Patients")
            }
        )
    }
) { innerPadding ->
    PatientListScreenContent(
        modifier = Modifier.padding(innerPadding)
    )
}
```

Read it as:

```text
Draw a top app bar.
Give the main content padding so it does not hide behind the app bar.
```

The `innerPadding` is important.

If you ignore it, your content can be drawn underneath the app bar or system bars.

Beginner rule:

```text
Use Scaffold when a screen needs standard screen structure.
```

## 5. Surface Versus Scaffold

They solve different problems.

```text
Surface:
    visual container/background

Scaffold:
    screen layout structure
```

Common beginner setup:

```text
MainActivity:
    Theme
        Surface
            ResearchApp

ResearchApp:
    NavHost

Each screen:
    Column/Row/Card content
```

Later, when screens need app bars or snackbars:

```text
MainActivity:
    Theme
        Surface
            ResearchApp

ResearchApp:
    NavHost

Each screen:
    Scaffold
        topBar
        content
        snackbar
```

Another valid pattern is one shared `Scaffold` around the `NavHost`. 
That is useful when the whole app shares the same top bar or bottom navigation.

## 6. NavHost

`NavHost` is the navigation screen container.

It decides which destination is currently visible.

Example:

```kotlin
NavHost(
    navController = navController,
    startDestination = Routes.PATIENT_LIST
) {
    composable(Routes.PATIENT_LIST) {
        PatientListScreen(...)
    }

    composable(
        route = "${Routes.PATIENT_DETAIL}/{patientId}"
    ) {
        PatientDetailScreen(...)
    }
}
```

Read it as:

```text
If the current route is patient_list:
    show PatientListScreen.

If the current route is patient_detail/{patientId}:
    show PatientDetailScreen.
```

`NavHost` controls which screen is displayed.

It does not arrange every button and text inside that screen.

The individual screen composables do that.

No matter which navigation style you use, the destination screen must be included in `NavHost`.

Think of the roles this way:

```text
NavController:
    sends a route request

NavHost:
    stores the route-to-screen map

composable(...):
    defines one destination in that map
```

Example:

```text
navController.navigate("patient_detail/1")
    -> NavHost finds the matching patient detail route
    -> PatientDetailScreen is displayed

navController.navigate("settings")
    -> NavHost finds the matching settings route
    -> SettingsScreen is displayed
```

## 7. Top App Bar

A top app bar is the bar at the top of a screen.

It often contains:

```text
screen title
back button
menu actions
save/export button
```

Example:

```text
< Back    Patient Details        Edit
```

In Compose, it often lives inside `Scaffold`:

```kotlin
@Composable
fun PatientDetailScreen(
    onBackClick: () -> Unit,
    onEditClick: () -> Unit
) {
    Scaffold(
        topBar = {
            TopAppBar(
                navigationIcon = {
                    IconButton(
                        onClick = onBackClick
                    ) {
                        Icon(
                            imageVector = Icons.AutoMirrored.Filled.ArrowBack,
                            contentDescription = "Back"
                        )
                    }
                },
                title = {
                    Text("Patient Details")
                },
                actions = {
                    TextButton(
                        onClick = onEditClick
                    ) {
                        Text("Edit")
                    }
                }
            )
        }
    ) { innerPadding ->
        PatientDetailContent(
            modifier = Modifier.padding(innerPadding)
        )
    }
}
```

Read the `TopAppBar` parts like this:

```text
navigationIcon:
    the left-side button, usually Back or Menu

title:
    the main title of the screen

actions:
    the right-side actions, such as Edit, Save, Delete, or Export
```

The screen receives `onBackClick` and `onEditClick` as callback parameters.

That keeps the screen reusable. The screen draws the buttons, but the parent decides what those buttons actually do.

Use a top app bar when the screen needs a clear title or top-level actions.

## 8. Bottom Navigation

Bottom navigation is a bar at the bottom of the app.

It is used to switch between major top-level areas.

Example:

```text
Patients | Sessions | Results | Settings
```

Use bottom navigation for sibling sections of the app.

Good use:

```text
Patients
Sessions
Settings
```

Not usually a good use:

```text
Patient List
Patient Detail
Measurement
Result
```

Why?

Because `Patient Detail`, `Measurement`, and `Result` are usually steps inside a workflow, not top-level app areas.

A research workflow often looks like this:

```text
Patient List
    -> Patient Detail
        -> Measurement
            -> Result
```

That kind of flow usually uses normal navigation and back buttons.

Bottom navigation is better for main app sections that the user can switch between freely.

## 9. Floating Action Button

A floating action button, or FAB, is a prominent action button that floats over the content.

It is often used for the main action on a screen.

Examples:

```text
+ Add patient
+ Start session
+ Add measurement
```

In Compose, a FAB usually goes in `Scaffold`:

```kotlin
Scaffold(
    floatingActionButton = {
        FloatingActionButton(
            onClick = {
                // Start the main action.
            }
        ) {
            Text("+")
        }
    }
) { innerPadding ->
    PatientListContent(
        modifier = Modifier.padding(innerPadding)
    )
}
```

Use a FAB when there is one obvious main action for the screen.

## 10. Snackbar

A snackbar is a short temporary message inside the app UI.

Example:

```text
Measurement saved
```

It usually appears near the bottom of the screen and disappears after a short time.

Snackbars are good for:

```text
saved messages
undo messages
non-critical errors
short feedback after an action
```

A snackbar usually belongs to a `Scaffold` because `Scaffold` knows where the snackbar should appear.

Beginner mental model:

```text
Toast:
    quick Android system message

Snackbar:
    Compose/Material message inside your app layout
```

## 11. Toast

`Toast.makeText(...)` shows a short Android system message.

Example:

```kotlin
Toast.makeText(
    context,
    "Measurement saved",
    Toast.LENGTH_SHORT
).show()
```

A toast is simple and useful for quick learning.

But it is outside your Compose layout.

You do not place it inside `Column`, `Scaffold`, or `NavHost`.

You call it as an action:

```text
Button clicked
    -> save data
    -> show Toast
```

For a polished Compose app, snackbar is often preferred because it fits better into Material screen structure.

## 12. Dialog

A dialog is a temporary window over the current screen.

It is useful when the app needs the user to make a decision before continuing.

Examples:

```text
Delete this session?
Discard unsaved changes?
Export complete. Open file?
```

In Compose, you might show an `AlertDialog` based on state:

```kotlin
if (showDeleteDialog) {
    AlertDialog(
        onDismissRequest = {
            showDeleteDialog = false
        },
        title = {
            Text("Delete session?")
        },
        confirmButton = {
            Button(
                onClick = {
                    showDeleteDialog = false
                    // Delete the session.
                }
            ) {
                Text("Delete")
            }
        },
        dismissButton = {
            Button(
                onClick = {
                    showDeleteDialog = false
                }
            ) {
                Text("Cancel")
            }
        }
    )
}
```

A dialog is not a full navigation destination.

It is a temporary overlay on top of the current screen.

## 13. Full Screen Versus Temporary UI

A full screen is a main destination in the app.

Examples:

```text
PatientListScreen
PatientDetailScreen
MeasurementScreen
ResultScreen
SettingsScreen
```

These usually belong in `NavHost`.

Temporary UI is something that appears on top of a screen or briefly inside it.

Examples:

```text
snackbar
toast
dialog
bottom sheet
loading overlay
```

These usually do not get their own main route.

They are usually controlled by state inside the current screen or ViewModel.

Simple rule:

```text
If the user is moving to a new place in the app:
    use navigation and NavHost.

If the user is seeing a temporary message or decision:
    use snackbar, toast, dialog, or another temporary UI pattern.
```

## 14. Two Common App Structures

### Simple Lesson 16 Structure

This is good while learning navigation:

```text
MainActivity
    setContent
        Theme
            Surface
                ResearchApp
                    Scaffold
                        NavHost
                            PatientListScreen
                            PatientDetailScreen
                            MeasurementScreen
                            ResultScreen
```

The individual screens can still use simple `Column` layouts.

Here, `Surface` belongs around the whole app.

`Scaffold` belongs around the `NavHost` because it manages the app screen area.

### Larger App Structure

Later, the app may grow into this:

```text
MainActivity
    setContent
        Theme
            Surface
                ResearchApp
                    Scaffold
                        topBar
                        bottomBar
                        snackbarHost
                        NavHost
                            PatientListScreen
                            PatientDetailScreen
                            MeasurementScreen
                            ResultScreen
```

In this structure, the shared `Scaffold` wraps the `NavHost`.

That makes sense if the top bar, bottom navigation, or snackbar system belongs to the whole app.

Another valid structure is:

```text
ResearchApp
    NavHost
        PatientListScreen
            Scaffold
        PatientDetailScreen
            Scaffold
        MeasurementScreen
            Scaffold
```

That makes sense if each screen has its own different top bar, actions, or snackbar behavior.

## 15. What Controls What?

Use this table as a map:

| Piece | Controls | Usually goes where? |
| --- | --- | --- |
| `setContent` | starts Compose UI | `MainActivity` |
| `Theme` | colors, typography, Material style | around the whole app |
| `Surface` | background/container | around whole app or small areas |
| `NavController` | navigation actions | `ResearchApp` |
| `NavHost` | which screen route is shown | `ResearchApp` |
| `Scaffold` | top bar, bottom bar, FAB, snackbar, content padding | whole app or individual screen |
| `Column` / `Row` | local content layout | inside a screen |
| `Card` | grouped content | inside a screen |
| `Snackbar` | temporary in-app message | usually through `Scaffold` |
| `Toast` | temporary Android system message | called from event code |
| `Dialog` | temporary decision overlay | shown from screen state |

## 16. Beginner Rules

Use these rules for now:

```text
1. Put setContent in MainActivity.

2. Put the app Theme around everything.

3. Use Surface for the whole app background.

4. Put rememberNavController and NavHost in ResearchApp.

5. Put one composable(...) route in NavHost for each full screen.

6. Put Column, Row, Card, Text, and Button inside each screen.

7. Use Scaffold when a screen needs a top bar, bottom bar, snackbar, or FAB.

8. Use bottom navigation only for major top-level sections.

9. Use snackbar or toast for short temporary messages.

10. Use dialog for temporary decisions that need user confirmation.
```

The most important idea:

```text
Small layout components arrange content.
Scaffold organizes a screen.
NavHost organizes multiple screens.
Surface gives content a visual container.
```
