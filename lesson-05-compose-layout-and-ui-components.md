# Lesson 5 Compose Layout and UI Components

In Lesson 4, we built a very plain Android screen:

Research Measurement App

- Sample ID
- [             ]

Current value:

No measurement yet

Measurements: 0

[ Start Measurement ]

In Lesson 5, we improve the layout so it becomes closer to a usable tablet research app interface.

We will cover:

```text
Column
Row
Spacer
Modifier
padding
fillMaxWidth
Arrangement
Alignment
font size
Card
OutlinedTextField
Button layout
basic styling
```

## 1. Why layout matters

For a research app, the UI should be clear and difficult to misuse.

For example, you probably want the user to quickly see:

- which sample is being measured

- whether the device is connected

- the latest reading

- how many repetitions have been recorded

- which button starts/stops/resets the measurement

So instead of only learning generic UI examples, we will keep using a research-data-collection screen.

## 2. The basic Column

A Column arranges items vertically.

Column {

```kotlin
    Text("Research Measurement App")
    Text("Sample ID: S001")
    Text("Current value: 2.43")
}
```

This displays:

```text
Research Measurement App
Sample ID: S001
Current value: 2.43
```

A better version adds padding:

```kotlin
Column(
    modifier = Modifier.padding(16.dp)
) {
    Text("Research Measurement App")
    Text("Sample ID: S001")
    Text("Current value: 2.43")
}
```

The pattern is:

```kotlin
Column(
    modifier = Modifier.padding(16.dp)
) {
    // UI elements
}
```

## 3. Modifier

Modifier is one of the most important Compose concepts.

It changes how a UI element behaves or appears.

Common examples:

```kotlin
Modifier.padding(16.dp)
Modifier.fillMaxWidth()
Modifier.height(16.dp)
Modifier.weight(1f)
```

**How Modifiers work (Modifier chaining):**

Modifiers build instructions as a chain. Each modifier function creates a new object with additional instructions:

```kotlin
// Step 1: Start with the "blank" Modifier (identity object)
Modifier

// Step 2: Add padding instruction
Modifier.padding(10.dp)

// Step 3: Chain more instructions
Modifier
    .padding(10.dp)
    .fillMaxWidth()
    .background(Color.Gray)
```

**In practice:**

```kotlin
Text(
    text = "Research Measurement App",
    modifier = Modifier
        .padding(16.dp)        // First: add padding
        .fillMaxWidth()        // Second: make it full width
        .background(Color.LightGray)  // Third: add background color
)
```

This reads like: "Display text with 16dp padding, fill the width, and add a light gray background."

**Important rule**: Modifiers from the parent do NOT automatically apply to children. Each composable needs its own Modifier instructions.

For example:

```text
Modifier = layout/styling instructions attached to a UI element
```

For example:

```kotlin
Text(
    text = "Research Measurement App",
    modifier = Modifier.padding(16.dp)
)
```

means:

Display this text with 16 dp padding.

## 3.4 Important: Modifier Order Matters

The order in which you chain modifiers can change the visual result.

**Example 1: Background first, then padding**

```kotlin
Text(
    text = "Research Measurement App",
    modifier = Modifier
        .background(Color.Yellow)
        .padding(16.dp)
)
```

This means:
1. Draw a yellow background
2. Add padding INSIDE that yellow background

Result: Yellow background with 16dp padding around the text inside it.

```text
鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹? (yellow)                    鈹?鈹?   (16dp padding)            鈹?鈹?   Research Measurement App  鈹?鈹?   (16dp padding)            鈹?鈹? (yellow)                    鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?```

**Example 2: Padding first, then background**

```kotlin
Text(
    text = "Research Measurement App",
    modifier = Modifier
        .padding(16.dp)
        .background(Color.Yellow)
)
```

This means:
1. Add padding around the element
2. Draw the yellow background only behind the padded area

Result: Padding space with no background, then yellow background only behind the text.

```text
(16dp spacing with no color)
鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?   (yellow background)       鈹?鈹?   Research Measurement App  鈹?鈹?   (yellow background)       鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?(16dp spacing with no color)
```

**Key takeaway**: Modifier order matters for visual appearance. Test different orders to get the look you want.

## 3.5 IMPORTANT: Jetpack Compose Modifier Rules

```kotlin
/*
 * IMPORTANT: Jetpack Compose Modifier Rules
 *
 * 1. Modifiers are "Private" to the Component:
 *    When you write parent container like Surface/Card (modifier = Modifier.fillMaxWidth()), that instruction belongs only to the Surface/Card container.
 *    The Row or Column inside the Surface/Card container cannot "see" or inherit that specific Modifier object.
 *
 * 2. Constraints vs. Modifiers:
 *    Children don't inherit modifiers, but they DO inherit layout constraints.
 *    - The parent (e.g., Surface, Card) uses its modifier to decide its own size.
 *    - Because the parent is full-width, the child is placed in a full-width container.
 *    - The child doesn't need the parent's fillMaxWidth() modifier, but still receives
 *      the constraint that space is available. It can use its own fillMaxWidth() to fill that space.
 *
 * 3. The "Function Modifier" is the only Bridge:
 *    The only way to pass instructions to a child composable is via function parameters.
 *    - Modifier (capital M): The Singleton object. Starts a fresh list of instructions.
 *      Use this for internal layouts (Column inside a function, padding inside a Row, etc.).
 *    - modifier (lowercase): The function parameter variable. Contains instructions passed from parent function.
 *      Use this as the "base" for your top-level component inside a composable function.
 */
```

These rules explain why:

- A child doesn't automatically get the parent's padding or width
- Each composable must specify its own modifier
- The parent and child can have different modifiers for different purposes
- The only "communication" between parent and child about layout is through explicit function parameters

## 4. Spacer

A Spacer adds empty space.

```kotlin
Spacer(
    modifier = Modifier.height(16.dp)
)
```

Example:

Column {

```kotlin
    Text("Title")

    Spacer(modifier = Modifier.height(16.dp))

    Text("Content")
}
```

Without Spacer, elements may look too close together.

For learning, Spacer is easier to understand than more advanced layout arrangements.

## 5. fillMaxWidth()

This makes a component use all available horizontal width.

For example:

```kotlin
TextField(
    value = sampleId,
    onValueChange = { sampleId = it },
    label = { Text("Sample ID") },
    modifier = Modifier.fillMaxWidth()
)
```

Without fillMaxWidth(), the text field may only use part of the screen.

For tablet apps, this is often useful.

## 6. Row

A Row arranges items horizontally.

Row {

```kotlin
    Text("Device status:")
    Text("Connected")
}
```

This displays roughly:

Device status: Connected

A common use is putting buttons side by side:

Row {

```kotlin
    Button(
        onClick = {
            // start
        }
    ) {
        Text("Start")
    }

    Button(
        onClick = {
            // reset
        }
    ) {
        Text("Reset")
    }
}
```

However, without spacing, the buttons may be too close together.

Add a Spacer:

Row {

```kotlin
    Button(
        onClick = {
            // start
        }
    ) {
        Text("Start")
    }

    Spacer(modifier = Modifier.width(12.dp))

    Button(
        onClick = {
            // reset
        }
    ) {
        Text("Reset")
    }
}
```

You need:

```kotlin
import androidx.compose.foundation.layout.width
```

## 7. Arrangement

Arrangement controls **spacing** (how much space between children) inside a Column or Row. It affects **all children as a group**.

For example:

```kotlin
Column(
    verticalArrangement = Arrangement.spacedBy(16.dp)
) {
    Text("A")
    Text("B")
    Text("C")
}
```

This automatically places 16 dp between each item.

**Other common Arrangement options:**

```kotlin
Arrangement.Start          // Pack children at the start (top/left)
Arrangement.End            // Pack children at the end (bottom/right)
Arrangement.Center         // Center all children
Arrangement.SpaceBetween   // Spread items evenly with space between
Arrangement.SpaceAround    // Space around each item equally
Arrangement.SpaceEvenly    // Equal space between and around items
```

For example, to center items vertically:

```kotlin
Column(
    modifier = Modifier.height(300.dp),
    verticalArrangement = Arrangement.Center
) {
    Text("Centered content")
}
```

**Important**: `Arrangement.Center` only has a visible effect when the `Column` has extra vertical space available. If the `Column` only wraps its content (without a fixed height), there may be no extra space to center within. In that case, the arrangement has no effect because there's nothing to center.

That means instead of writing many Spacers, you can write:

```kotlin
Column(
    modifier = Modifier.padding(16.dp),
    verticalArrangement = Arrangement.spacedBy(16.dp)
) {
    Text("Research Measurement App")
    Text("Sample ID: S001")
    Text("Current value: 2.43")
}
```

For a beginner, both styles are fine:

```text
Spacer = explicit and easy to see
Arrangement.spacedBy = cleaner for longer layouts
```

I suggest you understand Spacer first, then gradually use Arrangement.spacedBy.

## 8. Alignment

Alignment controls **positioning** of **individual** child elements within their container. It affects **one child at a time**.

For example, to horizontally center an individual item inside a Column:

```kotlin
Column() {
    Text("Research Measurement App")
    
    Button(
        onClick = {},
        modifier = Modifier.align(Alignment.CenterHorizontally)  // This button alone is centered
    ) {
        Text("Start")
    }
    
    Text("Other text")  // This text is not centered
}
```

Compare to **Arrangement.Center** which would center **all** children:

```kotlin
Column(
    horizontalAlignment = Alignment.CenterHorizontally  // Applies to ALL children
) {
    Text("Research Measurement App")  // Centered
    Button(onClick = {}) {
        Text("Start")  // Centered
    }
    Text("Other text")  // Centered
}
```

**Key difference:**
- `Arrangement`: Controls spacing between/distribution of all children
- `Alignment`: Positions individual children within the container

You need:

```kotlin
import androidx.compose.ui.Alignment
```

For a research app, I usually would not centre everything, because form-like screens are easier to read when left-aligned.

But for a large measurement display, centering can be useful.

## 9. Box: Stacking items Front to Back

A Box stacks items on top of each other, like a stack of photos. Each new item placed in a Box appears on top of previous items.

```kotlin
Box(modifier = Modifier.fillMaxSize()) {
    // Item 1: Background (renders first)
    Image(
        painter = painterResource(R.drawable.background),
        contentDescription = null,
        modifier = Modifier.fillMaxSize(),
        contentScale = ContentScale.Crop,
        alpha = 0.5F  // Semi-transparent so items on top are visible
    )
    
    // Item 2: Text on top of the image (renders on top)
    Text(
        text = "Overlay Text",
        fontSize = 24.sp,
        modifier = Modifier.align(Alignment.Center)
    )
}
```

**Key concept**: Box renders items in order: first item is in the back, last item is in the front.

This is useful for:
- Layering an image with text overlay
- Creating backgrounds with content on top
- Card-like designs with depth

## 10. Surface: Adding a Background Layer

A Surface provides a background layer (like a sheet of paper) for its children. It can have a color, elevation (shadow), and other properties.

```kotlin
Surface(
    color = Color.Cyan,
    modifier = Modifier.padding(16.dp)
) {
    Text(
        text = "Hello Android",
        modifier = Modifier.padding(24.dp)
    )
}
```

**Conceptually:**

```text
鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹? 鈫?Surface (cyan background)
鈹?(24dp padding)           鈹?鈹? Hello Android           鈹?鈹?(24dp padding)           鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?```

Surface is commonly used with other composables to provide visual separation and background colors.


## 11. Card

A Card groups related UI content and displays it with a shadow/border to highlight it from the rest of the screen.

For example, you can place the latest measurement inside a card:

```kotlin
Card(
    modifier = Modifier.fillMaxWidth()
) {
    Column(
        modifier = Modifier.padding(16.dp)
    ) {
        Text("Current value")
        Text("2.43", fontSize = 40.sp)
    }
}
```

**Key idea**: The `Column` inside uses `Modifier.padding(16.dp)` to add space around its children inside the card container. This is internal padding, separate from the Card's own dimensions.

Conceptually:

```text
鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹? (16dp padding)             鈹?鈹? Current value              鈹?鈹? 2.43                       鈹?鈹? (16dp padding)             鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?```

This is useful for research apps because it visually separates:

- sample information

- device status

- measurement result

- action buttons

## 12. Scaffold: The Master Layout Container

As your research app grows, you may need more structure. `Scaffold` is a pre-built layout container that implements Material Design best practices.

Scaffold manages:
- Padding for system bars (status bar, navigation bar)
- Top app bar
- Bottom navigation
- Floating action button
- Snackbar notifications

**Basic example:**

```kotlin
import androidx.compose.material3.Scaffold

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        setContent {
            MaterialTheme {
                Scaffold(modifier = Modifier.fillMaxSize()) { innerPadding ->
                    Column(
                        modifier = Modifier
                            .padding(innerPadding)
                            .fillMaxSize()
                    ) {
                        Text("Research Measurement App")
                        ResearchScreen()
                    }
                }
            }
        }
    }
}
```

**Key concepts**: 

- `Scaffold` provides `innerPadding`, which accounts for system bars and app bars
- **Best practice**: Apply `innerPadding` ONCE to the root Column (or root layout container)
- This makes the screen structure clear: all content respects the Scaffold padding at the top level
- The root Column decides to fill the available size, and its children are positioned inside

**Common mistake to avoid:**

Do NOT apply `innerPadding` separately to each child. For example, avoid:

```kotlin
Scaffold(modifier = Modifier.fillMaxSize()) { innerPadding ->
    Column {
        Greeting(modifier = Modifier.padding(innerPadding))  // 鉂?Don't do this separately
        ResearchScreen(modifier = Modifier.padding(innerPadding))  // 鉂?Don't do this separately
    }
}
```

Instead, apply it once to the root layout container (the Column):

```kotlin
Scaffold(modifier = Modifier.fillMaxSize()) { innerPadding ->
    Column(
        modifier = Modifier
            .padding(innerPadding)  // 鉁?Apply once to root
            .fillMaxSize()
    ) {
        Greeting()  // Inherits space from parent
        ResearchScreen()  // Inherits space from parent
    }
}
```

This makes your app structure easier to understand: "The Scaffold handles system bars, the root Column manages content layout inside that space."

## 13. Resources: Images, Strings, and Drawables

Android organizes non-code resources in the `res/` folder. The `R` class automatically contains IDs for all resources.

**Using images:**

```kotlin
import androidx.compose.foundation.Image
import androidx.compose.ui.res.painterResource

val image = painterResource(R.drawable.androidparty)
```

**Using strings:**

Define in `res/values/strings.xml`, then reference:

```kotlin
import androidx.compose.ui.res.stringResource

Text(text = stringResource(R.string.app_name))
```

**Benefits**: Easy translation, no hardcoding, centralized updates.
## 14. Font size

You can change text size using:

```text
fontSize = 24.sp
```

Example:

```kotlin
Text(
    text = "Research Measurement App",
    fontSize = 24.sp
)
```

You need:

```kotlin
import androidx.compose.ui.unit.sp
```

For a tablet-based research app, you might want:

- Title: 24鈥?0 sp
- Section label: 16鈥?8 sp
- Main measurement value: 32鈥?8 sp
- Button text: default or 18 sp

Example:

```kotlin
Text(
    text = measurementValue?.toString() ?: "--",
    fontSize = 40.sp
)
```

This makes the measurement value easy to see.

### `sp` vs `dp`: When to Use Each Unit

You may see both `sp` and `dp` used for different things:

- **`sp` (scaled pixels)**: For text sizes only. `sp` respects the user's font-size accessibility settings.
  - Use: `fontSize = 24.sp`
  - Example: If a user increases system text size, your text will automatically get larger

- **`dp` (density-independent pixels)**: For all layout sizes, spacing, padding, width, and height.
  - Use: `modifier = Modifier.padding(16.dp)`, `height = 100.dp`, `width = 50.dp`
  - Why: Different devices have different screen densities, and `dp` scales appropriately

**Example:**

```kotlin
Text(
    text = "Research Measurement App",
    fontSize = 26.sp,  // Text size uses sp (respects accessibility settings)
    modifier = Modifier
        .padding(16.dp)     // Layout spacing uses dp
        .fillMaxWidth()     // Width constraint uses dp internally
)
```

**Memory aid**: `sp` = special for "size of text", `dp` = default for "dimensions and padding".

## 15. OutlinedTextField

In Lesson 4, we used TextField.

For forms, I often prefer OutlinedTextField because it looks clearer:

```kotlin
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
```

You need:

```kotlin
import androidx.compose.material3.OutlinedTextField
```

For a research app, this is a good default for data entry fields.

## 16. Better research screen design

Let鈥檚 design a simple screen:

Research Measurement App

[Sample ID                  ]

Device status: Connected

```text
鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?Current value                鈹?鈹?2.43                         鈹?鈹?Measurements: 3              鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?```

[Start Measurement] [Reset]

This is still simple, but much more usable than the previous version.

## 17. Full code 鈥?improved layout

Replace your MainActivity.kt with this version, adjusting the package name if needed:

```kotlin
package com.example.researchapp

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.width
import androidx.compose.material3.Button
import androidx.compose.material3.Card
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
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

/*
 * IMPORTANT: Jetpack Compose Modifier Rules
 *
 * 1. Modifiers are "Private" to the Component:
 *    When you write Card(modifier = Modifier.fillMaxWidth()), that instruction belongs only to the Card.
 *    The Row or Column inside the Card cannot "see" or inherit that specific Modifier object.
 *
 * 2. Constraints vs. Modifiers:
 *    Children don't inherit modifiers, but they DO inherit layout constraints.
 *    - The parent (e.g., Card) uses its modifier to decide its own size.
 *    - Because the parent is full-width, the child is placed in a full-width container.
 *    - The child doesn't need the parent's fillMaxWidth() modifier, but still receives
 *      the constraint that space is available. It can use its own fillMaxWidth() to fill that space.
 *
 * 3. The "Function Modifier" is the only Bridge:
 *    The only way to pass instructions to a child composable is via function parameters.
 *    - Modifier (capital M): The Singleton object. Starts a fresh list of instructions.
 *      Use this for internal layouts (Column inside a function, padding inside a Row, etc.).
 *    - modifier (lowercase): The function parameter variable. Contains instructions passed from parent function.
 *      Use this as the "base" for your top-level component inside a composable function.
 */

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

    val deviceStatus = "Connected"

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

        Text(
            text = "Device status: $deviceStatus"
        )

        Card(
            modifier = Modifier.fillMaxWidth()
        ) {
            Column(
                modifier = Modifier.padding(16.dp),
                verticalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                Text("Current value")

                Text(
                    text = measurementValue?.let {
                        "%.3f".format(it)
                    } ?: "--",
                    fontSize = 40.sp
                )

                Text("Measurements: $measurementCount")
            }
        }

        Row {
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

            Spacer(modifier = Modifier.width(12.dp))

            Button(
                onClick = {
                    measurementValue = null
                    measurementCount = 0
                }
            ) {
                Text("Reset")
            }
        }
    }
}
```

## 18. Important new line: formatted number

This part may look new:

"%.3f".format(it)

It means:

Format this number with 3 decimal places.

For example:

```kotlin
val x = 2.438912
println("%.3f".format(x))
```

Output:

2.439

This is useful in research apps because raw sensor values may have too many decimals.

So instead of displaying:

2.438912398123

you display:

2.439

## 19. Important new line: let

This part:

measurementValue?.let {

```text
    "%.3f".format(it)
} ?: "--"
```

means:

If measurementValue is not null:

```text
    format it to 3 decimal places
```

Otherwise:

```text
    show "--"
```

This is equivalent to the longer version:

```kotlin
val displayText = if (measurementValue != null) {
    "%.3f".format(measurementValue)
} else {
    "--"
}
```

However, the let version works nicely with nullable values.

You do not need to use let immediately, but you should start recognising it.

## 20. A simpler version without let

If the let syntax feels too advanced, use this inside your Text:

```kotlin
Text(
    text = if (measurementValue != null) {
        "%.3f".format(measurementValue)
    } else {
        "--"
    },
    fontSize = 40.sp
)
```

However, Android Studio may complain slightly because measurementValue is nullable. A safer beginner-friendly way is:

```kotlin
val displayValue = measurementValue

Text(
    text = if (displayValue != null) {
        "%.3f".format(displayValue)
    } else {
        "--"
    },
    fontSize = 40.sp
)
```

But in practice, this is clean:

measurementValue?.let {

```text
    "%.3f".format(it)
} ?: "--"
```

## 21. Make buttons fill the width

On a tablet, buttons side by side can look better if each occupies half the available width.

You can write:

```kotlin
Row(
    modifier = Modifier.fillMaxWidth(),
    horizontalArrangement = Arrangement.spacedBy(12.dp)
) {
    Button(
        onClick = {
            // start
        },
        modifier = Modifier.weight(1f)
    ) {
        Text("Start Measurement")
    }

    Button(
        onClick = {
            // reset
        },
        modifier = Modifier.weight(1f)
    ) {
        Text("Reset")
    }
}
```

The important part is:

```kotlin
modifier = Modifier.weight(1f)
```

inside each button.

This means:

Each button takes an equal share of the row width.

So instead of:

[Start Measurement] [Reset]

you get something more like:

[ Start Measurement      ] [ Reset                  ]

## 22. Updated button row

Replace the previous Row with this:

```kotlin
Row(
    modifier = Modifier.fillMaxWidth(),
    horizontalArrangement = Arrangement.spacedBy(12.dp)
) {
    Button(
        onClick = {
            measurementValue = Random.nextDouble(
                from = 0.0,
                until = 5.0
            )

            measurementCount = measurementCount + 1
        },
        modifier = Modifier.weight(1f)
    ) {
        Text("Start Measurement")
    }

    Button(
        onClick = {
            measurementValue = null
            measurementCount = 0
        },
        modifier = Modifier.weight(1f)
    ) {
        Text("Reset")
    }
}
```

Since we are now using:

```text
horizontalArrangement = Arrangement.spacedBy(12.dp)
```

we no longer need:

```kotlin
Spacer(modifier = Modifier.width(12.dp))
```

So you can remove the Spacer and width import if they are unused.

## 23. enabled property

Buttons can be enabled or disabled.

For example, maybe the Start button should only be enabled if the user has entered a sample ID.

```kotlin
Button(
    onClick = {
        measurementValue = Random.nextDouble(0.0, 5.0)
        measurementCount = measurementCount + 1
    },
    enabled = sampleId.isNotBlank()
) {
    Text("Start Measurement")
}
```

This means:

If sampleId is blank, disable the button.

Useful string checks:

```text
sampleId.isBlank()
sampleId.isNotBlank()
```

Examples:

- ""              // blank
- "   "           // blank
- "S001"          // not blank

This is important for research apps because you may want to prevent accidental recordings without labels.

## 24. Add a warning message

If the sample ID is empty, show:

Please enter a sample ID before measuring.

Example:

```kotlin
if (sampleId.isBlank()) {
    Text("Please enter a sample ID before measuring.")
}
```

In Compose, you can use normal Kotlin if statements inside the UI.

That is very powerful.

For example:

Column {

```kotlin
    Text("Research Measurement App")

    if (sampleId.isBlank()) {
        Text("Please enter a sample ID.")
    }
}
```

If the condition is true, the message appears.

If the condition is false, the message disappears.

## 25. Final improved version for Lesson 5

This version has:

- nicer layout

- title

- editable sample ID

- warning if sample ID is empty

- device status

- card for latest result

- formatted value

- measurement count

- full-width buttons

- disabled Start button until sample ID is entered

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
import androidx.compose.material3.Button
import androidx.compose.material3.Card
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
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

    val deviceStatus = "Connected"

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
            text = "Device status: $deviceStatus"
        )

        Card(
            modifier = Modifier.fillMaxWidth()
        ) {
            Column(
                modifier = Modifier.padding(16.dp),
                verticalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                Text("Current value")

                Text(
                    text = measurementValue?.let {
                        "%.3f".format(it)
                    } ?: "--",
                    fontSize = 40.sp
                )

                Text("Measurements: $measurementCount")
            }
        }

        Row(
            modifier = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.spacedBy(12.dp)
        ) {
            Button(
                onClick = {
                    measurementValue = Random.nextDouble(
                        from = 0.0,
                        until = 5.0
                    )

                    measurementCount = measurementCount + 1
                },
                enabled = sampleId.isNotBlank(),
                modifier = Modifier.weight(1f)
            ) {
                Text("Start Measurement")
            }

            Button(
                onClick = {
                    measurementValue = null
                    measurementCount = 0
                },
                modifier = Modifier.weight(1f)
            ) {
                Text("Reset")
            }
        }
    }
}
```

## 26. What this teaches you

This lesson is not just about making the app prettier.

You learned several very important Android UI ideas:

```kotlin
Column = vertical layout
Row = horizontal layout
Spacer = manual space
Arrangement.spacedBy = automatic space
Modifier.padding = outside/inside spacing
Modifier.fillMaxWidth = use full width
Modifier.weight = share row/column space
Card = group related content
OutlinedTextField = form input
```

- Button enabled = prevent invalid action
- if inside Compose = conditional UI

These are enough to build many simple screens.

## 27. Your small exercise

Modify the app so the device status is not just fixed text.

Add this state:

```kotlin
var isConnected by remember {
    mutableStateOf(false)
}
```

Display:

```kotlin
Text(
    text = if (isConnected) {
        "Device status: Connected"
    } else {
        "Device status: Disconnected"
    }
)
```

Then add a button:

Connect / Disconnect

Its click logic should be:

```text
isConnected = !isConnected
```

And make the measurement button enabled only when:

sampleId.isNotBlank() && isConnected

So the user can only start measurement when:

```text
sample ID exists
AND
device is connected
```

This is very close to real research-app logic.

## Lesson 6 preview

Next, I suggest we move from 鈥渓atest value only鈥?to a list of measurements.

Lesson 6 should cover:

```kotlin
data class Measurement
```

```text
MutableList<Measurement>
adding a new measurement
displaying measurement history
LazyColumn
calculating mean/min/max
```

That will make the app feel much closer to an actual data-collection tool.
