# 4 Práctica Incremental — Reactividad, Robustez y Optimización

| Campo | Detalle |
|---|---|
| **Duración** | 252 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |

## Descripción General

Este laboratorio de cierre integra tres pilares profesionales en el proyecto **TaskManagerApp**: reactividad con Combine Framework, manejo exhaustivo de errores tipados y perfilado con Instruments. Partiendo del estado funcional alcanzado en el Lab 04, construirás pipelines reactivos para búsqueda en tiempo real y monitoreo de sincronización, auditarás el código para eliminar force-unwraps y centralizar errores, perfilarás la app para detectar y corregir problemas de rendimiento, y finalmente realizarás el merge a `main` con el tag `v1.0.0`.

## Objetivos de Aprendizaje

- [ ] Implementar reactividad con Combine: `Publishers`, `Subscribers` y operadores (`map`, `filter`, `debounce`, `sink`, `combineLatest`) para búsqueda en tiempo real de tareas.
- [ ] Construir un pipeline Combine robusto que combine el estado de red (`TaskNetworkService`) con el estado local (`SwiftDataTaskRepository`) usando `combineLatest` y `merge`.
- [ ] Implementar manejo de errores exhaustivo con `do/catch`, `Result<T, E>`, `throws` y propagación de errores tipados en toda la cadena de capas.
- [ ] Perfilar la aplicación con Instruments 15.4 identificando y corrigiendo al menos un problema de rendimiento real.
- [ ] Realizar el merge final a `main` con tag `v1.0.0`, completando el ciclo de desarrollo del proyecto incremental.

## Prerrequisitos

### Conocimientos Requeridos

| Concepto | Nivel |
|---|---|
| Swift 5.10 (closures, generics, protocols) | Intermedio |
| SwiftUI (vistas, estado, navegación) | Intermedio |
| async/await y URLSession | Intermedio |
| SwiftData (ModelContext, @Query) | Básico |
| Git branching y merging | Básico |
| Programación reactiva (Publisher/Subscriber) | Conceptual |

### Acceso Requerido

- Lab 04-00-01 completado con la capa de red funcional.
- Rama `develop` en GitHub con commits de Labs 01–04.
- App ejecutándose sin crashes en simulador iPhone 15 Pro / iOS 17.5.
- Repositorio `task-manager-ios` en GitHub (privado).

## Entorno de Laboratorio

### Hardware

| Recurso | Especificación |
|---|---|
| Procesador | Apple Silicon (M1/M2/M3) o Intel 8ª gen+ |
| RAM | 16 GB mínimo |
| Disco libre | 50 GB mínimo |
| Pantalla | 1440×900 mínimo (1920×1080 recomendado) |

### Software

| Herramienta | Versión |
|---|---|
| macOS | Ventura 13.6.7 o Sonoma 14.5 |
| Xcode | 15.4 |
| Swift | 5.10 |
| Simulador | iPhone 15 Pro – iOS 17.5 |
| Git | 2.45.2 |
| Instruments | 15.4 (incluido con Xcode) |

### Preparación del Entorno

```bash
# Verificar directorio del proyecto
cd ~/Developer/TaskManagerApp

# Verificar estado de Git
git status
git branch

# Asegurarse de estar en develop y actualizado
git checkout develop
git pull origin develop

# Crear rama para este lab
git checkout -b feature/combine-errors-instruments
```

**Verificación:** El comando `git log --oneline -5` debe mostrar commits de los Labs anteriores. La app debe compilar sin errores con ⌘B.

---

## Paso 1 — Crear el Enum Unificado de Errores (`AppError`)

**Objetivo:** Establecer un sistema de errores tipados que se usará en toda la aplicación antes de construir los pipelines Combine, garantizando que cada capa propague errores de forma explícita.

### Instrucciones

1. En Xcode, crea un nuevo grupo llamado `Errors` dentro de la carpeta `TaskManagerApp` del Project Navigator (clic derecho → New Group).

2. Dentro de `Errors`, crea un nuevo archivo Swift: **File → New → File → Swift File** → nombra `AppError.swift`.

3. Implementa el enum unificado de errores:

```swift
// AppError.swift
// TaskManagerApp

import Foundation

/// Enum unificado de errores para toda la aplicación.
/// Engloba errores de red, persistencia y validación.
enum AppError: Error, LocalizedError, Equatable {

    // MARK: - Network Errors
    case networkUnavailable
    case invalidURL(String)
    case serverError(statusCode: Int)
    case decodingFailed(String)
    case requestTimeout

    // MARK: - Persistence Errors
    case saveFailed(String)
    case fetchFailed(String)
    case deleteFailed(String)
    case modelNotFound(id: String)

    // MARK: - Validation Errors
    case emptyTitle
    case titleTooLong(maxLength: Int)
    case invalidPriority
    case duplicateTask(title: String)

    // MARK: - General
    case unknown(String)

    var errorDescription: String? {
        switch self {
        case .networkUnavailable:
            return "No hay conexión a Internet disponible."
        case .invalidURL(let url):
            return "URL inválida: \(url)"
        case .serverError(let code):
            return "Error del servidor (código \(code))."
        case .decodingFailed(let detail):
            return "Error al decodificar datos: \(detail)"
        case .requestTimeout:
            return "La solicitud excedió el tiempo de espera."
        case .saveFailed(let detail):
            return "Error al guardar: \(detail)"
        case .fetchFailed(let detail):
            return "Error al obtener datos: \(detail)"
        case .deleteFailed(let detail):
            return "Error al eliminar: \(detail)"
        case .modelNotFound(let id):
            return "No se encontró el elemento con ID: \(id)"
        case .emptyTitle:
            return "El título de la tarea no puede estar vacío."
        case .titleTooLong(let max):
            return "El título excede el máximo de \(max) caracteres."
        case .invalidPriority:
            return "La prioridad seleccionada no es válida."
        case .duplicateTask(let title):
            return "Ya existe una tarea con el título: \(title)"
        case .unknown(let detail):
            return "Error desconocido: \(detail)"
        }
    }
}
```

4. En el mismo grupo `Errors`, crea `ErrorHandler.swift`:

