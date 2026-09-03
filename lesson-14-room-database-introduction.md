# Lesson 14 — Room Database Introduction

In Lesson 13, we introduced the **Repository layer**.

Before Lesson 13, the ViewModel was doing too much:

```text
ResearchViewModel
 ├── UI state
 ├── acquisition flow
 ├── fake data generation
 ├── file saving
 └── file loading
```

After Lesson 13, we started moving data-related work into:

```text
MeasurementRepository
```

Now we take the next step.

So far, our app has used simple internal file storage and CSV-style saving. That is useful for learning, and the original tutorial already introduced internal storage as a beginner-friendly auto-save method. fileciteturn1file0L397-L405

But for a more serious research app, simple files may become inconvenient.

For example, later we may need to store lots of structural dataset:

```text
patients
sessions
measurements
results
timestamps
device information
model predictions
export history
```

For that, Android commonly uses **Room**.

Room is a persistence library that provides an abstraction layer over SQLite. Google recommends using Room instead of working directly with the low-level SQLite APIs, because Room gives benefits such as compile-time SQL checking, useful annotations, and better migration support. citeturn172874view0

---

## 1. Why do we need Room?

Our current file-based storage is roughly like this:

```text
measurements.csv
```

That is simple and useful.

But imagine your app becomes more realistic:

```text
Patient A
 ├── Session 1
 │    ├── Measurement 1
 │    ├── Measurement 2
 │    └── Result
 └── Session 2
      ├── Measurement 1
      ├── Measurement 2
      └── Result

Patient B
 └── Session 1
      ├── Measurement 1
      └── Result
```

If everything is just one CSV file, it becomes harder to answer questions like:

```text
Which measurements belong to this session?
Which sessions belong to this patient?
What was the latest session?
Which data has not been exported yet?
Which measurements produced an abnormal result?
```

A database is better for this kind of structured data.

Room helps us use a database in an Android-friendly way.

---

## 2. Room is not the same as CSV export

This is important.

Room is for **local structured storage inside the app**.

CSV export is for **sharing data outside the app**.

So we should not think:

```text
Room replaces CSV export completely.
```

A better model is:

```text
Room
 ↓
stores app data locally and structurally

CSV export
 ↓
exports selected data for Excel, Python, backup, or analysis
```

For a research app, both are useful.

```text
Room = app's internal database
CSV/JSON = external research data export
```

---

## 3. The three main parts of Room

Room has three main components:

```text
Entity
DAO
Database
```

Google’s Room documentation describes these three major components as data entities, data access objects or DAOs, and the database class. The database class gives access to the DAOs; DAOs provide functions for querying, inserting, updating, and deleting data. citeturn172874view0

For our research app:

```text
Entity
 ↓
What does one database table look like?

DAO
 ↓
What operations can we perform on that table?

Database
 ↓
The main Room database object that connects everything.
```

Let us go through them one by one.

---

## 4. Entity

An entity is a Kotlin class that represents a table in the database.

Room documentation explains that each entity corresponds to a table, and each instance of an entity represents a row in that table. citeturn172874view1

For our app, we want to store measurements.

So we can create:

```kotlin
@Entity(tableName = "measurements") // @Entity marks a class as a database table.
data class Measurement // Measurement becomes a row in the table and each entry becomes a column
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    val sampleId: String,
    val repetition: Int,
    val value: Double,
    val timestamp: Long,
    val status: String
)
```

This looks similar to our previous `Measurement` data class, but now we added Room annotations.

The annotation:

```kotlin
@Entity(tableName = "measurements")
```

means:

```text
Create a database table called measurements.
```

The annotation:

```kotlin
@PrimaryKey(autoGenerate = true)
val id: Long = 0
```

means:

```text
Each measurement row needs a unique ID.
Room can generate this ID automatically.
```

The other properties become columns:

```text
sampleId
repetition
value
timestamp
status
```

So this class:

```kotlin
Measurement(...)
```

becomes a row in the database.

---

## 5. Important note about `id`

Previously, our `Measurement` had:

```kotlin
val sampleId: String
val repetition: Int
val value: Double
val timestamp: Long
val status: String
```

Now we added:

```kotlin
val id: Long = 0
```

Why?

Because database tables usually need a unique identifier for each row.

For example, two measurements may have the same sample ID and even similar values.

So Room needs a stable way to identify each row.

