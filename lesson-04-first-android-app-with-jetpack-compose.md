# Lesson 4 — Your First Android App with Jetpack Compose

In Lesson 1–3, we learned enough Kotlin grammar to start building a real Android screen. Now we move from Kotlin console code to Android app code.

The goal of this lesson is to understand this chain:

```text
Android app
    ↓
MainActivity
    ↓
onCreate()
    ↓
setContent { ... }
    ↓
@Composable functions
    ↓
Text / Button / TextField / Column
    ↓
state changes update the UI
```

Jetpack Compose is Android’s modern UI toolkit. In a Compose project, you write the UI directly in Kotlin instead of using XML layouts. Google’s current Compose setup documentation also notes that Kotlin is the language used for Compose projects.

## 1. Create a new Android project

In Android Studio:

```text
New Project
    → Phone and Tablet
    → Empty Activity
    → Language: Kotlin
Use something like:
Name: ResearchApp
Package name: com.example.researchapp
Minimum SDK: API 26 or higher is fine for learning
```

The exact Android Studio screen may look slightly different depending on version, but the important point is to choose an Empty Activity Compose-style project. Google’s first-app codelab uses the Empty Activity template and then edits MainActivity.kt.

## 2. The most important file: MainActivity.kt

After creating the project, find:

```text
app
 └── src
```

```text
     └── main
         └── java
             └── com.example.researchapp
                 └── MainActivity.kt
```

This file is the starting point of your app.

A simplified version looks like this:

```kotlin
class MainActivity : ComponentActivity() {
    // MainActivity inherits from ComponentActivity, which provides the base logic for a modern Android screen.

    // onCreate() is the first method called when the Activity is created.
    // This is where you initialize your UI and setup the app's state.
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // setContent is where you start the Compose UI.
        // Inside the curly braces { }, you define your UI using composable functions.
        setContent {
            // UI goes here
        }
    }
}
```

**Key concept**: MainActivity is the first screen launched when the app starts. In modern Android Compose apps, `setContent { ... }` is where you define the UI using composable functions rather than XML layout files.

**Why not XML?** Compose allows you to write UI as Kotlin code directly, making it more flexible, type-safe, and easier to reason about compared to separate XML layout files.

## 3. What is @Composable?

A composable function is a Kotlin function that describes part of the UI.

For example:

```kotlin
@Composable
fun Greeting() {
    Text("Hello Android")
}
```

The annotation:

```kotlin
@Composable
```

means:

This function can be used by Jetpack Compose to create UI.

So this is not a normal calculation function like:

```kotlin
fun calculateMean(a: Double, b: Double): Double {
    return (a + b) / 2
}
```

It is a UI function.

Google’s Compose tutorial describes composable functions as Kotlin functions marked with @Composable, used by Compose to generate UI.
### Composables Should Accept a `modifier` Parameter

A composable function should accept a `modifier` parameter so the parent can customize it:

```kotlin
@Composable
fun Greeting(
    name: String,
    modifier: Modifier = Modifier
) {
    Text(
        text = "Hello $name",
        modifier = modifier
    )
}
```

This allows the parent to apply modifiers when calling the composable:

```kotlin
Greeting(
    name = "Android",
    modifier = Modifier.padding(16.dp)
)
```

**Good practice**: Composables that return UI should accept a `modifier: Modifier = Modifier` parameter and apply it to their top-level UI element.

### Small, Focused Composables Are Easier to Reason About

Instead of one large `ResearchScreen` function with all the UI, it is better to break it into smaller composables:

```kotlin
@Composable
fun ResearchScreen() {
    Column {
        HeaderSection()
        MeasurementCard()
        ControlButtons()
    }
}

@Composable
fun HeaderSection(modifier: Modifier = Modifier) {
    Text("Research Measurement App", modifier = modifier)
}

@Composable
fun MeasurementCard(modifier: Modifier = Modifier) {
    Card(modifier = modifier) {
        Text("Current value: 2.43")
    }
}

@Composable
fun ControlButtons(modifier: Modifier = Modifier) {
    Row(modifier = modifier) {
        Button(onClick = {}) { Text("Start") }
        Button(onClick = {}) { Text("Reset") }
    }
}
```

**Benefits**:
- Each composable has a focused job
- Easier to read and understand
- Easier to preview and test in isolation
- Easier to reuse in other screens