```swift
// ErrorHandler.swift
// TaskManagerApp

import Foundation
import Combine
import os.log

/// Servicio centralizado para manejo, logging y presentación de errores.
final class ErrorHandler: ObservableObject {

    static let shared = ErrorHandler()

    /// Publisher que emite errores para que la UI pueda reaccionar.
    @Published var currentError: AppError?
    @Published var showErrorAlert: Bool = false

    private let logger = Logger(
        subsystem: Bundle.main.bundleIdentifier ?? "com.cursounam.taskmanagerapp",
        category: "ErrorHandler"
    )

    private init() {}

    /// Registra y presenta un error.
    func handle(_ error: AppError, file: String = #file, line: Int = #line) {
        let fileName = (file as NSString).lastPathComponent
        logger.error("[\(fileName):\(line)] \(error.localizedDescription)")

        DispatchQueue.main.async { [weak self] in
            self?.currentError = error
            self?.showErrorAlert = true
        }
    }

    /// Convierte errores genéricos del sistema a AppError.
    func mapToAppError(_ error: Error) -> AppError {
        if let appError = error as? AppError {
            return appError
        }

        if let urlError = error as? URLError {
            switch urlError.code {
            case .notConnectedToInternet, .networkConnectionLost:
                return .networkUnavailable
            case .timedOut:
                return .requestTimeout
            default:
                return .unknown(urlError.localizedDescription)
            }
        }

        if error is DecodingError {
            return .decodingFailed(error.localizedDescription)
        }

        return .unknown(error.localizedDescription)
    }

    /// Resetea el estado de error (llamar al cerrar la alerta).
    func clearError() {
        currentError = nil
        showErrorAlert = false
    }
}
```

5. Compila el proyecto con **⌘B** para verificar que no hay errores.

6. Haz commit:

```bash
git add -A
git commit -m "feat: add AppError enum and ErrorHandler service for unified error handling"
```

### Resultado Esperado

Dos archivos nuevos en el grupo `Errors/`. El proyecto compila sin errores ni warnings.

### Verificación

- `AppError.emptyTitle.localizedDescription` devuelve `"El título de la tarea no puede estar vacío."`.
- `ErrorHandler.shared` es accesible desde cualquier punto de la app.

---

## Paso 2 — Implementar Validación de Formulario con `Result<T, E>`

**Objetivo:** Añadir validación robusta al formulario de creación de tareas usando `Result<TaskItem, AppError>` y reemplazar cualquier force-unwrap existente.

### Instrucciones

1. Crea un nuevo grupo `Validation` y dentro un archivo `TaskValidator.swift`:

```swift
// TaskValidator.swift
// TaskManagerApp

import Foundation

/// Validador de datos de tareas antes de la persistencia.
struct TaskValidator {

    static let maxTitleLength = 100

    /// Valida los campos de entrada y retorna un Result.
    /// - Parameters:
    ///   - title: Título de la tarea.
    ///   - notes: Notas opcionales.
    /// - Returns: Result con los datos validados o un AppError.
    static func validate(
        title: String,
        notes: String? = nil
    ) -> Result<ValidatedTaskData, AppError> {

        let trimmedTitle = title.trimmingCharacters(in: .whitespacesAndNewlines)

        guard !trimmedTitle.isEmpty else {
            return .failure(.emptyTitle)
        }

        guard trimmedTitle.count <= maxTitleLength else {
            return .failure(.titleTooLong(maxLength: maxTitleLength))
        }

        let validatedData = ValidatedTaskData(
            title: trimmedTitle,
            notes: notes?.trimmingCharacters(in: .whitespacesAndNewlines)
        )

        return .success(validatedData)
    }
}

/// Datos validados listos para crear un TaskItem.
struct ValidatedTaskData {
    let title: String
    let notes: String?
}
```

2. Ahora integra la validación en tu ViewModel de creación de tareas. Abre (o crea) `AddTaskViewModel.swift`:

```swift
// AddTaskViewModel.swift
// TaskManagerApp

import Foundation
import Combine

final class AddTaskViewModel: ObservableObject {

    @Published var title: String = ""
    @Published var notes: String = ""
    @Published var isFormValid: Bool = false
    @Published var validationMessage: String = ""

    private var cancellables = Set<AnyCancellable>()

    init() {
        // Pipeline reactivo: valida el título en tiempo real
        $title
            .debounce(for: .milliseconds(200), scheduler: RunLoop.main)
            .removeDuplicates()
            .map { title -> (Bool, String) in
                let result = TaskValidator.validate(title: title)
                switch result {
                case .success:
                    return (true, "")
                case .failure(let error):
                    // No mostrar error si el campo está vacío (aún no ha escrito)
                    if title.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty {
                        return (false, "")
                    }
                    return (false, error.localizedDescription)
                }
            }
            .sink { [weak self] isValid, message in
                self?.isFormValid = isValid
                self?.validationMessage = message
            }
            .store(in: &cancellables)
    }

    /// Intenta crear la tarea. Retorna Result para que la vista reaccione.
    func createTask() -> Result<ValidatedTaskData, AppError> {
        let result = TaskValidator.validate(title: title, notes: notes)

        if case .failure(let error) = result {
            ErrorHandler.shared.handle(error)
        }

        return result
    }
}
```

3. Actualiza `AddTaskView.swift` (SwiftUI) para usar el ViewModel con validación:

```swift
// AddTaskView.swift
// TaskManagerApp

import SwiftUI
import SwiftData

struct AddTaskView: View {

    @Environment(\.dismiss) private var dismiss
    @Environment(\.modelContext) private var modelContext
    @StateObject private var viewModel = AddTaskViewModel()

    var body: some View {
        NavigationStack {
            Form {
                Section("Detalles de la Tarea") {
                    TextField("Título", text: $viewModel.title)
                        .autocorrectionDisabled()

                    if !viewModel.validationMessage.isEmpty {
                        Text(viewModel.validationMessage)
                            .font(.caption)
                            .foregroundStyle(.red)
                    }

                    TextField("Notas (opcional)", text: $viewModel.notes, axis: .vertical)
                        .lineLimit(3...6)
                }
            }
            .navigationTitle("Nueva Tarea")
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("Cancelar") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("Guardar") {
                        guardarTarea()
                    }
                    .disabled(!viewModel.isFormValid)
                }
            }
        }
    }

    private func guardarTarea() {
        let result = viewModel.createTask()

        switch result {
        case .success(let data):
            // Crear TaskItem usando SwiftData
            let newTask = TaskItem(title: data.title, notes: data.notes)
            modelContext.insert(newTask)

            do {
                try modelContext.save()
                dismiss()
            } catch {
                let appError = AppError.saveFailed(error.localizedDescription)
                ErrorHandler.shared.handle(appError)
            }

        case .failure:
            // El error ya fue manejado por el ViewModel vía ErrorHandler
            break
        }
    }
}
```

