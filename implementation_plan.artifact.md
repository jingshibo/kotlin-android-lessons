# Implementation Plan: Modular Repositories & Room Database Persistence

Split the monolithic repository into dedicated, domain-focused repositories connected directly to Room DAOs and Domain Mappers, adhering strictly to Clean Architecture / MVVM principles.

## Architectural Clarifications & Rules

1. **`MeasurementRepository` Isolation:** `MeasurementRepository.kt` will NOT be modified or used.
2. **Strict UI/ViewModel/Repository Separation:**
   - **UI Layer (Composables):** Only observes `StateFlow` from ViewModels and triggers ViewModel functions. **Zero direct access to Repositories.**
   - **ViewModel Layer:** Mediates between UI and Repositories.
   - **Repository Layer:** Connects to Room Database DAOs and converts entities to/from domain models using `domain/mapper`.
3. **Repository Modularization:**
   Instead of keeping all entities in a single class, split repository responsibilities into dedicated domain repositories in `com.example.researchdeviceui.data.repository`:
   - `PatientRepository` (Patient CRUD & Room DAO mapping)
   - `SessionRepository` (Session CRUD & Room DAO mapping)
   - `DeviceRepository` (Device CRUD & Room DAO mapping)
   - `ResultRepository` (Result CRUD & Room DAO mapping)

---

## Proposed Changes

### Repository Layer (`com.example.researchdeviceui.data.repository`)

#### [NEW] [PatientRepository.kt](file:///D:/Project/Endpoint_Android_App/app/src/main/java/com/example/researchdeviceui/data/repository/PatientRepository.kt)
- Uses `PatientDao` and `PatientMappers`:
  - `observePatients()` / `getPatients()` -> returns `List<PatientRecord>`
  - `findPatient(patientCode: String)` -> returns `PatientRecord?`
  - `addPatient(patient: PatientRecord)` / `updatePatient(patient: PatientRecord)`
  - `voidPatient(patientCode: String, reason: String)` / `restorePatient(patientCode: String)`

#### [NEW] [DeviceRepository.kt](file:///D:/Project/Endpoint_Android_App/app/src/main/java/com/example/researchdeviceui/data/repository/DeviceRepository.kt)
- Uses `DeviceDao` and `DeviceMappers`:
  - `observeDevices()` / `getDevices()` -> returns `List<DeviceRecord>`
  - `findDevice(deviceCode: String)` -> returns `DeviceRecord?`
  - `updateDevice(device: DeviceRecord)`

#### [NEW] [ResultRepository.kt](file:///D:/Project/Endpoint_Android_App/app/src/main/java/com/example/researchdeviceui/data/repository/ResultRepository.kt)
- Uses `ResultDao` and `ResultMappers`:
  - `saveResult(result: ResultRecord)`
  - `getResultsForSession(sessionId: Long)` -> returns `List<ResultRecord>`
  - `markResultExported(resultId: Long)`

#### [MODIFY] [SessionRepository.kt](file:///D:/Project/Endpoint_Android_App/app/src/main/java/com/example/researchdeviceui/data/repository/SessionRepository.kt)
- Refactor `SessionRepository` to focus strictly on `SessionRecord` persistence using `SessionDao` and `SessionMappers`:
  - `observeSessions()` / `getSessions()` -> returns `List<SessionRecord>`
  - `findSession(patientCode: String, recordingDate: String)` -> returns `SessionRecord?`
  - `saveSession(session: SessionRecord)`
  - `voidSession(sessionId: Long, reason: String)` / `restoreSession(sessionId: Long)`

#### Database Seeding / Initialization
- Include database initialization logic to seed initial sample records into Room on first application launch if the tables are empty, ensuring seamless transition.

---

## Verification Plan

### Automated Tests
- Run `:app:compileDebugKotlin` via Gradle to verify all repository classes, imports, and domain mapper usage compile cleanly.

### Manual Verification
- Verify clean separation of concerns:
  - Repository layer handles Room DAOs and `domain/mapper` conversion.
  - No UI file references repositories directly.