## 4. Your first simple screen

Inside setContent, you can call a composable function:

```kotlin
setContent {
    ResearchScreen()
}
```

Then define:

```kotlin
@Composable
fun ResearchScreen() {
    Text("Research Measurement App")
}
```

This would show text on the screen.

## 5. A complete first version

Your MainActivity.kt can look like this:

```kotlin
package com.example.researchapp

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable

class MainActivity : ComponentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContent {
            ResearchScreen()
        }
    }
}

@Composable
fun ResearchScreen() {
    Text("Research Measurement App")
}
```

One important detail:

```kotlin
package com.example.researchapp
```

must match your own project package name. Android Studio normally creates this automatically, so if your package name is different, keep the one Android Studio generated.

## 6. Layout: Column

A real screen needs more than one line of text.

In Compose, a Column places UI elements vertically.

```kotlin
@Composable
fun ResearchScreen() {
    Column {
        Text("Research Measurement App")
        Text("Sample ID: S001")
        Text("Current value: 2.43")
    }
}
```

Conceptually:

```text
Research Measurement App
Sample ID: S001
Current value: 2.43
```

For horizontal layout, you use:

Row { ... }

For vertical layout, you use:

Column { ... }

Compose provides layout components such as Column and Row for arranging UI elements.

## 7. Add spacing with Modifier

A Modifier changes layout, size, padding, alignment, and other visual behaviour.

Example:

```kotlin
Column(
    modifier = Modifier.padding(16.dp)
) {
    Text("Research Measurement App")
    Text("Sample ID: S001")
}
```

This means:

Put 16 dp of padding around the column.

You need these imports:

```kotlin
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.padding
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
```

dp means density-independent pixels. In normal terms, it is Android’s standard layout unit.

## 8. Add a button

A button looks like this:

```kotlin
Button(
    onClick = {
        println("Button clicked")
    }
) {
    Text("Start Measurement")
}
```

**Understanding the structure (Trailing Lambda Syntax):**

The curly braces `{ }` after a function call are a Kotlin feature called a **trailing lambda**. When a function's last parameter is a lambda (a code block), you can move it outside the parentheses for better readability.

So this:

```kotlin
Button(
    onClick = { println("Button clicked") },
    content = { Text("Start Measurement") }
)
```

can be written as:

```kotlin
Button(
    onClick = { println("Button clicked") }
) {
    Text("Start Measurement")
}
```

**What it means:**