> **Nota:** Si tu modelo `TaskItem` tiene una firma de inicializador diferente (por ejemplo, con parámetros adicionales de Labs anteriores), ajusta la llamada `TaskItem(title:notes:)` según corresponda.

4. **Auditoría de force-unwraps:** Busca en todo el proyecto cualquier uso de `!` (force-unwrap) y `try!`:

```bash
# Desde la terminal, en el directorio del proyecto
cd ~/Developer/TaskManagerApp
grep -rn "try!" --include="*.swift" TaskManagerApp/
grep -rn '\![^=]' --include="*.swift" TaskManagerApp/ | grep -v "//\|///\|!=\|!="
```

Reemplaza cada ocurrencia con manejo explícito (`guard let`, `if let`, `do/catch`). Documenta cada cambio.

5. Compila y haz commit:

```bash
git add -A
git commit -m "feat: add TaskValidator with Result type and reactive form validation"
```

### Resultado Esperado

- El formulario muestra mensajes de validación en tiempo real conforme el usuario escribe.
- El botón "Guardar" está deshabilitado hasta que la validación pasa.
- No quedan `try!` ni force-unwraps injustificados en el proyecto.

### Verificación

- Escribe un título de más de 100 caracteres → aparece mensaje de error y el botón permanece deshabilitado.
- Deja el título vacío y escribe solo espacios → el botón permanece deshabilitado.
- Escribe un título válido → el botón se habilita tras ~200ms de debounce.

---

## Paso 3 — Construir el `SearchViewModel` con Pipeline Combine

**Objetivo:** Implementar búsqueda en tiempo real de tareas usando un pipeline Combine completo con `debounce`, `removeDuplicates` y `map`.

### Instrucciones

1. Crea el archivo `SearchViewModel.swift` en el grupo `ViewModels`:

```swift
// SearchViewModel.swift
// TaskManagerApp

import Foundation
import Combine
import SwiftData

final class SearchViewModel: ObservableObject {

    // MARK: - Inputs
    @Published var searchText: String = ""

    // MARK: - Outputs
    @Published var filteredTasks: [TaskItem] = []
    @Published var isSearching: Bool = false

    // MARK: - Dependencies
    private var allTasks: [TaskItem] = []
    private var cancellables = Set<AnyCancellable>()

    init() {
        setupSearchPipeline()
    }

    /// Configura el pipeline Combine para búsqueda reactiva.
    private func setupSearchPipeline() {
        $searchText
            .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
            .removeDuplicates()
            .handleEvents(receiveOutput: { [weak self] query in
                self?.isSearching = !query.isEmpty
            })
            .map { [weak self] query -> [TaskItem] in
                guard let self = self else { return [] }

                let trimmed = query.trimmingCharacters(in: .whitespacesAndNewlines)

                guard !trimmed.isEmpty else {
                    return self.allTasks
                }

                return self.allTasks.filter { task in
                    task.title.localizedCaseInsensitiveContains(trimmed) ||
                    (task.notes?.localizedCaseInsensitiveContains(trimmed) ?? false)
                }
            }
            .receive(on: RunLoop.main)
            .sink { [weak self] results in
                self?.filteredTasks = results
                if self?.searchText.isEmpty == true {
                    self?.isSearching = false
                }
            }
            .store(in: &cancellables)
    }

    /// Actualiza la fuente de datos completa.
    /// Llamar cada vez que cambie la lista de tareas en SwiftData.
    func updateTasks(_ tasks: [TaskItem]) {
        self.allTasks = tasks
        // Re-trigger del filtro con el texto actual
        let current = searchText
        searchText = current
    }
}
```

2. Integra la barra de búsqueda en `TaskListView.swift` usando el modificador `.searchable`:

```swift
// TaskListView.swift (actualización)
// TaskManagerApp

import SwiftUI
import SwiftData

struct TaskListView: View {

    @Environment(\.modelContext) private var modelContext
    @Query(sort: \TaskItem.createdAt, order: .reverse) private var tasks: [TaskItem]

    @StateObject private var searchVM = SearchViewModel()
    @State private var showAddTask = false

    var body: some View {
        NavigationStack {
            List {
                ForEach(displayedTasks) { task in
                    TaskRowView(task: task)
                }
                .onDelete(perform: deleteTasks)
            }
            .navigationTitle("Mis Tareas")
            .searchable(
                text: $searchVM.searchText,
                placement: .navigationBarDrawer(displayMode: .always),
                prompt: "Buscar tareas..."
            )
            .overlay {
                if displayedTasks.isEmpty {
                    ContentUnavailableView.search(text: searchVM.searchText)
                }
            }
            .toolbar {
                ToolbarItem(placement: .primaryAction) {
                    Button(action: { showAddTask = true }) {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $showAddTask) {
                AddTaskView()
            }
            .onChange(of: tasks) { _, newTasks in
                searchVM.updateTasks(newTasks)
            }
            .onAppear {
                searchVM.updateTasks(tasks)
            }
        }
    }

    /// Tareas a mostrar: filtradas si hay búsqueda, todas si no.
    private var displayedTasks: [TaskItem] {
        searchVM.searchText.isEmpty ? tasks : searchVM.filteredTasks
    }

    private func deleteTasks(at offsets: IndexSet) {
        for index in offsets {
            let task = displayedTasks[index]
            modelContext.delete(task)
        }
        do {
            try modelContext.save()
        } catch {
            ErrorHandler.shared.handle(.deleteFailed(error.localizedDescription))
        }
    }
}
```

