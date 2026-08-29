# Lesson 24 — Creating the Core Data Model Files

In Lesson 23, we turned the overall architecture into a clean Android project structure.

We created the target package structure:

```text
com.example.researchapp
 ├── ui
 ├── viewmodel
 ├── data
 │    ├── entity
 │    └── dao
 ├── device
 ├── processing
 ├── ml
 └── export
```

Now we start implementing the first real part of the project:

```text
core data model files
```

This corresponds to **Step A2** in Direction A:

```text
Define core data classes
```

The goal of Lesson 24 is to create these files:

```text
PatientEntity.kt
SessionEntity.kt
MeasurementEntity.kt
ResultEntity.kt
```

These files define what the app stores.

---

## 1. Why data model comes first

Before we build screens, device connection, processing, ML, or export, we need to know:

```text
What data does the app manage?
```

For our research app, the core model is:

```text
Patient
 ↓
Session
 ↓
Measurement
 ↓
Result
```

This structure was introduced in Lesson 15.

The most important rule was:

```text
Do not only save values.
Save values with context.
```

A measurement should not exist alone.

It should belong to a session.

A session should belong to a patient or sample.

A result should belong to a session.

So our Room data model should reflect this structure.

---

## 2. Create the `entity` package

Inside your Android project, create this folder:

```text
app/src/main/java/com/example/researchapp/data/entity
```

Inside it, create four files:

```text
PatientEntity.kt
SessionEntity.kt
MeasurementEntity.kt
ResultEntity.kt
```

The package name at the top of each file should be:

```kotlin
package com.example.researchapp.data.entity
```

This means these files belong to the `data.entity` package.

---

## 3. Why the name ends with `Entity`

You may wonder why we use:

```text
PatientEntity
```

instead of simply:

```text
Patient
```

The reason is clarity.

In Room, an entity means:

```text
a Kotlin class that represents a database table
```

So:

```text
PatientEntity
 ↓
patients table

SessionEntity
 ↓
sessions table

MeasurementEntity
 ↓
measurements table

ResultEntity
 ↓
results table
```

Later, in a bigger app, you may also have domain models such as:

```text
Patient
Session
Measurement
Result
```

separate from database entities.

But for now, using `Entity` in the name makes the purpose clear.

---

## 4. Patient entity

Create:

```text
PatientEntity.kt
```

Code:

```kotlin
package com.example.researchapp.data.entity

import androidx.room3.Entity
import androidx.room3.PrimaryKey

@Entity(tableName = "patients")
data class PatientEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,

    val patientCode: String,
    val notes: String = "",
    val createdAt: Long = System.currentTimeMillis()
)
```

This creates a Room table called:

```text
patients
```

Each row represents one patient, participant, sample, or subject.

---

## 5. Understanding `PatientEntity`

Let us explain each field.

```kotlin
@PrimaryKey(autoGenerate = true)
val id: Long = 0
```

This is the database ID.

Room will generate it automatically when we insert a new patient.

Example:

```text
Patient P001 may get database id = 1.
Patient P002 may get database id = 2.
```

Then:

```kotlin
val patientCode: String
```

This is the human-readable ID.

For example:

```text
P001
P002
S001
Sample-A
D1-ETO-W0-U1-S1
```

This is the ID the user recognises.

Then:

```kotlin
val notes: String = ""
```

This stores optional notes.

For example:

```text
"Baseline subject"
"Validation sample"
"Uniform sample before washing"
```

Then:

```kotlin
val createdAt: Long = System.currentTimeMillis()
```

This stores when the patient/sample record was created.

We use `Long` because `System.currentTimeMillis()` returns time in milliseconds.

---

## 6. Why have both `id` and `patientCode`?

This is important.

The database ID:

```kotlin
id
```

is for the app and database.

The patient code:

```kotlin
patientCode
```

is for the researcher/user.

For example:

```text
id = 1
patientCode = "P001"
```

The app uses `id` to connect tables.

The user sees `patientCode`.

Do not use `patientCode` as the only database link, because it might change, contain typing mistakes, or not be guaranteed unique unless you enforce it.

For the beginner version:

```text
Use id for database relationships.
Use patientCode for display and export.
```

---

## 7. Session entity

Create:

```text
SessionEntity.kt
```

Code:

```kotlin
package com.example.researchapp.data.entity

import androidx.room3.Entity
import androidx.room3.PrimaryKey

@Entity(tableName = "sessions")
data class SessionEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,

    val patientId: Long,
    val sessionName: String,
    val startedAt: Long = System.currentTimeMillis(),
    val endedAt: Long? = null,
    val notes: String = ""
)
```

This creates a table called:

```text
sessions
```

Each row represents one data-collection session.

---

## 8. Understanding `SessionEntity`

The first field is:

```kotlin
val id: Long = 0
```

This is the database ID for the session.

Then:

```kotlin
val patientId: Long
```

This is very important.

It connects the session to a patient.

For example:

```text
PatientEntity:
id = 1
patientCode = "P001"

SessionEntity:
id = 10
patientId = 1
sessionName = "Baseline"
```

This means:

```text
Session 10 belongs to Patient 1.
```

Then:

```kotlin
val sessionName: String
```

This is a human-readable session name.

Examples:

```text
Baseline
After treatment
Morning test
Validation run 1
Uniform face A
```

Then:

```kotlin
val startedAt: Long = System.currentTimeMillis()
```

This stores when the session started.

Then:

```kotlin
val endedAt: Long? = null
```

This stores when the session ended.

It is nullable because when the session is first created, it has not ended yet.

```text
endedAt = null
 ↓
session is not ended yet

endedAt = 1755700600
 ↓
session has ended
```

Then:

```kotlin
val notes: String = ""
```

This can store session notes.

---

## 9. Why `endedAt` is nullable

This field is:

```kotlin
Long?
```

not:

```kotlin
Long
```

because the session may be active.

When you create a session:

```text
startedAt has a value
endedAt is null
```

When you stop acquisition:

```text
endedAt becomes current time
```

This matches the real app flow:

```text
Create session
 ↓
Start acquisition
 ↓
Stop acquisition
 ↓
Set endedAt
```

So nullable fields are useful when the data may not exist yet.

---

## 10. Measurement entity

Create:

```text
MeasurementEntity.kt
```

Code:

```kotlin
package com.example.researchapp.data.entity

import androidx.room3.Entity
import androidx.room3.PrimaryKey

@Entity(tableName = "measurements")
data class MeasurementEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,

    val sessionId: Long,
    val repetition: Int,

    val rawValue: Double,
    val processedValue: Double,

    val timestamp: Long = System.currentTimeMillis(),
    val status: String = "OK"
)
```

This creates a table called:

```text
measurements
```

Each row represents one measurement value.

---

## 11. Understanding `MeasurementEntity`

The database ID:

```kotlin
val id: Long = 0
```

is the unique row ID.

Then:

```kotlin
val sessionId: Long
```

connects the measurement to a session.

For example:

```text
SessionEntity:
id = 10

MeasurementEntity:
id = 100
sessionId = 10
rawValue = 2.438
processedValue = 2.238
```

This means:

```text
Measurement 100 belongs to Session 10.
```

Then:

```kotlin
val repetition: Int
```

stores the measurement repetition number.

For example:

```text
1
2
3
4
5
```

Then:

```kotlin
val rawValue: Double
```

stores the value received directly from the device.

Then:

```kotlin
val processedValue: Double
```

stores the value after signal processing.

For example:

```text
rawValue = 2.438
processedValue = 2.238
```

The difference may come from baseline correction, smoothing, or other processing.

Then:

```kotlin
val timestamp: Long = System.currentTimeMillis()
```

stores when the measurement was created.

Then:

```kotlin
val status: String = "OK"
```

stores whether the measurement is valid.

Examples:

```text
OK
INVALID
NOISY
ERROR
```

For now, we use a simple `String`.

Later, we could use an enum.

---

## 12. Why save both raw and processed values?

This is an important research-app decision.

A simple app might save only:

```text
value
```

But a research app should usually save:

```text
rawValue
processedValue
```

Why?

Because later you may change your processing method.

For example:

```text
Version 1:
processedValue = rawValue - baseline

Version 2:
processedValue = smoothed(rawValue - baseline)

Version 3:
processedValue = filtered and normalised value
```

If you only save the processed value, you may lose the original data.

If you save raw value too, you can reprocess later.

So the safer research habit is:

```text
Save raw data when possible.
Save processed data separately.
```

---

## 13. Result entity

Create:

```text
ResultEntity.kt
```

Code:

```kotlin
package com.example.researchapp.data.entity

import androidx.room3.Entity
import androidx.room3.PrimaryKey

@Entity(tableName = "results")
data class ResultEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,

    val sessionId: Long,
    val label: String,
    val confidence: Double,
    val createdAt: Long = System.currentTimeMillis()
)
```

This creates a table called:

```text
results
```

Each row represents one prediction, classification, or analysis result.

---

## 14. Understanding `ResultEntity`

The database ID:

```kotlin
val id: Long = 0
```

is the unique row ID.

Then:

```kotlin
val sessionId: Long
```

connects the result to a session.

For example:

```text
SessionEntity:
id = 10

ResultEntity:
id = 200
sessionId = 10
label = "Positive"
confidence = 0.92
```

This means:

```text
Result 200 belongs to Session 10.
```

Then:

```kotlin
val label: String
```

stores the predicted class or result label.

Examples:

```text
Positive
Negative
Class A
Class B
Valid
Invalid
```

Then:

```kotlin
val confidence: Double
```

stores the model confidence.

For example:

```text
0.92
0.85
0.63
```

Then:

```kotlin
val createdAt: Long = System.currentTimeMillis()
```

stores when the result was created.

---

## 15. Should result include model version?

Eventually, yes.

For a serious research app, `ResultEntity` should probably include:

```kotlin
val modelVersion: String
val processingVersion: String
```

because later you may need to know:

```text
Which model produced this result?
Which processing pipeline was used?
Which feature window was used?
```

But for the beginner Direction A skeleton, keep it simple first:

```text
sessionId
label
confidence
createdAt
```

We can add model version later when we integrate a real model.

---

## 16. Relationship between the four entities

Now we have four tables:

```text
patients
sessions
measurements
results
```

Their relationships are:

```text
PatientEntity.id
 ↓
SessionEntity.patientId

SessionEntity.id
 ↓
MeasurementEntity.sessionId

SessionEntity.id
 ↓
ResultEntity.sessionId
```

Visually:

```text
PatientEntity
 └── SessionEntity
      ├── MeasurementEntity
      ├── MeasurementEntity
      ├── MeasurementEntity
      └── ResultEntity
```

This is the core research data structure.

---

## 17. Example complete data flow

Imagine the user creates a patient:

```text
patientCode = "P001"
```

Room inserts:

```text
PatientEntity(id = 1, patientCode = "P001")
```

Then the user creates a session:

```text
sessionName = "Baseline"
```

Room inserts:

```text
SessionEntity(id = 10, patientId = 1, sessionName = "Baseline")
```

Then the app collects measurements:

```text
MeasurementEntity(id = 100, sessionId = 10, repetition = 1)
MeasurementEntity(id = 101, sessionId = 10, repetition = 2)
MeasurementEntity(id = 102, sessionId = 10, repetition = 3)
```

Then the app runs inference:

```text
ResultEntity(id = 200, sessionId = 10, label = "Positive", confidence = 0.92)
```

Now the app knows:

```text
Result 200 came from Session 10.
Session 10 belongs to Patient 1.
Patient 1 is P001.
```

That is why IDs are important.

---

## 18. Important note about Room schema changes

These entity classes define the database schema.

If you later change a field, for example:

```kotlin
val deviceId: String
```

or:

```kotlin
val modelVersion: String
```

you are changing the database structure.

In a learning app, you may uninstall and reinstall the app during development.

But in a real research deployment, you must not casually delete the app because that may delete collected data.

Later, you would need:

```text
Room migrations
```

For now, just remember:

```text
Changing entity fields changes the database schema.
```

---

## 19. Should we add foreign keys now?

Room supports foreign keys, which can formally enforce relationships such as:

```text
Session must belong to an existing Patient.
Measurement must belong to an existing Session.
Result must belong to an existing Session.
```

That is useful.

However, for this Direction A beginner skeleton, I suggest not adding foreign keys immediately.

Why?

Because foreign keys add more details:

```text
cascade delete
relationship constraints
migration concerns
more annotation syntax
```

For now, we keep relationships simple using ID fields:

```text
patientId
sessionId
```

Later, in an advanced Room lesson, we can add formal foreign keys and relationships.

The beginner goal is:

```text
understand and implement the core structure first
```

not solve every database rule at once.

---

## 20. Should we use enums instead of strings?

Some fields could become enums later.

For example:

```kotlin
val status: String = "OK"
```

could become:

```kotlin
enum class MeasurementStatus {
    OK,
    INVALID,
    ERROR
}
```

But Room needs type converters to store custom enum types cleanly.

So for now, using `String` is simpler.

Beginner version:

```text
status: String
```

Advanced version later:

```text
status: MeasurementStatus with Room type converter
```

Again, the pattern is:

```text
simple first
more robust later
```

---

## 21. Current files after Lesson 24

After this lesson, your `entity` folder should contain:

```text
data/entity
 ├── PatientEntity.kt
 ├── SessionEntity.kt
 ├── MeasurementEntity.kt
 └── ResultEntity.kt
```

And each file should contain one Room entity.

The app data model is now real Kotlin code.

---

## 22. What you learned in Lesson 24

You learned how to create the core database entity files:

```text
PatientEntity
SessionEntity
MeasurementEntity
ResultEntity
```

You learned that:

```text
PatientEntity
```

stores the patient/sample record.

```text
SessionEntity
```

stores one data-collection session and links to a patient through `patientId`.

```text
MeasurementEntity
```

stores raw and processed measurement values and links to a session through `sessionId`.

```text
ResultEntity
```

stores ML or analysis results and links to a session through `sessionId`.

The most important mental model is:

```text
Data needs context.
```

So our structure is:

```text
Patient
 ↓
Session
 ↓
Measurement
 ↓
Result
```

or more precisely:

```text
PatientEntity.id
 ↓
SessionEntity.patientId

SessionEntity.id
 ↓
MeasurementEntity.sessionId
ResultEntity.sessionId
```

This is the foundation for Room, repository, export, and ML results.

---

# Lesson 25 preview

In Lesson 25, we will build the Room database layer around these entities.

We will create:

```text
PatientDao
SessionDao
MeasurementDao
ResultDao
ResearchDatabase
```

That will let the app actually insert, read, update, and query these entities.
