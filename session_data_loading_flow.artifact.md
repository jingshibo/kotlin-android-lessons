# Session Data Loading Architecture Flow

This document details the complete technical code chain that controls auto-loading and displaying session records when the user selects the **Repository → Sessions** tab.

---

## 1. Step-by-Step Code Chain Diagram

```text
[ User Taps Sessions Tab ]
          │
          ▼
1. SessionsScreen.kt ─────► Calls: val mappedSessions = viewModel.getMappedSessions()
          │
          ▼
2. SessionsViewModel.kt ──► init { viewModelScope.launch { sessionRepository.sessionsFlow.collect { ... } } }
          │
          ▼
3. SessionRepository.kt ──► val sessionsFlow = sessionDao.getAllSessionsWithDetails().map { ... }
          │
          ▼
4. SessionDao.kt ────────► @Transaction @Query("SELECT * FROM sessions ORDER BY recording_day DESC")
```

---

## 2. Detailed Code Implementation

### Step 1: UI Layer (`SessionsScreen.kt`)
When the Sessions tab renders, `SessionsScreen` requests the mapped session list from `SessionsViewModel`:

```kotlin
// Location: ui/screens/repository/sessions/SessionsScreen.kt
@Composable
fun SessionsScreen(
    viewModel: SessionsViewModel = viewModel()
) {
    // Fetches mapped sessions for table rendering
    val mappedSessions = viewModel.getMappedSessions()

    // Renders table rows in HistoryComponents.kt
    HistoryTable(sessions = mappedSessions)
}
```

---

### Step 2: ViewModel Layer (`SessionsViewModel.kt`)
When `SessionsViewModel` is instantiated on tab selection, its `init` block automatically starts listening to the reactive database stream:

```kotlin
// Location: viewmodel/repository/SessionsViewModel.kt
class SessionsViewModel(
    private val sessionRepository: SessionRepository = SessionRepository.instance
) : ViewModel() {

    init {
        // 1. Seeds initial sample records into Room SQLite if database is empty on first launch
        viewModelScope.launch {
            sessionRepository.seedInitialDataIfEmpty()
        }

        // 2. Collects real-time Room DB Flow emissions into the UI state list
        val flow = sessionRepository.sessionsFlow
        if (flow != null) {
            viewModelScope.launch {
                flow.collect { dbSessions ->
                    if (dbSessions.isNotEmpty()) {
                        sessionRepository.sessions.clear()
                        sessionRepository.sessions.addAll(dbSessions) // Auto-triggers Composable re-render!
                    }
                }
            }
        }
    }

    fun getMappedSessions(): List<SessionDetailsUi> {
        return sessionRepository.sessions.map { session ->
            // Maps SessionRecord to UI presentation model
            SessionDetailsUi(...)
        }
    }
}
```

---

### Step 3: Repository Layer (`SessionRepository.kt`)
`SessionRepository.sessionsFlow` listens to `SessionDao.getAllSessionsWithDetails()` and maps raw database entities into strongly-typed `SessionRecord` domain models:

```kotlin
// Location: data/repository/SessionRepository.kt
class SessionRepository(
    private val sessionDao: SessionDao? = null
) {
    /**
     * Reactive stream of all measurement sessions directly from Room SQLite database.
     */
    val sessionsFlow: Flow<List<SessionRecord>>? = sessionDao?.getAllSessionsWithDetails()
        ?.map { list -> list.map { it.toDomainModel() } }
}
```

---

### Step 4: Room SQLite Database Layer (`SessionDao.kt`)
Room executes a single atomic SQL transaction joining `SessionEntity` with its related `PatientEntity`, `DeviceEntity`, `ResultEntity`, and `MeasurementEntity` rows:

```kotlin
// Location: data/local/dao/SessionDao.kt
@Dao
interface SessionDao {

    /**
     * Reactive stream of all sessions with full details (patient, device, results, measurements).
     */
    @Transaction
    @Query("SELECT * FROM sessions ORDER BY recording_day DESC, session_id DESC")
    fun getAllSessionsWithDetails(): Flow<List<SessionWithDetails>>
}
```

---

## 3. Summary
1. **Trigger**: Selecting the Sessions tab instantiates `SessionsViewModel`.
2. **Auto-Load**: `SessionsViewModel.init` collects `sessionRepository.sessionsFlow`.
3. **Database Query**: Room executes `getAllSessionsWithDetails()` in SQLite and streams the joined records directly to `SessionsScreen.kt` for rendering!




Question: a question: when i created the database, i should initialize it when i install the app, right? how should i do it in general?
Not exactly. Your code does not run at the moment the user installs the app.

There are three separate moments:

```text
App installation
    -> Android installs the APK

App process starts
    -> your Application class runs

Database is first opened
    -> Room creates the database file and tables if they do not exist
```

Room databases are relatively expensive, so Android recommends creating **one database instance per app process** and reusing it. Use the application context rather than an Activity or composable context. [Official Room documentation](https://developer.android.com/training/data-storage/room)

### General setup

Define the database:

```kotlin
@Database(
    entities = [
        PatientEntity::class,
        SessionEntity::class,
        MeasurementEntity::class,
        ResultEntity::class
    ],
    version = 1
)
abstract class ResearchDatabase : RoomDatabase() {

    abstract fun patientDao(): PatientDao
    abstract fun sessionDao(): SessionDao
    abstract fun measurementDao(): MeasurementDao
    abstract fun resultDao(): ResultDao

    companion object {
        @Volatile
        private var INSTANCE: ResearchDatabase? = null

        fun getInstance(context: Context): ResearchDatabase {
            return INSTANCE ?: synchronized(this) {
                INSTANCE ?: Room.databaseBuilder(
                    context.applicationContext,
                    ResearchDatabase::class.java,
                    "research_database"
                ).build().also { database ->
                    INSTANCE = database
                }
            }
        }
    }
}
```

Then initialize the dependency from an `Application` class:

```kotlin
class ResearchApplication : Application() {

    val database: ResearchDatabase by lazy {
        ResearchDatabase.getInstance(this)
    }

    val repository: MeasurementRepository by lazy {
        MeasurementRepository(
            patientDao = database.patientDao()
        )
    }
}
```

Register it in `AndroidManifest.xml`:

```xml
<application
    android:name=".ResearchApplication"
    ... >
```

Because `database` uses `by lazy`, the database instance is created only when something first requests it:

```text
App starts
    -> ResearchApplication exists
    -> ViewModel needs repository
    -> repository needs DAO
    -> database instance is created
    -> first database access opens/creates the database
```

Do not call `Room.databaseBuilder()` inside a composable or create a new database instance for every screen.

### What happens later?

```text
App process stops and restarts
    -> create a new RoomDatabase instance
    -> reuse the existing database file

App is updated
    -> database remains
    -> schema changes require migrations

App is uninstalled or its storage is cleared
    -> database file is deleted
```

If you need initial rows, such as default study groups, use a Room creation callback or a prepackaged database. `RoomDatabase.Callback.onCreate()` runs only when the database is first created, after its tables are created. [Room callback documentation](https://developer.android.com/reference/androidx/room/RoomDatabase.Callback)

The central rule is:

> Initialize one reusable Room database instance at the application level, usually lazily. Room creates the actual database when it is first opened, not when the APK is installed.