> **Nota:** Si tu `TaskRowView` tiene un nombre o firma diferente, ajústalo. Lo importante es que la lista use `displayedTasks` y la barra de búsqueda esté conectada al `searchVM.searchText`.

3. Compila y ejecuta en el simulador iPhone 15 Pro (⌘R). Crea 3–4 tareas de prueba y verifica que la búsqueda filtra en tiempo real.

4. Haz commit:

```bash
git add -A
git commit -m "feat: implement SearchViewModel with Combine debounce pipeline and .searchable"
```

### Resultado Esperado

Al escribir en la barra de búsqueda, la lista se filtra tras 300ms de pausa. Al borrar el texto, se muestran todas las tareas. Si no hay resultados, aparece `ContentUnavailableView.search`.

### Verificación

- Escribe "ta" → solo aparecen tareas cuyo título o notas contienen "ta".
- Escribe y borra rápidamente → no se producen filtrados innecesarios (debounce).
- Verifica en la consola de Xcode que no hay warnings ni errores.

---

## Paso 4 — Pipeline de Estado de Sincronización con `combineLatest`

**Objetivo:** Combinar el estado de red (`TaskNetworkService`) con el estado local para mostrar un banner de sincronización usando `combineLatest` y `CurrentValueSubject`.

### Instrucciones

1. Crea `SyncStatusViewModel.swift` en el grupo `ViewModels`:

```swift
// SyncStatusViewModel.swift
// TaskManagerApp

import Foundation
import Combine

/// Representa el estado de sincronización entre datos locales y remotos.
enum SyncStatus: Equatable {
    case idle
    case syncing
    case synced(Date)
    case offline
    case error(String)

    var displayText: String {
        switch self {
        case .idle:
            return "Listo"
        case .syncing:
            return "Sincronizando..."
        case .synced(let date):
            let formatter = RelativeDateTimeFormatter()
            formatter.locale = Locale(identifier: "es_MX")
            return "Sincronizado \(formatter.localizedString(for: date, relativeTo: .now))"
        case .offline:
            return "Sin conexión"
        case .error(let msg):
            return "Error: \(msg)"
        }
    }

    var iconName: String {
        switch self {
        case .idle: return "checkmark.circle"
        case .syncing: return "arrow.triangle.2.circlepath"
        case .synced: return "checkmark.icloud"
        case .offline: return "wifi.slash"
        case .error: return "exclamationmark.triangle"
        }
    }
}

final class SyncStatusViewModel: ObservableObject {

    // MARK: - Outputs
    @Published var syncStatus: SyncStatus = .idle

    // MARK: - Subjects (puentes imperativos)
    /// Indica si hay conexión de red disponible.
    let networkStatusSubject = CurrentValueSubject<Bool, Never>(true)

    /// Indica el número de tareas locales pendientes de sincronizar.
    let localPendingCountSubject = CurrentValueSubject<Int, Never>(0)

    /// Subject para disparar sincronización manualmente.
    let syncTrigger = PassthroughSubject<Void, Never>()

    private var cancellables = Set<AnyCancellable>()

    init() {
        setupSyncPipeline()
    }

    private func setupSyncPipeline() {
        // Pipeline 1: Combinar estado de red + tareas pendientes
        networkStatusSubject
            .combineLatest(localPendingCountSubject)
            .map { isOnline, pendingCount -> SyncStatus in
                if !isOnline {
                    return .offline
                }
                if pendingCount > 0 {
                    return .syncing
                }
                return .synced(Date())
            }
            .removeDuplicates()
            .receive(on: RunLoop.main)
            .assign(to: &$syncStatus)

        // Pipeline 2: Trigger manual de sincronización
        syncTrigger
            .filter { [weak self] in
                self?.networkStatusSubject.value == true
            }
            .sink { [weak self] in
                self?.performSync()
            }
            .store(in: &cancellables)
    }

    /// Ejecuta la sincronización con el servicio de red.
    private func performSync() {
        syncStatus = .syncing

        // Simular llamada de red usando TaskNetworkService existente
        Task { [weak self] in
            do {
                // Usa tu TaskNetworkService del Lab 04
                // let remoteTasks = try await TaskNetworkService.shared.fetchTasks()
                // Procesar y mergear con datos locales...

                await MainActor.run {
                    self?.localPendingCountSubject.send(0)
                    self?.syncStatus = .synced(Date())
                }
            } catch {
                await MainActor.run {
                    let appError = ErrorHandler.shared.mapToAppError(error)
                    self?.syncStatus = .error(appError.localizedDescription)
                    ErrorHandler.shared.handle(appError)
                }
            }
        }
    }

    // MARK: - Métodos públicos para actualizar estado

    func updateNetworkStatus(isOnline: Bool) {
        networkStatusSubject.send(isOnline)
    }

    func updatePendingCount(_ count: Int) {
        localPendingCountSubject.send(count)
    }

    func triggerSync() {
        syncTrigger.send()
    }
}
```

2. Crea el componente de UI `SyncBannerView.swift`:

```swift
// SyncBannerView.swift
// TaskManagerApp

import SwiftUI

struct SyncBannerView: View {

    @ObservedObject var viewModel: SyncStatusViewModel

    var body: some View {
        HStack(spacing: 8) {
            Image(systemName: viewModel.syncStatus.iconName)
                .font(.caption)
                .foregroundStyle(bannerColor)

            Text(viewModel.syncStatus.displayText)
                .font(.caption)
                .foregroundStyle(.secondary)

            Spacer()

            if viewModel.syncStatus == .offline || viewModel.syncStatus != .syncing {
                Button {
                    viewModel.triggerSync()
                } label: {
                    Image(systemName: "arrow.clockwise")
                        .font(.caption)
                }
                .disabled(viewModel.syncStatus == .offline)
            }

            if viewModel.syncStatus == .syncing {
                ProgressView()
                    .controlSize(.mini)
            }
        }
        .padding(.horizontal)
        .padding(.vertical, 6)
        .background(bannerBackground)
    }

    private var bannerColor: Color {
        switch viewModel.syncStatus {
        case .synced: return .green
        case .syncing: return .blue
        case .offline: return .orange
        case .error: return .red
        case .idle: return .secondary
        }
    }

    private var bannerBackground: some ShapeStyle {
        bannerColor.opacity(0.1)
    }
}
```