- `onClick = { ... }`: When the user clicks the button, run the code inside this lambda (code block)
- `{ Text("Start Measurement") }`: The content of the button (what's displayed inside)

**User feedback with Toast:**

For better user feedback, you can show a popup message using `Toast`:

```kotlin
import android.widget.Toast
import androidx.compose.ui.platform.LocalContext

@Composable
fun ResearchScreen() {
    val context = LocalContext.current
    
    Button(
        onClick = {
            // Show a popup message to the user
            Toast.makeText(context, "Measurement Started!", Toast.LENGTH_SHORT).show()
        }
    ) {
        Text("Start Measurement")
    }
}
```

This displays a brief notification at the bottom of the screen, giving the user immediate feedback that their action was registered.


means:

When the user clicks the button, run the code inside onClick.
Display the text “Start Measurement” inside the button.

## 9. Add state

Now we need the screen to change when the button is clicked.

In Compose, the UI is driven by state. When the state changes, Compose updates the screen. Google’s Compose state codelab describes how state determines what is displayed and how Compose keeps the UI updated when state changes.

For example:

```kotlin
var measurementValue by remember {
    mutableStateOf(0.0)
}
```

This creates a value remembered by the UI.

When you change:

```kotlin
measurementValue = 2.43
```

the displayed UI can update automatically.

You need these imports:

```kotlin
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
```

The syntax:

```kotlin
var measurementValue by remember { mutableStateOf(0.0) }
```

may look a bit advanced. For now, just understand the practical meaning:

This is a UI variable. When it changes, the screen updates.

## 10. A small interactive research screen

Now let’s build something useful.

The screen will have:

Research Measurement

Sample ID: S001

Current value:

0.0

[ Start Measurement ]

When the button is clicked, we generate a fake measurement value.

```kotlin
package com.example.researchapp

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Button
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import kotlin.random.Random

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

    var measurementValue by remember {
        mutableStateOf(0.0)
    }

    Column(
        modifier = Modifier.padding(16.dp)
    ) {
        Text("Research Measurement App")

        Spacer(modifier = Modifier.height(16.dp))

        Text("Sample ID: S001")

        Spacer(modifier = Modifier.height(16.dp))

        Text("Current value:")
        Text("$measurementValue")

        Spacer(modifier = Modifier.height(16.dp))

        Button(
            onClick = {
                measurementValue = Random.nextDouble(
                    from = 0.0,
                    until = 5.0
                )
            }
        ) {
            Text("Start Measurement")
        }
    }
}
```

This is already a real Android app screen.

It does not read from a real sensor yet, but the structure is similar:

```text
Button click
    ↓
obtain value
    ↓
store value in state
    ↓
UI updates
```

Later, instead of:

```kotlin
Random.nextDouble(0.0, 5.0)
```

you will use something like:

```text
readFromDevice()
```

or:

```text
runModelInference()
```

## 11. Add a text input field

Now let’s make sampleId editable.

Add TextField.

```kotlin
import androidx.compose.material3.TextField
```

Then:

```kotlin
var sampleId by remember {
    mutableStateOf("")
}
```

And:

```kotlin
TextField(
    value = sampleId,
    onValueChange = {
        sampleId = it
    },
    label = {
        Text("Sample ID")
    }
)
```

**Breaking it down:**

`value = sampleId`

The text field displays the current sampleId value.

`onValueChange = { sampleId = it }`

When the user types, Compose calls this lambda with the new text. The keyword `it` refers to that new text, so we update the state: `sampleId = it`.

**Key concept**: This creates two-way binding: the state controls what is displayed, and user input updates the state.

## 12. Research screen with editable sample ID

Now the code becomes:

```kotlin
package com.example.researchapp

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Button
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.material3.TextField
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import kotlin.random.Random

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

    var measurementValue by remember {
        mutableStateOf<Double?>(null)
    }

    Column(
        modifier = Modifier.padding(16.dp)
    ) {
        Text("Research Measurement App")

        Spacer(modifier = Modifier.height(16.dp))

        TextField(
            value = sampleId,
            onValueChange = {
                sampleId = it
            },
            label = {
                Text("Sample ID")
            }
        )

        Spacer(modifier = Modifier.height(16.dp))

        Text("Current value:")

        Text(
            text = measurementValue?.toString() ?: "No measurement yet"
        )

        Spacer(modifier = Modifier.height(16.dp))

        Button(
            onClick = {
                measurementValue = Random.nextDouble(
                    from = 0.0,
                    until = 5.0
                )
            }
        ) {
            Text("Start Measurement")
        }
    }
}
```

Now notice something from Lesson 3:

```kotlin
var measurementValue by remember {
    mutableStateOf<Double?>(null)
}
```

The measurement value is nullable:

Double?

because before the first measurement, there is no value yet.

Then we display it safely:

measurementValue?.toString() ?: "No measurement yet"

That is exactly the same null-safety concept from Lesson 3.

## 13. Why this is important

This simple app already combines the main ideas you need:

```text
Kotlin variable
    ↓
Compose state
    ↓
UI display
    ↓
button event
    ↓
state update
    ↓
UI refresh
```

In research-app terms:

```text
sampleId = user input
measurementValue = latest sensor value
Button = start acquisition
Text = live display
```

This is the basic pattern behind many practical Android research tools.

## 14. Add a measurement counter

Now let’s count how many measurements have been taken.

Add:

```kotlin
var measurementCount by remember {
    mutableStateOf(0)
}
```

Inside the button click:

```text
measurementCount = measurementCount + 1
```

Display it:

```kotlin
Text("Measurements: $measurementCount")
```

Full relevant part:

```kotlin
var measurementCount by remember {
    mutableStateOf(0)
}

Button(
    onClick = {
        measurementValue = Random.nextDouble(0.0, 5.0)
        measurementCount = measurementCount + 1
    }
) {
    Text("Start Measurement")
}

Text("Measurements: $measurementCount")
```

This is similar to your research protocol idea:

```text
Sample ID
Repetition count
Measurement value
```

## 15. Improved full version

Here is the improved first research app screen:

```kotlin
package com.example.researchapp

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Button
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.material3.TextField
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import kotlin.random.Random

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

    var measurementValue by remember {
        mutableStateOf<Double?>(null)
    }

    var measurementCount by remember {
        mutableStateOf(0)
    }

    Column(
        modifier = Modifier.padding(16.dp)
    ) {
        Text("Research Measurement App")

        Spacer(modifier = Modifier.height(16.dp))

        TextField(
            value = sampleId,
            onValueChange = {
                sampleId = it
            },
            label = {
                Text("Sample ID")
            }
        )

        Spacer(modifier = Modifier.height(16.dp))

        Text("Current value:")

        Text(
            text = measurementValue?.toString() ?: "No measurement yet"
        )

        Spacer(modifier = Modifier.height(16.dp))

        Text("Measurements: $measurementCount")

        Spacer(modifier = Modifier.height(16.dp))

        Button(
            onClick = {
                measurementValue = Random.nextDouble(
                    from = 0.0,
                    until = 5.0
                )

                measurementCount = measurementCount + 1
            }
        ) {
            Text("Start Measurement")
        }
    }
}
```

At this stage, this app can:

- accept a sample ID

- display latest measurement

- simulate a new reading

- count measurements

- update UI after button click

That is already a very useful first milestone.

## 16. Android Studio Preview

You can also preview a composable without running the full app on an emulator.

Add this import:

```kotlin
import androidx.compose.ui.tooling.preview.Preview
```

Then add:

@Preview(showBackground = true)

```kotlin
@Composable
fun ResearchScreenPreview() {
    MaterialTheme {
        ResearchScreen()
    }
}
```

Android Studio can show this screen in the design preview. Google’s Compose tutorial describes @Preview as a way to preview composable functions in Android Studio without building and installing the app on a device or emulator.

## 17. One important warning

For now, this is okay:

```kotlin
Button(
    onClick = {
        measurementValue = Random.nextDouble(0.0, 5.0)
    }
)
```

But later, when reading from a real device, do not do long blocking work directly inside onClick.

For example, avoid this kind of idea:

```kotlin
Button(
    onClick = {
        while (true) {
            // read sensor forever
        }
    }
)
```

That can freeze the app UI.

Later, we will use:

```text
coroutines
background tasks
ViewModel
state updates
```

But not yet. For Lesson 4, focus only on the UI-state pattern.

## 18. Key mental model

Compose is different from traditional UI programming.

You do not usually say:

```text
Find this text view
Change its text manually
Refresh the screen
```

Instead, you say:

```text
Here is the state
Here is how the UI should look for that state
When state changes, Compose redraws the necessary UI
```

For example:

```kotlin
var measurementValue by remember {
    mutableStateOf<Double?>(null)
}
```

This is the state.

```kotlin
Text(
    text = measurementValue?.toString() ?: "No measurement yet"
)
```

This is how the UI displays the state.

```kotlin
measurementValue = Random.nextDouble(0.0, 5.0)
```

This changes the state.

Then the UI updates.

## What you need to remember from Lesson 4

These are the essential patterns:

Column {

```kotlin
class MainActivity : ComponentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContent {
            ResearchScreen()
        }
    }
}
@Composable
fun ResearchScreen() {
    Text("Hello")
}
    Text("Title")
    Text("Value")
}
Button(
    onClick = {
        println("Clicked")
    }
) {
    Text("Start")
}
var value by remember {
    mutableStateOf(0)
}
Text("Value = $value")
TextField(
    value = sampleId,
    onValueChange = {
        sampleId = it
    },
    label = {
        Text("Sample ID")
    }
)
```

If these make sense, you understand the core of a simple Compose app.

## Small exercise

Try modifying the screen so it also shows:

Device status: Connected

Then add another button:

Reset

When clicked, it should set:

```text
measurementValue = null
measurementCount = 0
```

The key code will look like:

```kotlin
Button(
    onClick = {
        measurementValue = null
        measurementCount = 0
    }
) {
    Text("Reset")
}
```

## Lesson 5 preview

Next, I suggest we cover:

## Compose layout and UI components in more detail

including:

```text
Column
Row
Spacer
Modifier
padding
fillMaxWidth
font size
Button
TextField
Card
simple screen styling
```

That will let us make the research app look more like a usable tablet interface rather than a very plain demo.

- Top of Form
- lesson
- Bottom of Form
- Android Basics with Compose course  |  Android Developers
- Quick start  |  Jetpack Compose  |  Android Developers
- Jetpack Compose basics  |  Android Developers
