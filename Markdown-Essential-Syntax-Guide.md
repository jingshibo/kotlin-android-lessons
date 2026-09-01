# Markdown Essential Syntax Guide

This guide covers the main Markdown syntax needed for writing structured notes, technical documentation, and Kotlin/Android lessons in Visual Studio Code.


## Quick reference

````text
# Heading
## Subheading
**Bold**
*Italic*
`Inline code`
- Bullet point
1. Numbered item
- [ ] Checklist item
[Link text](URL)
![Image description](path)
> Blockquote

```language
Code block
```
````

These elements are sufficient for most lesson notes and technical documentation.

---


## 1. Headings

Use `#` symbols to create headings. More symbols produce a lower-level heading.

```markdown
# Lesson title
## Main section
### Subsection
#### Smaller subsection
```

Use headings hierarchically: `#` for the document title, `##` for major sections, and `###` for subsections.

---

## 2. Paragraphs and line breaks

Separate paragraphs with a blank line:

```markdown
This is the first paragraph.

This is the second paragraph.
```

To insert a line break without starting a new paragraph, end the first line with two spaces:

```markdown
First line.  
Second line.
```

---

## 3. Bold, italic, and strikethrough

```markdown
**Bold text**

*Italic text*

***Bold and italic text***

~~Strikethrough text~~
```

Rendered result:

**Bold text**

*Italic text*

***Bold and italic text***

~~Strikethrough text~~

---

## 4. Bulleted lists

Use a hyphen followed by a space:

```markdown
- Kotlin
- Android Studio
- Jetpack Compose
```

Indent items to create a nested list:

```markdown
- Kotlin basics
  - Variables
  - Functions
  - Classes
- Android basics
  - Activities
  - User interfaces
```

---

## 5. Numbered lists

```markdown
1. Install Android Studio.
2. Create a new project.
3. Run the application.
```

Nested numbered list:

```markdown
1. Create the project.
   1. Select a template.
   2. Enter the project name.
2. Run the project.
```

Markdown can number the list automatically if every item begins with `1.`:

```markdown
1. First step
1. Second step
1. Third step
```

---

## 6. Checklists

```markdown
- [x] Install Android Studio
- [x] Create a Kotlin file
- [ ] Build the research application
- [ ] Deploy the CNN model
```

Rendered result:

- [x] Install Android Studio
- [x] Create a Kotlin file
- [ ] Build the research application
- [ ] Deploy the CNN model

---

## 7. Inline code

Enclose a short piece of code in single backticks:

```markdown
Use `val` to declare a read-only variable.
```

Rendered result:

Use `val` to declare a read-only variable.

Inline code is useful for variable names, function names, filenames, and commands.

---

## 8. Code blocks

Place three backticks before and after the code. Add the programming language after the opening backticks to enable syntax highlighting.

````markdown
```kotlin
fun main() {
    val sampleCount = 10
    println("Number of samples: $sampleCount")
}
```
````

Other useful language labels include:

````markdown
```python
print("Python example")
```

```bash
mkdir Kotlin-Lessons
```

```json
{
  "sampleCount": 10
}
```
````

---

## 9. Links

Link to a webpage:

```markdown
[Android Developers](https://developer.android.com/)
```

Display a URL directly:

```markdown
<https://developer.android.com/>
```

Link to another Markdown file:

```markdown
[Continue to Lesson 2](02-Control-Flow.md)
```

Link to a heading in the same file:

```markdown
[Jump to Code blocks](#8-code-blocks)
```

Heading links are generally lowercase, with spaces replaced by hyphens.

---

## 10. Images

Online image:

```markdown
![Description of the image](https://example.com/image.png)
```

Local image:

```markdown
![Android application interface](images/app-interface.png)
```

Example folder structure:

```text
Kotlin-Android-Learning/
├── 01-Kotlin-Basics.md
└── images/
    └── app-interface.png
```

The text inside the square brackets is alternative text describing the image.

---

## 11. Blockquotes

```markdown
> Kotlin is the recommended language for modern Android development.
```

Rendered result:

> Kotlin is the recommended language for modern Android development.

A blockquote can contain multiple paragraphs:

```markdown
> **Important:**
>
> Always test the application using independent research samples.
```

---

## 12. Horizontal separators

Enter three hyphens on a separate line:

```markdown
---
```

This creates a horizontal line for separating major sections.

---

## 13. Tables

```markdown
| Kotlin type | Python equivalent | Example |
|---|---|---|
| `Int` | `int` | `val n: Int = 10` |
| `Double` | `float` | `val x = 2.5` |
| `String` | `str` | `val name = "Sample"` |
| `Boolean` | `bool` | `val valid = true` |
```

Rendered result:

| Kotlin type | Python equivalent | Example |
|---|---|---|
| `Int` | `int` | `val n: Int = 10` |
| `Double` | `float` | `val x = 2.5` |
| `String` | `str` | `val name = "Sample"` |
| `Boolean` | `bool` | `val valid = true` |

Control column alignment using colons:

```markdown
| Left aligned | Centred | Right aligned |
|:---|:---:|---:|
| Text | Text | 100 |
```

---

## 14. Escaping special characters

Add a backslash when you want a Markdown formatting symbol to appear literally:

```markdown
\*This is not italic\*

\# This is not a heading
```

Rendered result:

\*This is not italic\*

\# This is not a heading

---

## 15. Practical lesson template

````markdown
# Lesson 1: Kotlin Variables and Data Types

## Learning objectives

By the end of this lesson, you should be able to:

- Declare variables using `val` and `var`
- Recognise common Kotlin data types
- Print values to the console

---

## 1. Variables

Kotlin provides two main ways to declare variables:

- `val`: a read-only variable
- `var`: a changeable variable

### Example

```kotlin
val sampleName = "Sample A"
var measurementCount = 10

measurementCount = 11
```

> **Research example:** Use `val` when a value should not change after it has been assigned.

## 2. Comparison with Python

| Purpose | Kotlin | Python |
|---|---|---|
| Read-only variable | `val count = 10` | No direct equivalent |
| Changeable variable | `var count = 10` | `count = 10` |

## Exercise

- [ ] Declare a sample name.
- [ ] Declare a measurement count.
- [ ] Print both values.

## Summary

In this lesson, you learned:

1. How to use `val`
2. How to use `var`
3. How Kotlin variables compare with Python

[Continue to Lesson 2](02-Control-Flow.md)
````