3. Integra el banner en `TaskListView` añadiendo justo encima de la `List`:

```swift
// Dentro del NavigationStack de TaskListView, antes de List:
@StateObject private var syncVM = SyncStatusViewModel()

// En el body, justo antes de List {...}:
VStack(spacing: 0) {
    SyncBannerView(viewModel: syncVM)

    List {
        // ... contenido existente de la lista ...
    }
}
```

4. Compila y ejecuta. Verifica que el banner aparece con el estado "Listo" o "Sincronizado".

5. Haz commit:

```bash
git add -A
git commit -m "feat: add SyncStatusViewModel with combineLatest pipeline and SyncBannerView"
```

### Resultado Esperado

Un banner compacto aparece en la parte superior de la lista de tareas mostrando el estado de sincronización con icono, texto descriptivo y botón de recarga.

### Verificación

- El banner muestra "Listo" al iniciar.
- Si llamas `syncVM.updateNetworkStatus(isOnline: false)` desde un botón de prueba, el banner cambia a "Sin conexión" con icono de wifi tachado.
- El pipeline `combineLatest` actualiza el estado automáticamente cuando cambia cualquiera de los dos subjects.

---

## Paso 5 — Auditoría Completa de Manejo de Errores

**Objetivo:** Recorrer todas las capas de la app (UI → ViewModel → Service → Repository) reemplazando manejo genérico de errores con `AppError` tipado y presentación de alertas centralizadas.

### Instrucciones

1. **Actualizar `TaskNetworkService`** (del Lab 04) para lanzar `AppError`:

```swift
// TaskNetworkService.swift (actualización parcial)
// Reemplazar los errores genéricos con AppError

func fetchTasks() async throws -> [TaskDTO] {
    guard let url = URL(string: "\(baseURL)/todos?_limit=20") else {
        throw AppError.invalidURL("\(baseURL)/todos?_limit=20")
    }

    let (data, response): (Data, URLResponse)
    do {
        (data, response) = try await URLSession.shared.data(from: url)
    } catch let urlError as URLError {
        throw ErrorHandler.shared.mapToAppError(urlError)
    } catch {
        throw AppError.unknown(error.localizedDescription)
    }

    guard let httpResponse = response as? HTTPURLResponse else {
        throw AppError.unknown("Respuesta no HTTP")
    }

    guard (200...299).contains(httpResponse.statusCode) else {
        throw AppError.serverError(statusCode: httpResponse.statusCode)
    }

    do {
        return try JSONDecoder().decode([TaskDTO].self, from: data)
    } catch {
        throw AppError.decodingFailed(error.localizedDescription)
    }
}
```

> **Nota:** Adapta este patrón a todos los métodos de red que tengas (`postTask`, etc.). La clave es que cada `catch` genérico se convierta en un `AppError` específico.

2. **Actualizar el repositorio SwiftData** para propagar `AppError`:

```swift
// En tu repositorio de SwiftData (ej. SwiftDataTaskRepository)
func save(_ task: TaskItem) throws {
    do {
        modelContext.insert(task)
        try modelContext.save()
    } catch {
        throw AppError.saveFailed(error.localizedDescription)
    }
}

func fetchAll() throws -> [TaskItem] {
    do {
        let descriptor = FetchDescriptor<TaskItem>(
            sortBy: [SortDescriptor(\.createdAt, order: .reverse)]
        )
        return try modelContext.fetch(descriptor)
    } catch {
        throw AppError.fetchFailed(error.localizedDescription)
    }
}
```

3. **Integrar alertas centralizadas** en la vista raíz de la app. Modifica tu `ContentView` o `TaskManagerAppApp.swift`:

```swift
// En la vista raíz principal
@ObservedObject private var errorHandler = ErrorHandler.shared

// Añadir este modificador al contenedor principal:
.alert(
    "Error",
    isPresented: $errorHandler.showErrorAlert,
    presenting: errorHandler.currentError
) { _ in
    Button("Aceptar") {
        errorHandler.clearError()
    }
} message: { error in
    Text(error.localizedDescription)
}
```

4. **Verificación exhaustiva:** Ejecuta una búsqueda global en el proyecto:

```bash
cd ~/Developer/TaskManagerApp
# Buscar force-unwraps restantes
grep -rn 'try!' --include="*.swift" TaskManagerApp/
grep -rn '\.value!' --include="*.swift" TaskManagerApp/

# Buscar catch vacíos
grep -rn 'catch {' --include="*.swift" TaskManagerApp/ -A1 | grep -B1 "}"
```

Corrige cualquier hallazgo: reemplaza `try!` con `do/catch`, force-unwraps con `guard let`/`if let`, y `catch {}` vacíos con logging a `ErrorHandler`.

5. Compila, ejecuta y prueba provocando errores (ej. desactiva la conexión de red en el simulador: **Device → Network → Disconnect**). Verifica que aparezcan alertas contextuales.

6. Haz commit:

```bash
git add -A
git commit -m "refactor: replace all force-unwraps and generic catches with typed AppError propagation"
```

### Resultado Esperado

Toda la cadena de errores fluye de forma tipada: `Repository throws AppError` → `Service throws AppError` → `ViewModel catches AppError` → `ErrorHandler presenta alerta`. No quedan `try!` ni force-unwraps en el código.

### Verificación

- Desconecta la red → la app muestra alerta con "No hay conexión a Internet disponible."
- Intenta guardar una tarea con título vacío → la validación previene el guardado y muestra mensaje inline.
- Revisa la consola de Xcode → los errores se loguean con `os.log` indicando archivo y línea.

---

## Paso 6 — Perfilar con Instruments: Time Profiler y Leaks

**Objetivo:** Usar Instruments 15.4 para identificar problemas de rendimiento, introducir intencionalmente un retain cycle, detectarlo con el instrumento Leaks y corregirlo.

### Instrucciones

1. **Introducir intencionalmente un retain cycle** para practicar su detección. Crea `LeakDemoViewModel.swift` (temporal, para Instruments):