When we create a new measurement, we can still write:

```kotlin
Measurement(
    sampleId = "S001",
    repetition = 1,
    value = 2.45,
    timestamp = System.currentTimeMillis(),
    status = "OK"
)
```

We do not need to manually provide `id`, because the default value is:

```kotlin
id = 0
```

Room will generate the real ID when inserting it.

---

## 6. Required imports for the entity

With current Room 3 style, the imports are:

```kotlin
import androidx.room3.Entity
import androidx.room3.PrimaryKey
```

You may see older tutorials using:

```kotlin
import androidx.room.Entity
import androidx.room.PrimaryKey
```

That is from Room 2.x. Current Room 3 uses the `androidx.room3` package and Room 3 requires KSP for annotation processing. citeturn172874view4

For this tutorial, the main concept is the same:

```text
@Entity marks a class as a database table.
@PrimaryKey marks the unique ID column.
```

---

## 7. DAO

DAO means:

```text
Data Access Object
```

The DAO defines how we access the database.

For our measurement table, create:

```kotlin
@Dao
interface MeasurementDao {

    @Insert
    suspend fun insertMeasurement(
        measurement: Measurement
    )

    @Query("SELECT * FROM measurements ORDER BY timestamp ASC")
    suspend fun getAllMeasurements(): List<Measurement>

    @Query("DELETE FROM measurements")
    suspend fun deleteAllMeasurements()
}
```

Required imports:

```kotlin
import androidx.room3.Dao
import androidx.room3.Insert
import androidx.room3.Query
```

The DAO is very important.

The ViewModel or repository should not directly write SQL everywhere.

Instead, they call DAO functions.

Room’s documentation says DAOs contain methods that provide abstract access to the database, and Room generates the DAO implementations at compile time. citeturn172874view2

---

## 8. Understand the DAO functions

### Insert one measurement

```kotlin
@Insert
suspend fun insertMeasurement(
    measurement: Measurement
)
```

This means:

```text
Insert this Measurement object into the measurements table.
```

The `@Insert` annotation tells Room:

```text
This function inserts data.
```

---

### Read all measurements

```kotlin
@Query("SELECT * FROM measurements ORDER BY timestamp ASC")
suspend fun getAllMeasurements(): List<Measurement>
```

This means:

```text
Read all rows from the measurements table,
ordered by timestamp from oldest to newest.
```

The SQL part is:

```sql
SELECT * FROM measurements ORDER BY timestamp ASC
```

Do not worry too much about SQL yet.

For now, understand:

```text
SELECT * FROM measurements
 ↓
get all measurements

ORDER BY timestamp ASC
 ↓
sort them by time
```

Room validates SQL queries at compile time, so query problems can be caught while building the app rather than only failing at runtime. citeturn172874view2

---

### Delete all measurements

```kotlin
@Query("DELETE FROM measurements")
suspend fun deleteAllMeasurements()
```

This means:

```text
Remove all saved measurements from the table.
```

This may be useful for a simple learning app.

Later, for a real research app, we may not delete everything so casually.

Instead, we may delete by session:

```text
delete measurements from Session 3 only
```

But not yet.

---

## 9. Why DAO functions are `suspend`

You may notice:

```kotlin
suspend fun insertMeasurement(...)
suspend fun getAllMeasurements()
suspend fun deleteAllMeasurements()
```

This connects directly to Lesson 10.

Database operations should not block the UI.

Room supports coroutines, and its documentation says one-shot read/write DAO queries use `suspend`, while observable reads can use `Flow`. citeturn172874view3

For now, use:

```kotlin
suspend
```

for basic insert/read/delete DAO functions.

This means we can call them inside:

```kotlin
viewModelScope.launch {
    ...
}
```

or inside repository functions that are also `suspend`.

---

## 10. Database class

Now we need the Room database class.

Create:

```kotlin
@Database(
    entities = [Measurement::class],
    version = 1
)
abstract class ResearchDatabase : RoomDatabase() {
    abstract fun measurementDao(): MeasurementDao
}
```

Required imports:

```kotlin
import androidx.room3.Database
import androidx.room3.RoomDatabase
```

This means:

```text
This database contains the Measurement table.
The current database version is 1.
The database can provide a MeasurementDao.
```

This part:

```kotlin
entities = [Measurement::class]
```

tells Room:

```text
The Measurement entity belongs to this database.
```

This part:

```kotlin
abstract fun measurementDao(): MeasurementDao
```

means:

```text
Give me access to measurement database operations.
```

---

## 11. How the pieces connect

Now the Room structure is:

```text
Measurement
 ↓
Entity/table

MeasurementDao
 ↓
insert/read/delete measurements

ResearchDatabase
 ↓
main database object that gives access to MeasurementDao
```

The flow is:

```text
Repository
 ↓
MeasurementDao
 ↓
ResearchDatabase
 ↓
SQLite database on the tablet
```

So the full architecture is becoming:

```text
ResearchScreen
 ↓
ResearchViewModel
 ↓
MeasurementRepository
 ↓
MeasurementDao
 ↓
ResearchDatabase
 ↓
local database file
```

This is much more realistic.

---

## 12. Creating the database instance

To use the database, we need to create it.

For a beginner version, we can do this inside the repository.

```kotlin
class MeasurementRepository(
    private val context: Context
) {
    private val database = Room.databaseBuilder(
        context.applicationContext,
        ResearchDatabase::class.java,
        "research_database"
    ).build()

    private val measurementDao = database.measurementDao()

    suspend fun insertMeasurement(
        measurement: Measurement
    ) {
        measurementDao.insertMeasurement(measurement)
    }

    suspend fun getAllMeasurements(): List<Measurement> {
        return measurementDao.getAllMeasurements()
    }

    suspend fun deleteAllMeasurements() {
        measurementDao.deleteAllMeasurements()
    }
}
```

Required imports:

```kotlin
import android.content.Context
import androidx.room3.Room
```

Notice this:

```kotlin
context.applicationContext
```

This is safer than storing an Activity context.

In Lesson 13, we already mentioned that storing Activity-related context in long-lived classes can be problematic. For database creation, using application context is the safer beginner-friendly option.

---

## 13. Important change from Lesson 13

In Lesson 13, the repository had this style:

```kotlin
class MeasurementRepository {

    suspend fun saveMeasurements(
        context: Context,
        measurements: List<Measurement>
    ) {
        ...
    }

    suspend fun loadMeasurements(
        context: Context
    ): List<Measurement> {
        ...
    }
}
```

Now with Room, the repository is created with a `Context` once:

```kotlin
class MeasurementRepository(
    context: Context
) {
    ...
}
```

Then the repository functions no longer need `context` every time:

```kotlin
suspend fun insertMeasurement(measurement: Measurement)

suspend fun getAllMeasurements(): List<Measurement>
```

This is cleaner.

The repository owns the database connection.

---

## 14. But how do we create the repository in the ViewModel?

Here we meet one beginner complication.

Previously, our ViewModel could simply do:

```kotlin
private val measurementRepository = MeasurementRepository()
```

But now the repository needs:

```kotlin
context: Context
```

A normal `ViewModel` should not directly receive Activity context.

For a simple learning app, we can use `AndroidViewModel`, which gives access to the application.

```kotlin
class ResearchViewModel(
    application: Application
) : AndroidViewModel(application) {

    private val measurementRepository =
        MeasurementRepository(application)

    var uiState by mutableStateOf(ResearchUiState())
        private set

    ...
}
```

Required imports:

```kotlin
import android.app.Application
import androidx.lifecycle.AndroidViewModel
```

This is not the only way.

A more professional app may use dependency injection later.

But for this tutorial, this is a practical beginner-friendly path.

The mental model is:

```text
Application context
 ↓
Repository
 ↓
Room database
```

---

## 15. Insert measurement into Room

In Lesson 13, after creating a measurement, we saved the whole list.

With Room, we can insert only the new measurement.

Previously:

```text
create new measurement
 ↓
append to list
 ↓
save entire list to file
```

Now:

```text
create new measurement
 ↓
insert only this new measurement into Room
 ↓
update UI state
```

Inside the ViewModel:

```kotlin
private fun addSimulatedMeasurement() {
    val newMeasurement = measurementRepository.createSimulatedMeasurement(
        sampleId = uiState.sampleId,
        repetition = uiState.measurements.size + 1
    )

    viewModelScope.launch {
        try {
            measurementRepository.insertMeasurement(newMeasurement)

            val updatedMeasurements =
                measurementRepository.getAllMeasurements()

            uiState = uiState.copy(
                measurements = updatedMeasurements,
                latestValue = newMeasurement.value,
                message = "Measurement saved to database"
            )
        } catch (e: Exception) {
            uiState = uiState.copy(
                message = "Database save failed"
            )
        }
    }
}
```