```swift
// LeakDemoViewModel.swift
// TaskManagerApp
// ⚠️ ESTE ARCHIVO CONTIENE UN RETAIN CYCLE INTENCIONAL PARA PRÁCTICA

import Foundation
import Combine

final class LeakDemoViewModel: ObservableObject {

    @Published var counter: Int = 0
    private var cancellables = Set<AnyCancellable>()
    private var timer: Timer?

    init() {
        // ⚠️ BUG INTENCIONAL: closure captura self fuertemente
        timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { _ in
            self.counter += 1  // Retain cycle: Timer → closure → self → timer
            print("LeakDemo counter: \(self.counter)")
        }
    }

    deinit {
        print("LeakDemoViewModel deinit llamado")
        timer?.invalidate()
    }
}
```

2. Añade temporalmente una referencia en alguna vista para que se instancie:

```swift
// En TaskListView, añadir temporalmente:
@StateObject private var leakDemo = LeakDemoViewModel()

// Y en el body, añadir en algún lugar visible:
Text("Leak counter: \(leakDemo.counter)")
    .font(.caption2)
    .foregroundStyle(.red)
```

3. **Ejecutar Time Profiler:**

   a. En Xcode, selecciona **Product → Profile** (⌘I). Esto compila en modo Release y abre Instruments.

   b. En la ventana de selección de plantillas de Instruments, elige **Time Profiler**.

   c. Presiona el botón rojo **Record** (●).

   d. Interactúa con la app durante 30 segundos: navega entre vistas, crea tareas, usa la búsqueda.

   e. Presiona **Stop** (■).

   f. Analiza el Call Tree:
      - Marca las casillas **Hide System Libraries** y **Invert Call Tree** en la barra inferior.
      - Busca funciones de tu código que consuman más del 5% del tiempo total.
      - Toma una captura de pantalla del resultado (⌘⇧4 para captura de región).

4. **Ejecutar Leaks:**

   a. En Instruments, presiona **+** en la barra superior para añadir un instrumento.

   b. Busca y añade **Leaks**.

   c. Presiona **Record** nuevamente.

   d. Navega a la vista donde está `LeakDemoViewModel`. Luego navega fuera y regresa varias veces.

   e. Después de 20–30 segundos, Instruments debería marcar leaks con triángulos rojos en la línea de tiempo.

   f. Haz clic en un leak detectado → Instruments mostrará el retain cycle con el backtrace.

   g. Toma una captura de pantalla del leak detectado.

5. **Corregir el retain cycle:**

```swift
// LeakDemoViewModel.swift — VERSIÓN CORREGIDA

init() {
    // ✅ CORRECCIÓN: usar [weak self] en el closure del Timer
    timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] _ in
        guard let self = self else { return }
        self.counter += 1
        print("LeakDemo counter: \(self.counter)")
    }
}
```

6. Ejecuta Instruments → Leaks nuevamente y verifica que el leak desapareció. Toma captura.

7. **Ejecutar SwiftUI Instrument** (si disponible en tu versión de Instruments):

   a. Añade el instrumento **SwiftUI** en Instruments.

   b. Graba una sesión de 20 segundos interactuando con la búsqueda.

   c. Identifica si hay re-renders excesivos de `body` en alguna vista.

   d. Toma captura de los hallazgos.

8. **Eliminar el código de demostración** del leak (quita `@StateObject private var leakDemo` y el `Text` temporal de `TaskListView`). Sin embargo, **conserva** `LeakDemoViewModel.swift` con la versión corregida como referencia educativa.

9. Guarda las capturas de pantalla en una carpeta `docs/instruments/` en el proyecto:

```bash
mkdir -p ~/Developer/TaskManagerApp/docs/instruments
# Mover las capturas aquí (ajustar nombres según tus archivos)
# cp ~/Desktop/time-profiler.png docs/instruments/
# cp ~/Desktop/leaks-detected.png docs/instruments/
# cp ~/Desktop/leaks-fixed.png docs/instruments/
```

10. Haz commit:

```bash
git add -A
git commit -m "perf: profile with Instruments, fix retain cycle in Timer closure with [weak self]"
```

### Resultado Esperado

- Time Profiler muestra el call tree de tu app con funciones propias identificables.
- Leaks detecta el retain cycle del Timer antes de la corrección.
- Tras la corrección, Leaks no reporta nuevos leaks.
- Tres capturas de pantalla documentan los hallazgos.

### Verificación

- En la consola, tras navegar fuera de la vista con leak corregido, aparece `"LeakDemoViewModel deinit llamado"`.
- Instruments → Leaks no muestra triángulos rojos en la sesión post-corrección.
- Las capturas existen en `docs/instruments/`.

---

## Paso 7 — Documentar Reporte de Instruments en README.md

**Objetivo:** Crear documentación profesional del reporte de perfilado con 3 hallazgos.

### Instrucciones

1. Abre (o crea) `README.md` en la raíz del proyecto y añade la sección de Instruments:

```markdown
# TaskManagerApp

## Arquitectura

La aplicación sigue una arquitectura MVVM con las siguientes capas:

```
┌─────────────────────────────────────────┐
│              Vistas (SwiftUI)           │
│  TaskListView · AddTaskView · Search    │
├─────────────────────────────────────────┤
│            ViewModels (Combine)         │
│  SearchVM · AddTaskVM · SyncStatusVM    │
├─────────────────────────────────────────┤
│         Servicios / Repositorios        │
│  TaskNetworkService · SwiftDataRepo     │
├─────────────────────────────────────────┤
│           Modelos (SwiftData)           │
│  TaskItem · TaskDTO · AppError          │
└─────────────────────────────────────────┘
```

## Tecnologías

- Swift 5.10, SwiftUI, UIKit
- SwiftData (persistencia local)
- Combine (reactividad)
- URLSession + async/await (red)
- XCTest / XCUITest (pruebas)

## Reporte de Instruments (v1.0.0)

### Hallazgo 1: Retain Cycle en Timer Closure
- **Instrumento:** Leaks
- **Síntoma:** `LeakDemoViewModel` no se liberaba al navegar fuera de la vista.
- **Causa:** Closure del `Timer.scheduledTimer` capturaba `self` fuertemente.
- **Corrección:** Añadir `[weak self]` en la capture list del closure.
- **Evidencia:** ![Leaks detectado](docs/instruments/leaks-detected.png)

### Hallazgo 2: Tiempo en Main Thread durante búsqueda
- **Instrumento:** Time Profiler
- **Síntoma:** Picos de CPU al escribir rápidamente en la barra de búsqueda.
- **Causa:** El filtrado de tareas se ejecutaba sin debounce adecuado.
- **Corrección:** Pipeline Combine con `.debounce(for: .milliseconds(300))`.
- **Evidencia:** ![Time Profiler](docs/instruments/time-profiler.png)

### Hallazgo 3: Re-renders de SwiftUI en TaskListView
- **Instrumento:** SwiftUI Instrument
- **Síntoma:** El body de `TaskListView` se re-evaluaba en cada cambio de `searchText`.
- **Causa:** `@StateObject` del `SearchViewModel` provocaba actualizaciones.
- **Corrección:** Uso de `displayedTasks` computed property para minimizar cambios.
- **Evidencia:** ![SwiftUI Instrument](docs/instruments/swiftui-renders.png)

## Capturas de Pantalla

> Añadir capturas de la app funcionando aquí.

## Cómo Ejecutar

1. Clonar el repositorio
2. Abrir `TaskManagerApp.xcodeproj` en Xcode 15.4
3. Seleccionar simulador iPhone 15 Pro (iOS 17.5)
4. Ejecutar con ⌘R
```

> **Nota:** Si no pudiste obtener alguna captura específica (por ejemplo, el instrumento SwiftUI no está disponible), documenta lo que observaste en texto y ajusta el hallazgo 3 con información real de tu sesión de profiling.

2. Haz commit:

```bash
git add -A
git commit -m "docs: add Instruments profiling report and architecture diagram to README"
```

### Resultado Esperado

`README.md` contiene la arquitectura del proyecto, el reporte de 3 hallazgos de Instruments con referencias a capturas, e instrucciones de ejecución.

### Verificación

- `cat README.md` muestra contenido completo y bien formateado.
- Las referencias a imágenes apuntan a rutas existentes en `docs/instruments/`.

---

## Paso 8 — Merge Final a `main` y Tag `v1.0.0`

**Objetivo:** Completar el ciclo de desarrollo mergeando todas las features a `main` y creando el tag de versión `v1.0.0`.

### Instrucciones

1. Asegúrate de que todo compila y la app funciona correctamente:

```bash
cd ~/Developer/TaskManagerApp

# Compilar desde terminal (opcional, verificación extra)
xcodebuild -project TaskManagerApp.xcodeproj \
  -scheme TaskManagerApp \
  -destination 'platform=iOS Simulator,name=iPhone 15 Pro,OS=17.5' \
  build 2>&1 | tail -5
```

Debes ver `** BUILD SUCCEEDED **`.

2. Merge de la rama feature a `develop`:

```bash
git checkout develop
git merge feature/combine-errors-instruments --no-ff -m "feat: merge Combine, error handling, and Instruments profiling into develop"
git push origin develop
```

3. Merge de `develop` a `main`:

```bash
git checkout main
git merge develop --no-ff -m "release: merge develop into main for v1.0.0 release"
```

4. Crear el tag de versión:

```bash
git tag -a v1.0.0 -m "v1.0.0: Release final del proyecto TaskManagerApp

Incluye:
- Persistencia con SwiftData
- UI con SwiftUI y UIKit
- Capa de red con URLSession y async/await
- Búsqueda reactiva con Combine
- Manejo de errores tipados con AppError
- Perfilado con Instruments
- Pruebas unitarias con XCTest
- Pruebas de UI con XCUITest"
```

5. Push de `main` y el tag a GitHub:

```bash
git push origin main
git push origin v1.0.0
```

6. Verifica en GitHub:

```bash
# Verificar que el tag existe
git tag -l

# Verificar el log final
git log --oneline --graph --all -15
```

### Resultado Esperado

```
* abc1234 (HEAD -> main, tag: v1.0.0) release: merge develop into main for v1.0.0 release
|\
| * def5678 (develop) feat: merge Combine, error handling, and Instruments profiling into develop
| |\
| | * ghi9012 (feature/combine-errors-instruments) docs: add Instruments profiling report...
| | * jkl3456 perf: profile with Instruments, fix retain cycle...
| | * mno7890 refactor: replace all force-unwraps and generic catches...
| | * pqr1234 feat: add SyncStatusViewModel with combineLatest pipeline...
| | * stu5678 feat: implement SearchViewModel with Combine debounce pipeline...
| | * vwx9012 feat: add TaskValidator with Result type and reactive form validation
| | * yza3456 feat: add AppError enum and ErrorHandler service...
```

### Verificación

- `git describe --tags` retorna `v1.0.0`.
- En GitHub, la sección **Releases/Tags** muestra `v1.0.0`.
- La rama `main` contiene todo el código del proyecto completo.
- La app compila y ejecuta correctamente desde la rama `main`.

---

## Validación y Pruebas Finales

Ejecuta esta lista de verificación completa antes de considerar el lab terminado:

| # | Verificación | Comando / Acción | ✓ |
|---|---|---|---|
| 1 | Proyecto compila sin errores | ⌘B en Xcode | ☐ |
| 2 | Proyecto compila sin warnings | Revisar Issue Navigator (⌘5) | ☐ |
| 3 | Búsqueda filtra tareas en tiempo real | Escribir en barra de búsqueda | ☐ |
| 4 | Debounce funciona (no filtra al escribir rápido) | Escribir y borrar rápidamente | ☐ |
| 5 | Banner de sincronización visible | Verificar en TaskListView | ☐ |
| 6 | Validación de formulario funciona | Probar título vacío, largo, válido | ☐ |
| 7 | Alertas de error aparecen | Provocar error de red | ☐ |
| 8 | No hay `try!` en el código | `grep -rn 'try!' --include="*.swift"` | ☐ |
| 9 | No hay force-unwraps injustificados | Búsqueda manual en código | ☐ |
| 10 | Capturas de Instruments guardadas | `ls docs/instruments/` | ☐ |
| 11 | README.md completo | `cat README.md` | ☐ |
| 12 | Tag v1.0.0 creado | `git tag -l` | ☐ |
| 13 | Main actualizado en GitHub | `git log origin/main --oneline -3` | ☐ |
| 14 | `deinit` se llama tras corregir leak | Verificar en consola de Xcode | ☐ |

---

## Solución de Problemas

### Problema 1: El pipeline Combine no emite valores tras `debounce`

**Síntomas:** Al escribir en la barra de búsqueda, `filteredTasks` nunca se actualiza. La lista permanece vacía o sin cambios. No hay errores de compilación.

**Causa:** El `AnyCancellable` devuelto por `.sink` no se almacena en `Set<AnyCancellable>`, por lo que la suscripción se cancela inmediatamente al salir del scope del método `init()`. Alternativamente, si se usa `.assign(to: &$filteredTasks)` con un `@Published` que no existe o tiene un tipo incompatible, el pipeline se descarta silenciosamente.

**Solución:**

1. Verifica que cada `.sink` termina con `.store(in: &cancellables)`:

```swift
// ❌ Incorrecto: la suscripción se pierde
$searchText
    .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
    .sink { print($0) }
    // Falta .store(in:)

// ✅ Correcto
$searchText
    .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
    .sink { print($0) }
    .store(in: &cancellables)  // Retiene la suscripción
```

2. Verifica que `cancellables` es una propiedad de instancia (`private var cancellables = Set<AnyCancellable>()`), no una variable local.

3. Añade un `.handleEvents(receiveOutput: { print("DEBUG: \($0)") })` antes del `.sink` para confirmar que los valores fluyen por el pipeline.

---

### Problema 2: Instruments no detecta el retain cycle (Leaks no muestra triángulos rojos)

**Síntomas:** Ejecutas Instruments con el instrumento Leaks, interactúas con la app, pero no aparecen indicadores de leak a pesar de que el `deinit` de `LeakDemoViewModel` nunca se imprime en la consola.

**Causa:** Instruments realiza snapshots de memoria a intervalos. Si el objeto con leak se crea y la vista permanece activa durante toda la sesión de grabación, Instruments no lo detecta como leak porque el objeto aún es "alcanzable" desde la jerarquía de vistas activa. El leak solo se manifiesta cuando la vista debería haber liberado el ViewModel pero no lo hace.

**Solución:**

1. Asegúrate de navegar **fuera** de la vista que contiene `LeakDemoViewModel` durante la sesión de Instruments. El leak se detecta cuando el objeto debería haberse liberado pero persiste.

2. Usa una navegación push/pop: navega a la vista con el leak, luego regresa. Repite 3–4 veces para que Instruments acumule suficientes instancias huérfanas.

3. Si usas `@StateObject`, SwiftUI controla el ciclo de vida. Prueba presentar la vista como `.sheet` y cerrarla, lo que debería liberar el ViewModel:

```swift
.sheet(isPresented: $showLeakDemo) {
    LeakDemoView() // Contiene @StateObject var vm = LeakDemoViewModel()
}
```

4. Espera al menos 10 segundos después de cerrar la vista antes de detener la grabación. Instruments necesita tiempo para ejecutar su análisis de ciclos de retención.

5. En Instruments, haz clic en **Mark Generation** (botón con bandera) después de cada navegación fuera de la vista para crear checkpoints de memoria que faciliten la comparación.

---

## Limpieza

```bash
# Eliminar la rama feature local (ya mergeada)
git checkout main
git branch -d feature/combine-errors-instruments

# Verificar ramas restantes
git branch -a

# Opcional: limpiar datos del simulador si ocupa mucho espacio
# xcrun simctl erase all  # ⚠️ Borra TODOS los datos de TODOS los simuladores
```

> **Importante:** No elimines `LeakDemoViewModel.swift` (versión corregida) — sirve como referencia educativa del patrón `[weak self]` en closures.

---

## Resumen

En este laboratorio de cierre completaste el ciclo de desarrollo profesional del proyecto **TaskManagerApp**:

| Parte | Duración | Logro |
|---|---|---|
| **Combine** | ~100 min | `SearchViewModel` con `debounce`/`removeDuplicates`/`map`; `SyncStatusViewModel` con `combineLatest`; barra de búsqueda `.searchable`; banner de sincronización |
| **Errores** | ~80 min | `AppError` enum tipado; `ErrorHandler` centralizado con `os.log`; `TaskValidator` con `Result<T, E>`; validación reactiva de formularios; eliminación de force-unwraps |
| **Instruments** | ~42 min | Time Profiler, Leaks (retain cycle detectado y corregido), SwiftUI instrument; 3 hallazgos documentados |
| **Cierre** | ~30 min | Merge `develop` → `main`, tag `v1.0.0`, README.md con arquitectura y reporte |

### Conceptos Clave Aplicados

- **Publisher/Subscriber**: `@Published`, `CurrentValueSubject`, `PassthroughSubject` como puentes entre código imperativo y reactivo.
- **Operadores Combine**: `debounce`, `removeDuplicates`, `map`, `combineLatest`, `filter`, `handleEvents`, `receive(on:)`.
- **Gestión de memoria**: `AnyCancellable` + `store(in:)` para retener suscripciones; `[weak self]` en closures para evitar retain cycles.
- **Errores tipados**: `Result<T, AppError>` para validación; `throws` con `AppError` en capas de servicio; `do/catch` exhaustivo en ViewModels.
- **Profiling**: Instruments como herramienta esencial del ciclo de desarrollo, no solo de debugging.

### Recursos Adicionales

- [Documentación oficial de Combine (Apple)](https://developer.apple.com/documentation/combine)
- [Using Combine (guía comunitaria)](https://heckj.github.io/swiftui-notes/)
- [Instruments Help (Apple)](https://help.apple.com/instruments/mac/current/)
- [Swift Error Handling (Swift.org)](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/errorhandling/)
- [WWDC23: Analyze hangs with Instruments](https://developer.apple.com/videos/play/wwdc2023/10248/)