But this assumes the repository still has:

```kotlin
createSimulatedMeasurement(...)
```

So we can keep that function in the repository for now.

---

## 16. Updated repository with simulation and Room

A simple Lesson 14 repository can look like this:

```kotlin
class MeasurementRepository(
    context: Context
) {
    private val database = Room.databaseBuilder(
        context.applicationContext,
        ResearchDatabase::class.java,
        "research_database"
    ).build()

    private val measurementDao = database.measurementDao()

    fun createSimulatedMeasurement(
        sampleId: String,
        repetition: Int
    ): Measurement {
        val value = Random.nextDouble(0.0, 5.0)

        return Measurement(
            sampleId = sampleId,
            repetition = repetition,
            value = value,
            timestamp = System.currentTimeMillis(),
            status = "OK"
        )
    }

    suspend fun insertMeasurement(
        measurement: Measurement
    ) {
        measurementDao.insertMeasurement(measurement)
    }

    suspend fun getAllMeasurements(): List<Measurement> {
        return measurementDao.getAllMeasurements()
    }

    suspend fun deleteAllMeasurements() {
        measurementDao.deleteAllMeasurements()
    }
}
```

Imports:

```kotlin
import android.content.Context
import androidx.room3.Room
import kotlin.random.Random
```

Now the repository handles:

```text
creating fake data
inserting measurement
reading all measurements
deleting measurements
```

The ViewModel does not know the SQL details.

---

## 17. Load measurements from Room when app starts

Previously, we loaded from internal storage.

Now we load from Room:

```kotlin
fun loadSavedMeasurements() {
    viewModelScope.launch {
        uiState = uiState.copy(
            isLoading = true,
            message = "Loading measurements from database..."
        )

        try {
            val loadedMeasurements =
                measurementRepository.getAllMeasurements()

            uiState = uiState.copy(
                measurements = loadedMeasurements,
                latestValue = loadedMeasurements.lastOrNull()?.value,
                isLoading = false,
                message = "Loaded ${loadedMeasurements.size} measurements"
            )
        } catch (e: Exception) {
            uiState = uiState.copy(
                isLoading = false,
                message = "Could not load database measurements"
            )
        }
    }
}
```

Notice:

```kotlin
loadedMeasurements.lastOrNull()?.value
```

This means:

```text
If there is at least one saved measurement,
show the latest value.

If there are no saved measurements,
latestValue stays null.
```

Again, this uses Kotlin null safety.

---

## 18. Start acquisition with Room

In Lesson 12, `startAcquisition()` had:

```kotlin
fun startAcquisition(context: Context)
```

because we needed context for file saving.

With Room inside the repository, we no longer need to pass context into every acquisition function.

So now:

```kotlin
fun startAcquisition() {
    if (uiState.deviceConnectionState != DeviceConnectionState.CONNECTED) {
        uiState = uiState.copy(
            message = "Connect device before starting acquisition"
        )
        return
    }

    if (uiState.acquisitionState == AcquisitionState.RECORDING) {
        return
    }

    uiState = uiState.copy(
        acquisitionState = AcquisitionState.RECORDING,
        message = "Acquisition started"
    )

    viewModelScope.launch {
        while (uiState.acquisitionState == AcquisitionState.RECORDING) {
            addSimulatedMeasurement()
            delay(1000)
        }
    }
}
```

This is cleaner than before.

The UI button can now call:

```kotlin
viewModel.startAcquisition()
```

instead of:

```kotlin
viewModel.startAcquisition(context)
```

This is a nice result of moving storage into the repository.

---

## 19. Full ViewModel pattern after adding Room

Here is the main pattern:

```kotlin
class ResearchViewModel(
    application: Application
) : AndroidViewModel(application) {

    private val measurementRepository =
        MeasurementRepository(application)

    var uiState by mutableStateOf(ResearchUiState())
        private set

    fun updateSampleId(newSampleId: String) {
        uiState = uiState.copy(
            sampleId = newSampleId
        )
    }

    fun connectDevice() {
        uiState = uiState.copy(
            deviceConnectionState = DeviceConnectionState.CONNECTING,
            message = "Connecting to device..."
        )

        viewModelScope.launch {
            delay(1000)

            uiState = uiState.copy(
                deviceConnectionState = DeviceConnectionState.CONNECTED,
                message = "Device connected"
            )
        }
    }

    fun disconnectDevice() {
        if (uiState.acquisitionState == AcquisitionState.RECORDING) {
            stopAcquisition()
        }

        uiState = uiState.copy(
            deviceConnectionState = DeviceConnectionState.DISCONNECTED,
            acquisitionState = AcquisitionState.IDLE,
            message = "Device disconnected"
        )
    }

    fun startAcquisition() {
        if (uiState.deviceConnectionState != DeviceConnectionState.CONNECTED) {
            uiState = uiState.copy(
                message = "Connect device before starting acquisition"
            )
            return
        }

        if (uiState.acquisitionState == AcquisitionState.RECORDING) {
            return
        }

        uiState = uiState.copy(
            acquisitionState = AcquisitionState.RECORDING,
            message = "Acquisition started"
        )

        viewModelScope.launch {
            while (uiState.acquisitionState == AcquisitionState.RECORDING) {
                addSimulatedMeasurement()
                delay(1000)
            }
        }
    }

    fun stopAcquisition() {
        if (uiState.acquisitionState != AcquisitionState.RECORDING) {
            return
        }

        uiState = uiState.copy(
            acquisitionState = AcquisitionState.STOPPED,
            message = "Acquisition stopped"
        )
    }

    fun loadSavedMeasurements() {
        viewModelScope.launch {
            uiState = uiState.copy(
                isLoading = true,
                message = "Loading measurements from database..."
            )

            try {
                val loadedMeasurements =
                    measurementRepository.getAllMeasurements()

                uiState = uiState.copy(
                    measurements = loadedMeasurements,
                    latestValue = loadedMeasurements.lastOrNull()?.value,
                    isLoading = false,
                    message = "Loaded ${loadedMeasurements.size} measurements"
                )
            } catch (e: Exception) {
                uiState = uiState.copy(
                    isLoading = false,
                    message = "Could not load database measurements"
                )
            }
        }
    }

    private fun addSimulatedMeasurement() {
        val newMeasurement =
            measurementRepository.createSimulatedMeasurement(
                sampleId = uiState.sampleId,
                repetition = uiState.measurements.size + 1
            )

        viewModelScope.launch {
            try {
                measurementRepository.insertMeasurement(newMeasurement)

                val updatedMeasurements =
                    measurementRepository.getAllMeasurements()

                uiState = uiState.copy(
                    measurements = updatedMeasurements,
                    latestValue = newMeasurement.value,
                    message = "Measurement saved to database"
                )
            } catch (e: Exception) {
                uiState = uiState.copy(
                    message = "Database save failed"
                )
            }
        }
    }
}
```

Important imports:

```kotlin
import android.app.Application
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.setValue
import androidx.lifecycle.AndroidViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch
```

## 20. Calling `loadSavedMeasurements()` from the UI

Previously, you may have used LaunchedEffect to load saved data.
That pattern can stay.
Inside ResearchScreen:

```kotlin
LaunchedEffect(Unit) {
    viewModel.loadSavedMeasurements()
}
```

This means:
When this screen first appears,
ask the ViewModel to load saved measurements.
The function now loads from Room rather than from an internal CSV-style file.

## 21. UI Button Changes

Because startAcquisition() no longer needs context, the button becomes simpler:

```kotlin
Button(
    onClick = {
        viewModel.startAcquisition()
    },
    enabled =
        uiState.deviceConnectionState == DeviceConnectionState.CONNECTED &&
        uiState.acquisitionState != AcquisitionState.RECORDING
) {
    Text("Start Acquisition")
}
```

Stop button:

```kotlin
Button(
    onClick = {
        viewModel.stopAcquisition()
    },
    enabled = uiState.acquisitionState == AcquisitionState.RECORDING
) {
    Text("Stop Acquisition")
}
```

The UI does not know whether data is saved to:

- file
- Room database
- cloud
- memory

The UI only talks to the ViewModel.
That is good architecture.

## 22. Add a Clear Database Button

For learning, it is useful to clear the table.
In the ViewModel:

```kotlin
fun clearMeasurements() {
    viewModelScope.launch {
        try {
            measurementRepository.deleteAllMeasurements()

            uiState = uiState.copy(
                measurements = emptyList(),
                latestValue = null,
                message = "All measurements deleted"
            )
        } catch (e: Exception) {
            uiState = uiState.copy(
                message = "Could not delete measurements"
            )
        }
    }
}
```

In the UI:

```kotlin
Button(
    onClick = {
        viewModel.clearMeasurements()
    },
    enabled = uiState.acquisitionState != AcquisitionState.RECORDING
) {
    Text("Clear Measurements")
}
```

We disable it while recording because deleting data during acquisition is unsafe.
The logic is:

> Do not allow destructive actions while recording.

This is a good research-app habit.

## 23. What About CSV Export Now?

Even after adding Room, CSV export is still useful.
The new flow becomes:

```text
Room database
 ↓
stores all measurements locally

Export CSV button
 ↓
reads selected measurements from Room
 ↓
writes them to a CSV file
 ↓
user analyses the CSV in Excel/Python
```

So the app architecture becomes:

```text
MeasurementRepository
 ├── insert into Room
 ├── read from Room
 ├── delete from Room
 └── later export selected data
```

Room is not mainly for external analysis.
Room is for reliable local app storage.
CSV is still better for sharing with Python, Excel, or collaborators.

## 24. Does Room Replace Internal Storage?

For our simple measurement history, yes, Room can replace the internal auto-save file.
Before Room:

```text
measurement list
 ↓
save whole list to internal CSV-style file
 ↓
reload whole list on app start
```

With Room:

```text
new measurement
 ↓
insert one row into database
 ↓
query rows when needed
```

This is better as the app grows.
However, internal storage is still useful for other things, such as:

- exported CSV files
- temporary files
- raw binary logs
- model files
- configuration files

So the idea is not:

> Room good, files bad

The better idea is:

> Use the right storage for the right purpose.

## 25. Beginner Warning About Migrations

In our database class, we wrote:

```kotlin
@Database(
    entities = [Measurement::class],
    version = 1
)
```

The version number matters.
Later, if you change the database structure, for example by adding:

```kotlin
val deviceId: String
```

or:

```kotlin
val sessionId: Long
```

then the database schema changes.
Real apps need database migrations.
Room’s documentation lists streamlined database migration paths as one of its benefits. Android Developers
But for this beginner lesson, we will not handle migrations yet.
For learning, if you change the entity and the app complains, you can often uninstall the app from the emulator/tablet and reinstall it.
For real research deployment, do not rely on uninstalling the app, because that would delete collected data.
## 26. Current Architecture After Lesson 14

After adding Room, our architecture becomes:

```text
Presentation layer
ResearchScreen
 ↓
ViewModel layer
ResearchViewModel
 ↓
Repository layer
MeasurementRepository
 ↓
Database access layer
MeasurementDao
 ↓
Local database
ResearchDatabase / Room / SQLite
```

This is a major step toward a real research app.
We now have:

- UI state
- acquisition state
- repository
- local structured database
- simulated measurement stream

That is much more serious than only having a button that generates a random value.

## 27. What You Learned in Lesson 14

The key concepts are:

- `@Entity` marks a Kotlin data class as a database table.
- `@PrimaryKey(autoGenerate = true)` marks a unique ID column.
- `@Dao` marks an interface that defines database operations.
- `@Insert` inserts an entity into the database.
- `@Query("SELECT * FROM measurements")` reads data from the database.
- `@Database(entities = [Measurement::class], version = 1)` defines the Room database.

The most important mental model is:

```text
Entity = table structure
DAO = database operations
Database = main Room object
Repository = app-facing data manager
ViewModel = screen state and user flow
UI = display and buttons
```

For a research app:

- Room stores structured data locally.
- CSV exports data for analysis outside the app.

## Lesson 15 Preview

In Lesson 15, we should move from:

```text
one flat Measurement table
                ↓
a more realistic research data model:
Patient
Session
Measurement
Result
```

This is important because your final app should probably not only save values.
It should know:

- which patient/sample the data belongs to
- which session the measurement belongs to
- when the session started and ended
- what result or prediction was produced

So Lesson 15 will cover:

- Patient data class
- Session data class
- Measurement belongs to Session
- Session belongs to Patient
- basic relationship thinking
- why data modelling matters before app complexity grows
