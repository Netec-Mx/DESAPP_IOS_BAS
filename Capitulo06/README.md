# Práctica Incremental — Testing, UI Testing y Refactorización SOLID

| Campo | Detalle |
|---|---|
| **Duración** | 180 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |

## Descripción General

Esta práctica de cierre integra los tres pilares del Módulo 6: pruebas unitarias con XCTest, pruebas de interfaz con XCUITest y refactorización guiada por principios SOLID. Trabajarás sobre el proyecto `TaskManagerApp` construido en módulos anteriores, añadiendo una suite completa de pruebas que servirá como red de seguridad para una refactorización profunda. Al finalizar, tu proyecto contará con al menos 8 pruebas unitarias, 5 pruebas de UI y un código reestructurado con cobertura mínima del 70 % en modelos y servicios.

## Objetivos de Aprendizaje

- [ ] Configurar targets de Unit Testing y UI Testing en un proyecto Xcode 15.4 existente y escribir al menos 8 pruebas unitarias con `XCTestCase` usando el patrón AAA.
- [ ] Crear al menos 5 pruebas de interfaz con XCUITest que simulen flujos reales del usuario (creación, listado, eliminación, búsqueda y validación de errores).
- [ ] Identificar al menos 3 violaciones a principios SOLID en el código existente, aplicar refactorizaciones concretas y verificar con la suite de pruebas que no se introducen regresiones.
- [ ] Habilitar y analizar el reporte de Code Coverage en Xcode, alcanzando ≥ 70 % en archivos de modelo y servicio.
- [ ] Documentar la evolución del proyecto mediante commits de Git al final de cada fase.

## Prerrequisitos

### Conocimientos Previos

| Requisito | Detalle |
|---|---|
| Swift 5.10 | Structs, clases, protocolos, closures, `throws`/`catch` |
| Xcode 15.4 | Navegación por Project Navigator, Test Navigator, Debug Navigator |
| UIKit básico | Storyboards, Auto Layout, navegación entre ViewControllers |
| Git | Configurado con nombre de usuario y correo; familiaridad con commits y ramas |
| Módulos 1–5 | Proyecto `TaskManagerApp` funcional o starter project proporcionado |

### Acceso Requerido

- Mac con macOS Ventura 13.5+ o Sonoma 14.5
- Xcode 15.4 instalado con simulador iPhone 15 Pro (iOS 17.5)
- Repositorio `task-manager-ios` en GitHub (rama `develop`)

## Entorno del Laboratorio

### Hardware Mínimo

| Componente | Especificación |
|---|---|
| Procesador | Apple Silicon (M1/M2/M3) o Intel i5 8.ª gen+ |
| RAM | 16 GB mínimo |
| Disco libre | 50 GB |
| Pantalla | 1440 × 900 mínimo |

### Software

| Herramienta | Versión |
|---|---|
| macOS | Ventura 13.6.7 / Sonoma 14.5 |
| Xcode | 15.4 |
| Swift | 5.10 |
| Simulador | iPhone 15 Pro — iOS 17.5 |
| Git | 2.45.2 |
| XCTest Framework | 15.4 (incluido con Xcode) |

### Preparación Inicial del Entorno

Antes de comenzar, asegúrate de que el proyecto compila y el repositorio está limpio:

```bash
cd ~/Developer/TaskManagerApp
git checkout develop
git pull origin develop
open TaskManagerApp.xcodeproj
```

Verifica la compilación con **⌘ + B** en Xcode. Si no tienes el proyecto de módulos anteriores, utiliza el starter project proporcionado por el instructor (con los Labs 01–05 resueltos) y clónalo en `~/Developer/TaskManagerApp`.

---

## Código Base Requerido

Este laboratorio trabaja sobre el proyecto de los Labs 01–05. **No vuelvas a crear** `TaskItem`, `TaskRepository` ni `InMemoryTaskRepository`: ya existen y otros archivos dependen de ellos (si los declararas de nuevo, Xcode marcaría *"Invalid redeclaration"*). Lo único nuevo es `TaskManager`, la clase con las reglas de negocio que vas a probar y, en la Fase 3, a refactorizar.

### Paso 0 — Preparar el modelo y crear `TaskManager`

**Objetivo:** Confirmar que el modelo del Lab 01 está listo para las pruebas y crear `TaskManager.swift` con las reglas de negocio (validación, filtros y estadísticas).

**Instrucciones:**

1. Abre `Models/TaskItem.swift` (Lab 01) y verifica que contiene `enum Priority` (`low`, `medium`, `high`, `critical`) y `struct TaskItem` con `id`, `title`, `taskDescription` (opcional), `isCompleted`, `priority`, `createdAt` y `dueDate` (opcional). No lo modifiques.

> **Recuerda:** El inicializador del Lab 01 recibe los argumentos en este orden: `TaskItem(title:taskDescription:isCompleted:priority:dueDate:)`. Todos menos `title` tienen valor por defecto. `task.description` **no** es la descripción de la tarea: es el texto con formato de `CustomStringConvertible` (por ejemplo `⬜️ 🟡 [Media] Comprar leche`); la descripción está en `taskDescription`.

2. En el grupo `Services`, crea `TaskManager.swift` (target `TaskManagerApp`) con el siguiente contenido:

```swift
// TaskManager.swift — Target: TaskManagerApp

import Foundation

/// Errores de las reglas de negocio de TaskManager.
enum TaskError: Error, Equatable {
    case emptyTitle
    case invalidDate
    case duplicateTask
}

/// Permite comparar prioridades (.high > .low) en filtros y ordenamientos.
extension Priority: Comparable {
    static func < (lhs: Priority, rhs: Priority) -> Bool {
        lhs.rawValue < rhs.rawValue
    }
}

/// Reglas de negocio sobre las tareas.
/// ⚠️ Versión inicial: guarda las tareas en su propio array.
/// En la Fase 3 detectarás que esto viola DIP y lo refactorizarás.
final class TaskManager {
    private(set) var tasks: [TaskItem] = []

    // MARK: - Agregar tarea

    @discardableResult
    func addTask(title: String, description: String = "", dueDate: Date? = nil, priority: Priority = .medium) throws -> TaskItem {
        let trimmedTitle = title.trimmingCharacters(in: .whitespacesAndNewlines)
        guard !trimmedTitle.isEmpty else {
            throw TaskError.emptyTitle
        }

        if let date = dueDate, date < Date() {
            throw TaskError.invalidDate
        }

        if tasks.contains(where: { $0.title.lowercased() == trimmedTitle.lowercased() && !$0.isCompleted }) {
            throw TaskError.duplicateTask
        }

        // Inicializador del Lab 01 (orden: title, taskDescription, isCompleted, priority, dueDate)
        let task = TaskItem(
            title: trimmedTitle,
            taskDescription: description.isEmpty ? nil : description,
            priority: priority,
            dueDate: dueDate
        )
        tasks.append(task)
        return task
    }

    // MARK: - Eliminar tarea

    @discardableResult
    func removeTask(by id: UUID) -> Bool {
        guard let index = tasks.firstIndex(where: { $0.id == id }) else {
            return false
        }
        tasks.remove(at: index)
        return true
    }

    // MARK: - Completar tarea

    @discardableResult
    func toggleComplete(for id: UUID) -> Bool {
        guard let index = tasks.firstIndex(where: { $0.id == id }) else {
            return false
        }
        tasks[index].isCompleted.toggle()
        return true
    }

    // MARK: - Filtrar tareas

    func pendingTasks() -> [TaskItem] {
        tasks.filter { !$0.isCompleted }
    }

    func completedTasks() -> [TaskItem] {
        tasks.filter { $0.isCompleted }
    }

    func searchTasks(query: String) -> [TaskItem] {
        let lowered = query.lowercased()
        return tasks.filter {
            $0.title.lowercased().contains(lowered) ||
            ($0.taskDescription ?? "").lowercased().contains(lowered)
        }
    }

    func tasksSorted(by priority: Bool = true) -> [TaskItem] {
        if priority {
            return tasks.sorted { $0.priority > $1.priority }
        }
        return tasks.sorted { ($0.dueDate ?? .distantFuture) < ($1.dueDate ?? .distantFuture) }
    }

    // MARK: - Estadísticas

    func completionPercentage() -> Double {
        guard !tasks.isEmpty else { return 0.0 }
        let completed = Double(completedTasks().count)
        return (completed / Double(tasks.count)) * 100.0
    }
}
```

> **Nota:** `TaskError` es independiente de `AppError` (Lab 05): `TaskError` describe las reglas de `TaskManager` y es lo que verifican las pruebas con `XCTAssertThrowsError`.

3. Compila con **⌘ + B** para confirmar que no hay errores.

4. Haz un commit:

```bash
cd ~/Developer/TaskManagerApp
git add -A
git commit -m "feat: add TaskManager business logic"
```

---

## FASE 1 — Testing Unitario con XCTest (60 minutos)

### Paso 1 — Crear el Unit Test Target

**Objetivo:** Agregar un target de pruebas unitarias al proyecto existente.

**Instrucciones:**

1. En Xcode, ve a **File → New → Target…**
2. En la sección *Test*, selecciona **Unit Testing Bundle**.
3. Configura los campos:
   - **Product Name:** `TaskManagerAppTests`
   - **Team:** Tu equipo de desarrollo (o *None*)
   - **Organization Identifier:** `com.cursounam`
   - **Bundle Identifier:** se generará como `com.cursounam.TaskManagerAppTests`
   - **Language:** Swift
   - **Target to be Tested:** `TaskManagerApp`
4. Haz clic en **Finish**.
5. Xcode creará la carpeta `TaskManagerAppTests/` con un archivo `TaskManagerAppTests.swift` por defecto. **Elimina** ese archivo: clic derecho → *Delete* → *Move to Trash*.

**Resultado esperado:** En el Project Navigator aparece el grupo `TaskManagerAppTests` vacío y en la pestaña de targets del proyecto se lista `TaskManagerAppTests`.

**Verificación:** Ejecuta **⌘ + U**. Xcode debería compilar ambos targets sin errores (0 tests ejecutados, 0 fallos).

### Paso 2 — Crear el archivo TaskModelTests.swift

**Objetivo:** Implementar pruebas unitarias para validar la creación y propiedades del modelo `TaskItem`.

**Instrucciones:**

1. Clic derecho sobre la carpeta `TaskManagerAppTests` → **New File… → Swift File** → nombrar `TaskModelTests.swift`. Asegúrate de que **solo** el target `TaskManagerAppTests` esté marcado.

2. Escribe el siguiente código:

```swift
// TaskModelTests.swift — Target: TaskManagerAppTests

import XCTest
@testable import TaskManagerApp

final class TaskModelTests: XCTestCase {

    // MARK: - Pruebas de creación

    func testTaskCreation_WithRequiredFields_SetsPropertiesCorrectly() {
        // Arrange & Act
        let task = TaskItem(title: "Comprar leche")

        // Assert
        XCTAssertEqual(task.title, "Comprar leche")
        XCTAssertNil(task.taskDescription)
        XCTAssertFalse(task.isCompleted)
        XCTAssertNil(task.dueDate)
        XCTAssertEqual(task.priority, .medium)
    }

    func testTaskCreation_WithAllFields_SetsPropertiesCorrectly() {
        // Arrange
        let date = Date().addingTimeInterval(86400) // mañana

        // Act
        // (orden del inicializador del Lab 01)
        let task = TaskItem(
            title: "Estudiar Swift",
            taskDescription: "Leer capítulo 6",
            isCompleted: true,
            priority: .high,
            dueDate: date
        )

        // Assert
        XCTAssertEqual(task.title, "Estudiar Swift")
        XCTAssertEqual(task.taskDescription, "Leer capítulo 6")
        XCTAssertTrue(task.isCompleted)
        XCTAssertEqual(task.dueDate, date)
        XCTAssertEqual(task.priority, .high)
    }

    func testTaskCreation_GeneratesUniqueIDs() {
        let task1 = TaskItem(title: "Tarea 1")
        let task2 = TaskItem(title: "Tarea 2")

        XCTAssertNotEqual(task1.id, task2.id, "Cada tarea debe tener un UUID único")
    }

    // MARK: - Pruebas de prioridad

    func testTaskPriority_HighIsGreaterThanLow() {
        // Usa la conformancia Comparable agregada en TaskManager.swift (Paso 0)
        XCTAssertTrue(Priority.high > Priority.low)
        XCTAssertTrue(Priority.medium > Priority.low)
        XCTAssertTrue(Priority.critical > Priority.high)
    }
}
```

3. Ejecuta las pruebas con **⌘ + U**.

**Resultado esperado:** 4 pruebas ejecutadas, 4 aprobadas (rombos verdes en Test Navigator).

**Verificación:** Abre **Test Navigator** (⌘ + 6) y confirma que `TaskModelTests` muestra 4 ✅.

### Paso 3 — Crear el archivo TaskManagerTests.swift

**Objetivo:** Implementar pruebas unitarias para la lógica de negocio de `TaskManager` usando `setUp()` y `tearDown()`.

**Instrucciones:**

1. Crea un nuevo archivo `TaskManagerTests.swift` dentro de `TaskManagerAppTests` (mismo procedimiento que el paso anterior).

2. Escribe el siguiente código:

```swift
// TaskManagerTests.swift — Target: TaskManagerAppTests

import XCTest
@testable import TaskManagerApp

final class TaskManagerTests: XCTestCase {

    var sut: TaskManager!  // System Under Test

    // MARK: - Ciclo de vida

    override func setUp() {
        super.setUp()
        sut = TaskManager()
    }

    override func tearDown() {
        sut = nil
        super.tearDown()
    }

    // MARK: - Pruebas: Agregar tarea

    func testAddTask_WithValidTitle_IncreasesTaskCount() throws {
        // Act
        let _ = try sut.addTask(title: "Nueva tarea")

        // Assert
        XCTAssertEqual(sut.tasks.count, 1)
    }

    func testAddTask_WithEmptyTitle_ThrowsEmptyTitleError() {
        // Assert
        XCTAssertThrowsError(try sut.addTask(title: "   ")) { error in
            XCTAssertEqual(error as? TaskError, TaskError.emptyTitle)
        }
    }

    func testAddTask_WithPastDate_ThrowsInvalidDateError() {
        // Arrange
        let pastDate = Date().addingTimeInterval(-86400) // ayer

        // Assert
        XCTAssertThrowsError(try sut.addTask(title: "Tarea", dueDate: pastDate)) { error in
            XCTAssertEqual(error as? TaskError, TaskError.invalidDate)
        }
    }

    func testAddTask_DuplicateActiveTitle_ThrowsDuplicateError() throws {
        // Arrange
        let _ = try sut.addTask(title: "Duplicada")

        // Assert
        XCTAssertThrowsError(try sut.addTask(title: "duplicada")) { error in
            XCTAssertEqual(error as? TaskError, TaskError.duplicateTask)
        }
    }

    // MARK: - Pruebas: Eliminar tarea

    func testRemoveTask_WithValidID_RemovesTaskAndReturnsTrue() throws {
        // Arrange
        let task = try sut.addTask(title: "Para eliminar")

        // Act
        let result = sut.removeTask(by: task.id)

        // Assert
        XCTAssertTrue(result)
        XCTAssertEqual(sut.tasks.count, 0)
    }

    func testRemoveTask_WithInvalidID_ReturnsFalse() {
        let result = sut.removeTask(by: UUID())
        XCTAssertFalse(result)
    }

    // MARK: - Pruebas: Completar tarea

    func testToggleComplete_WithValidID_MarksTaskAsCompleted() throws {
        // Arrange
        let task = try sut.addTask(title: "Por completar")

        // Act
        let result = sut.toggleComplete(for: task.id)

        // Assert
        XCTAssertTrue(result)
        XCTAssertTrue(sut.tasks[0].isCompleted)
    }

    // MARK: - Pruebas: Filtrado

    func testPendingTasks_ReturnsOnlyIncompleteTasks() throws {
        // Arrange
        let _ = try sut.addTask(title: "Pendiente")
        let _ = try sut.addTask(title: "Completada")
        sut.toggleComplete(for: sut.tasks[1].id)

        // Act
        let pending = sut.pendingTasks()

        // Assert
        XCTAssertEqual(pending.count, 1)
        XCTAssertEqual(pending.first?.title, "Pendiente")
    }

    func testSearchTasks_WithMatchingQuery_ReturnsFilteredResults() throws {
        // Arrange
        let _ = try sut.addTask(title: "Comprar leche")
        let _ = try sut.addTask(title: "Estudiar Swift")
        let _ = try sut.addTask(title: "Comprar pan")

        // Act
        let results = sut.searchTasks(query: "comprar")

        // Assert
        XCTAssertEqual(results.count, 2)
    }

    // MARK: - Pruebas: Estadísticas

    func testCompletionPercentage_WithMixedTasks_ReturnsCorrectValue() throws {
        // Arrange
        let _ = try sut.addTask(title: "T1")
        let _ = try sut.addTask(title: "T2")
        let _ = try sut.addTask(title: "T3")
        let _ = try sut.addTask(title: "T4")
        sut.toggleComplete(for: sut.tasks[0].id)
        sut.toggleComplete(for: sut.tasks[1].id)

        // Act
        let percentage = sut.completionPercentage()

        // Assert
        XCTAssertEqual(percentage, 50.0, accuracy: 0.01)
    }

    func testCompletionPercentage_WithNoTasks_ReturnsZero() {
        XCTAssertEqual(sut.completionPercentage(), 0.0)
    }
}
```

3. Ejecuta **⌘ + U**.

**Resultado esperado:** Se ejecutan las pruebas de ambos archivos. Total: 15 pruebas, 15 aprobadas.

**Verificación:**
- Test Navigator muestra `TaskModelTests` (4 ✅) y `TaskManagerTests` (11 ✅).
- No hay rombos rojos.

### Paso 4 — Commit de la Fase 1

**Objetivo:** Documentar el progreso con un commit convencional.

```bash
cd ~/Developer/TaskManagerApp
git add -A
git commit -m "feat(tests): add unit tests for TaskItem model and TaskManager (15 tests)"
```

---

## FASE 2 — UI Testing con XCUITest (60 minutos)

### Paso 5 — Crear el UI Test Target

**Objetivo:** Agregar un target de pruebas de interfaz de usuario.

**Instrucciones:**

1. Ve a **File → New → Target…**
2. Selecciona **UI Testing Bundle**.
3. Configura:
   - **Product Name:** `TaskManagerAppUITests`
   - **Organization Identifier:** `com.cursounam`
   - **Target to be Tested:** `TaskManagerApp`
4. Haz clic en **Finish**.
5. Elimina el archivo generado `TaskManagerAppUITests.swift` y `TaskManagerAppUITestsLaunchTests.swift` (Move to Trash).

**Resultado esperado:** El grupo `TaskManagerAppUITests` aparece vacío en el Project Navigator.

### Paso 6 — Agregar Accessibility Identifiers a la interfaz

**Objetivo:** Preparar la interfaz UIKit existente (Labs 02–05) para que XCUITest pueda localizar sus elementos, y agregar la búsqueda que probarán las pruebas de UI.

**Instrucciones:**

Las pruebas de UI localizan elementos por `accessibilityIdentifier`. **No crees un `TaskListViewController` nuevo ni cambies el `SceneDelegate`**: la pantalla principal ya es la del Lab 02 (Storyboard con `UINavigationController` → `TaskListViewController`, formulario modal `AddTaskViewController`, borrado con swipe "Eliminar"). Solo vas a agregarle identificadores, una barra de búsqueda y datos de prueba.

1. **Datos fijos para las pruebas.** Abre `AppConfiguration.swift` (Lab 03) y agrega este bloque al inicio de `makeRepository(modelContext:)`. Cuando la app se lanza desde XCUITest, usa un repositorio en memoria con las 3 tareas de ejemplo del Lab 02 y preferencias por defecto, así las pruebas no dependen de lo que haya guardado en SwiftData:

```swift
// Al inicio de makeRepository(modelContext:)
if ProcessInfo.processInfo.arguments.contains("-UITesting") {
    UserPreferencesService.shared.resetToDefaults()
    return InMemoryTaskRepository(seedSampleData: true)
}
```

> **Nota:** Las 3 tareas de ejemplo son "Comprar víveres", "Estudiar Swift" y "Hacer ejercicio". Solo una contiene "Comprar", lo que usa la prueba de búsqueda.

2. **Identificadores en la lista.** Abre `UIKitViews/TaskListViewController.swift` y haz estos cambios:

   a. En `configureNavigationBar()`, identifica el botón **+** (agrega la línea después de crear `addButton`):

```swift
addButton.accessibilityIdentifier = "addTaskButton"
```

   b. En `viewDidLoad()`, después de `setupSyncUI()`, identifica la tabla y agrega la barra de búsqueda:

```swift
tableView.accessibilityIdentifier = "taskTableView"

let searchController = UISearchController(searchResultsController: nil)
searchController.searchBar.placeholder = "Buscar tareas..."
searchController.searchBar.accessibilityIdentifier = "taskSearchBar"
searchController.searchResultsUpdater = self
searchController.obscuresBackgroundDuringPresentation = false
navigationItem.searchController = searchController
```

   c. En `tableView(_:cellForRowAt:)`, antes del `return cell`:

```swift
cell.accessibilityIdentifier = "taskCell_\(indexPath.row)"
```

   d. Agrega la propiedad del texto de búsqueda junto a `tasks`, y al final de `loadTasks()`, **antes** de `tasks = result`, aplica el filtro:

```swift
// Propiedad
var searchQuery = ""

// En loadTasks(), antes de `tasks = result`:
if !searchQuery.isEmpty {
    result = result.filter {
        $0.title.localizedCaseInsensitiveContains(searchQuery) ||
        ($0.taskDescription ?? "").localizedCaseInsensitiveContains(searchQuery)
    }
}
```

   e. Al final del archivo, agrega la conformancia a `UISearchResultsUpdating`:

```swift
// MARK: - UISearchResultsUpdating

extension TaskListViewController: UISearchResultsUpdating {
    func updateSearchResults(for searchController: UISearchController) {
        searchQuery = searchController.searchBar.text ?? ""
        loadTasks()
    }
}
```

   f. En `prepare(for:sender:)`, dentro del bloque de `presentAddTask`, presenta el formulario a pantalla completa. Con la presentación por defecto (hoja), `viewWillAppear` no se vuelve a llamar al cerrar el formulario y la tarea nueva no aparecería hasta recargar:

```swift
} else if segue.identifier == "presentAddTask",
          let navController = segue.destination as? UINavigationController,
          let addVC = navController.topViewController as? AddTaskViewController {
    navController.modalPresentationStyle = .fullScreen   // 🆕
    addVC.repository = repository
}
```

3. **Identificadores en el formulario.** Abre `UIKitViews/AddTaskViewController.swift` (Lab 02) y, al final de `viewDidLoad()`, agrega:

```swift
titleTextField.accessibilityIdentifier = "taskTitleTextField"
navigationItem.rightBarButtonItem?.accessibilityIdentifier = "saveTaskButton"
```

> **Nota:** Se identifica el botón **Save** por `accessibilityIdentifier` porque su texto cambia según el idioma del simulador ("Save" / "Guardar"). La alerta de validación ya existe desde el Lab 02: título "Campo requerido", mensaje "El título de la tarea no puede estar vacío." y botón "OK".

4. Compila con **⌘ + B** y ejecuta en el simulador **iPhone 15 Pro (iOS 17.5)** con **⌘ + R**. Verifica que la lista muestra la barra de búsqueda y que al escribir se filtran las tareas.

5. Haz commit:

```bash
git add -A
git commit -m "feat(ui): add accessibility identifiers and search to TaskListViewController"
```

### Paso 7 — Implementar las pruebas de UI

**Objetivo:** Crear 5 pruebas de interfaz que cubran los flujos principales de la aplicación.

**Instrucciones:**

1. Crea `TaskManagerUITests.swift` dentro del grupo `TaskManagerAppUITests`. Asegúrate de que **solo** el target `TaskManagerAppUITests` esté marcado.

2. Escribe el siguiente código:

```swift
// TaskManagerUITests.swift — Target: TaskManagerAppUITests

import XCTest

final class TaskManagerUITests: XCTestCase {

    var app: XCUIApplication!

    override func setUp() {
        super.setUp()
        continueAfterFailure = false
        app = XCUIApplication()
        app.launchArguments.append("-UITesting")
        app.launch()
    }

    override func tearDown() {
        app = nil
        super.tearDown()
    }

    // MARK: - Test 1: Visualización de la lista de tareas

    func testTaskList_OnLaunch_DisplaysPreloadedTasks() {
        // Assert: La tabla de tareas existe y tiene contenido
        let table = app.tables["taskTableView"]
        XCTAssertTrue(table.waitForExistence(timeout: 5), "La tabla de tareas debe existir")

        let firstCell = table.cells["taskCell_0"]
        XCTAssertTrue(firstCell.exists, "Debe existir al menos una celda de tarea")

        // Verificar que las 3 tareas de ejemplo están visibles
        XCTAssertTrue(table.cells.count >= 3, "Deben mostrarse al menos 3 tareas precargadas")
    }

    // MARK: - Test 2: Creación de una tarea nueva

    func testAddTask_WithValidTitle_AppearsInList() {
        // Arrange
        let table = app.tables["taskTableView"]
        let initialCount = table.cells.count

        // Act: Pulsar botón agregar
        let addButton = app.navigationBars.buttons["addTaskButton"]
        XCTAssertTrue(addButton.waitForExistence(timeout: 3))
        addButton.tap()

        // Escribir título en el formulario
        let titleTextField = app.textFields["taskTitleTextField"]
        XCTAssertTrue(titleTextField.waitForExistence(timeout: 3), "El campo de texto debe aparecer")
        titleTextField.tap()
        titleTextField.typeText("Tarea UI Test")

        // Guardar (botón Save del formulario modal del Lab 02)
        app.navigationBars.buttons["saveTaskButton"].tap()

        // Assert: La nueva tarea aparece en la lista
        XCTAssertTrue(table.staticTexts["Tarea UI Test"].waitForExistence(timeout: 3),
                      "La tarea nueva debe aparecer en la lista")
        XCTAssertEqual(table.cells.count, initialCount + 1, "La lista debe tener una tarea más")
    }

    // MARK: - Test 3: Eliminación de una tarea mediante swipe

    func testDeleteTask_WithSwipeLeft_RemovesFromList() {
        // Arrange
        let table = app.tables["taskTableView"]
        let initialCount = table.cells.count
        XCTAssertTrue(initialCount > 0, "Debe haber tareas para eliminar")

        // Act: Swipe left en la primera celda
        let firstCell = table.cells["taskCell_0"]
        firstCell.swipeLeft()

        // Pulsar el botón "Eliminar" de la acción de swipe (Lab 02)
        let deleteButton = table.buttons["Eliminar"]
        if deleteButton.waitForExistence(timeout: 2) {
            deleteButton.tap()
        }

        // Assert
        // Esperamos brevemente para que la animación termine
        sleep(1)
        let newCount = table.cells.count
        XCTAssertEqual(newCount, initialCount - 1, "La lista debe tener una tarea menos")
    }

    // MARK: - Test 4: Búsqueda de tareas por nombre

    func testSearchTasks_WithQuery_FiltersResults() {
        // Arrange: La tabla tiene 3 tareas precargadas
        let table = app.tables["taskTableView"]
        XCTAssertTrue(table.waitForExistence(timeout: 5))

        // Act: Activar barra de búsqueda deslizando hacia abajo
        table.swipeDown()

        let searchBar = app.searchFields.firstMatch
        XCTAssertTrue(searchBar.waitForExistence(timeout: 3), "La barra de búsqueda debe existir")
        searchBar.tap()
        searchBar.typeText("Comprar")

        // Assert: Solo debe mostrarse la tarea que contiene "Comprar"
        sleep(1) // Esperar actualización del filtro
        let filteredCount = table.cells.count
        XCTAssertEqual(filteredCount, 1, "Solo debe mostrarse 1 tarea con 'Comprar' en el título")
    }

    // MARK: - Test 5: Validación de error al crear tarea con título vacío

    func testAddTask_WithEmptyTitle_ShowsErrorAlert() {
        // Act: Pulsar botón agregar
        let addButton = app.navigationBars.buttons["addTaskButton"]
        addButton.tap()

        // No escribir nada, directamente guardar
        let titleTextField = app.textFields["taskTitleTextField"]
        XCTAssertTrue(titleTextField.waitForExistence(timeout: 3))

        app.navigationBars.buttons["saveTaskButton"].tap()

        // Assert: Debe aparecer la alerta de validación del Lab 02
        let errorAlert = app.alerts["Campo requerido"]
        XCTAssertTrue(errorAlert.waitForExistence(timeout: 3), "Debe mostrarse una alerta de error")

        // Verificar el mensaje
        let errorMessage = errorAlert.staticTexts["El título de la tarea no puede estar vacío."]
        XCTAssertTrue(errorMessage.exists, "El mensaje de error debe indicar título vacío")

        // Cerrar el alert
        errorAlert.buttons["OK"].tap()
    }
}
```

3. Antes de ejecutar, configura el simulador correcto:
   - En la barra de esquemas de Xcode, selecciona el scheme `TaskManagerApp` y el destino **iPhone 15 Pro (iOS 17.5)**.

4. Ejecuta las pruebas de UI con **⌘ + U** (ejecutará tanto unit tests como UI tests). Alternativamente, haz clic derecho sobre `TaskManagerUITests` en el Test Navigator y selecciona **Run "TaskManagerUITests"**.

**Resultado esperado:** Las 5 pruebas de UI se ejecutan en el simulador. Verás la app lanzarse, interacciones automáticas (taps, texto, swipes) y los resultados en el Test Navigator.

**Verificación:**
- Test Navigator muestra `TaskManagerUITests` con 5 pruebas.
- Si alguna prueba falla, revisa los `accessibilityIdentifier` en el código de la app y ajusta los selectores.

> **Nota importante:** Las pruebas de UI dependen del estado visual de la app. Si la estructura de tu ViewController es diferente, adapta los identifiers. Lo esencial es que cada prueba localice elementos por `accessibilityIdentifier` y simule interacciones reales.

### Paso 8 — Commit de la Fase 2

```bash
cd ~/Developer/TaskManagerApp
git add -A
git commit -m "feat(ui-tests): add 5 UI tests for task list, creation, deletion, search, and validation"
```

---

## FASE 3 — Refactorización y Principios SOLID (60 minutos)

### Paso 9 — Identificar violaciones a principios SOLID

**Objetivo:** Analizar el código existente y documentar al menos 3 violaciones a principios SOLID.

**Instrucciones:**

1. Crea un archivo `REFACTORING_LOG.md` en la raíz del proyecto:

```bash
cd ~/Developer/TaskManagerApp
touch REFACTORING_LOG.md
```

2. Abre el archivo y documenta las violaciones identificadas. Analiza el código actual y registra:

```markdown
# Registro de Refactorización — TaskManagerApp

## Violaciones Identificadas

### Violación 1: Principio de Responsabilidad Única (SRP)
- **Archivo:** `TaskListViewController.swift`
- **Problema:** Además de la presentación (tabla, celdas, navegación, alertas),
  `loadTasks()` lee el repositorio, aplica el filtro y el orden de las preferencias
  (Lab 03) y la búsqueda (este lab), y el borrado llama directamente al repositorio.
  El ViewController tiene demasiadas responsabilidades.
- **Solución propuesta:** Extraer esa lógica a una clase `TaskService`; el
  ViewController solo le pide las tareas visibles y le delega las acciones.

### Violación 2: Principio de Inversión de Dependencias (DIP)
- **Archivo:** `TaskManager.swift`
- **Problema:** `TaskManager` guarda las tareas en su propio array. No puede usar
  Core Data ni SwiftData (Lab 03): las reglas de negocio dependen de un
  almacenamiento concreto.
- **Solución propuesta:** Inyectar en `TaskManager` la abstracción que el proyecto
  ya tiene desde el Lab 02: el protocolo `TaskRepository`.

### Violación 3: Principio de Segregación de Interfaces (ISP)
- **Archivo:** `TaskManager.swift`
- **Problema:** `TaskManager` expone operaciones CRUD, filtrado y estadísticas en
  una sola clase. Un consumidor que solo necesita leer tareas se ve obligado a
  depender de métodos de escritura.
- **Solución propuesta:** Separar en protocolos más específicos:
  `TaskReadable` (lectura/filtrado) y `TaskWritable` (CRUD).
```

3. Guarda el archivo.

**Verificación:** Lee las 3 violaciones y confirma que cada una identifica un principio SOLID específico, el archivo afectado, el problema concreto y la solución propuesta.

### Paso 10 — Refactorización 1: Inyectar `TaskRepository` en `TaskManager` (DIP)

**Objetivo:** Invertir la dependencia de almacenamiento reutilizando el protocolo `TaskRepository` del Lab 02, en lugar de crear uno nuevo.

**Instrucciones:**

1. **No crees** un protocolo `TaskRepositoryProtocol` ni otro `InMemoryTaskRepository`: el protocolo `TaskRepository` y sus tres implementaciones (`InMemoryTaskRepository`, `CoreDataTaskRepository`, `SwiftDataTaskRepository`) ya existen desde los Labs 02 y 03. Esa es precisamente la abstracción que pide DIP.

2. Reemplaza el contenido de `TaskManager.swift` por la versión refactorizada. `TaskError` y la extensión `Priority: Comparable` no cambian; solo cambia la clase:

```swift
// TaskManager.swift — Refactorizado con inyección de dependencias (DIP)

import Foundation

enum TaskError: Error, Equatable {
    case emptyTitle
    case invalidDate
    case duplicateTask
}

extension Priority: Comparable {
    static func < (lhs: Priority, rhs: Priority) -> Bool {
        lhs.rawValue < rhs.rawValue
    }
}

final class TaskManager {

    /// Abstracción del Lab 02: puede ser en memoria, Core Data o SwiftData.
    private let repository: TaskRepository

    var tasks: [TaskItem] {
        repository.fetchAll()
    }

    init(repository: TaskRepository = InMemoryTaskRepository()) {
        self.repository = repository
    }

    // MARK: - Agregar tarea

    @discardableResult
    func addTask(title: String, description: String = "", dueDate: Date? = nil, priority: Priority = .medium) throws -> TaskItem {
        let trimmedTitle = title.trimmingCharacters(in: .whitespacesAndNewlines)
        guard !trimmedTitle.isEmpty else {
            throw TaskError.emptyTitle
        }

        if let date = dueDate, date < Date() {
            throw TaskError.invalidDate
        }

        if tasks.contains(where: { $0.title.lowercased() == trimmedTitle.lowercased() && !$0.isCompleted }) {
            throw TaskError.duplicateTask
        }

        let task = TaskItem(
            title: trimmedTitle,
            taskDescription: description.isEmpty ? nil : description,
            priority: priority,
            dueDate: dueDate
        )
        repository.add(task)
        return task
    }

    // MARK: - Eliminar tarea

    @discardableResult
    func removeTask(by id: UUID) -> Bool {
        repository.delete(byId: id)
    }

    // MARK: - Completar tarea

    @discardableResult
    func toggleComplete(for id: UUID) -> Bool {
        guard var task = repository.fetchById(id) else { return false }
        task.isCompleted.toggle()
        return repository.update(task)
    }

    // MARK: - Filtrar tareas

    func pendingTasks() -> [TaskItem] {
        tasks.filter { !$0.isCompleted }
    }

    func completedTasks() -> [TaskItem] {
        tasks.filter { $0.isCompleted }
    }

    func searchTasks(query: String) -> [TaskItem] {
        let lowered = query.lowercased()
        return tasks.filter {
            $0.title.lowercased().contains(lowered) ||
            ($0.taskDescription ?? "").lowercased().contains(lowered)
        }
    }

    func tasksSorted(by priority: Bool = true) -> [TaskItem] {
        if priority {
            return tasks.sorted { $0.priority > $1.priority }
        }
        return tasks.sorted { ($0.dueDate ?? .distantFuture) < ($1.dueDate ?? .distantFuture) }
    }

    // MARK: - Estadísticas

    func completionPercentage() -> Double {
        guard !tasks.isEmpty else { return 0.0 }
        let completed = Double(completedTasks().count)
        return (completed / Double(tasks.count)) * 100.0
    }
}
```

3. **Ejecuta ⌘ + U** inmediatamente.

**Resultado esperado:** 20 pruebas ejecutadas (15 unitarias + 5 UI), todas aprobadas. Las pruebas de `TaskManagerTests` no cambian: `TaskManager()` usa por defecto un `InMemoryTaskRepository` vacío, que conserva el orden de inserción igual que el array original.

### Paso 11 — Refactorización 2: Extraer `TaskService` (SRP)

**Objetivo:** Sacar del ViewController la lógica de datos (preferencias, búsqueda, borrado y estado) a una clase `TaskService`.

**Instrucciones:**

1. Crea `TaskService.swift` en el grupo `Services` (target `TaskManagerApp`):

```swift
// TaskService.swift — Target: TaskManagerApp

import Foundation

/// Lógica de la lista de tareas que antes vivía en TaskListViewController.
final class TaskService {

    private let taskManager: TaskManager
    private let preferences: UserPreferencesService

    init(taskManager: TaskManager, preferences: UserPreferencesService = .shared) {
        self.taskManager = taskManager
        self.preferences = preferences
    }

    // MARK: - Operaciones de escritura

    @discardableResult
    func createTask(title: String, description: String = "", dueDate: Date? = nil, priority: Priority = .medium) throws -> TaskItem {
        try taskManager.addTask(title: title, description: description, dueDate: dueDate, priority: priority)
    }

    @discardableResult
    func deleteTask(by id: UUID) -> Bool {
        taskManager.removeTask(by: id)
    }

    @discardableResult
    func toggleTaskCompletion(for id: UUID) -> Bool {
        taskManager.toggleComplete(for: id)
    }

    // MARK: - Operaciones de lectura

    /// Tareas a mostrar: búsqueda + filtro y orden de las preferencias (Lab 03).
    func visibleTasks(matching query: String = "") -> [TaskItem] {
        var result = query.isEmpty ? taskManager.tasks : taskManager.searchTasks(query: query)

        if !preferences.showCompletedTasks {
            result = result.filter { !$0.isCompleted }
        }

        switch preferences.sortOrder {
        case .createdAtAscending:
            result.sort { $0.createdAt < $1.createdAt }
        case .createdAtDescending:
            result.sort { $0.createdAt > $1.createdAt }
        case .titleAscending:
            result.sort { $0.title.localizedCaseInsensitiveCompare($1.title) == .orderedAscending }
        case .titleDescending:
            result.sort { $0.title.localizedCaseInsensitiveCompare($1.title) == .orderedDescending }
        }
        return result
    }

    func completionPercentage() -> Double {
        taskManager.completionPercentage()
    }
}
```

2. En `TaskListViewController.swift`, agrega el servicio junto a `repository`. Usa el **mismo** repositorio, para que el formulario modal y la sincronización (Lab 04), que siguen usando `repository`, vean los mismos datos:

```swift
lazy var taskService = TaskService(taskManager: TaskManager(repository: repository))
```

3. Reemplaza `loadTasks()` (el filtro y el orden del Lab 03 y la búsqueda del Paso 6 quedan dentro de `TaskService`):

```swift
func loadTasks() {
    tasks = taskService.visibleTasks(matching: searchQuery)
    tableView.reloadData()
}
```

4. En la acción de swipe "Eliminar", cambia la llamada al repositorio por el servicio:

```swift
// ANTES:
// self.repository.delete(byId: task.id)

// DESPUÉS:
self.taskService.deleteTask(by: task.id)
```

5. **Ejecuta ⌘ + U** para verificar que todas las pruebas (unitarias y de UI) siguen en verde.

**Resultado esperado:** 20 pruebas ejecutadas (15 unitarias + 5 UI), todas aprobadas. La refactorización no introdujo regresiones.

### Paso 12 — Agregar pruebas unitarias para el repositorio

**Objetivo:** Verificar con pruebas el `InMemoryTaskRepository` del Lab 02, la implementación que ahora usa `TaskManager` por defecto.

**Instrucciones:**

1. Crea `TaskRepositoryTests.swift` en el target `TaskManagerAppTests`. `InMemoryTaskRepository()` sin argumentos empieza vacío (`seedSampleData` es `false` por defecto):

```swift
// TaskRepositoryTests.swift — Target: TaskManagerAppTests

import XCTest
@testable import TaskManagerApp

final class TaskRepositoryTests: XCTestCase {

    var sut: InMemoryTaskRepository!

    override func setUp() {
        super.setUp()
        sut = InMemoryTaskRepository()
    }

    override func tearDown() {
        sut = nil
        super.tearDown()
    }

    func testAdd_AddsTaskToRepository() {
        let task = TaskItem(title: "Test task")
        sut.add(task)
        XCTAssertEqual(sut.count, 1)
        XCTAssertEqual(sut.fetchAll().first?.title, "Test task")
    }

    func testDeleteById_RemovesTaskFromRepository() {
        let task = TaskItem(title: "To delete")
        sut.add(task)
        let result = sut.delete(byId: task.id)
        XCTAssertTrue(result)
        XCTAssertEqual(sut.count, 0)
    }

    func testUpdate_ModifiesExistingTask() {
        var task = TaskItem(title: "Original")
        sut.add(task)
        task.title = "Modificada"
        let result = sut.update(task)
        XCTAssertTrue(result)
        XCTAssertEqual(sut.fetchAll().first?.title, "Modificada")
    }

    func testFetchById_WithValidID_ReturnsTask() {
        let task = TaskItem(title: "Findable")
        sut.add(task)
        let found = sut.fetchById(task.id)
        XCTAssertNotNil(found)
        XCTAssertEqual(found?.title, "Findable")
    }

    func testFetchById_WithInvalidID_ReturnsNil() {
        XCTAssertNil(sut.fetchById(UUID()))
    }
}
```

2. Ejecuta **⌘ + U**.

**Resultado esperado:** 25 pruebas totales (15 originales + 5 repositorio + 5 UI), todas aprobadas.

### Paso 13 — Habilitar Code Coverage y analizar resultados

**Objetivo:** Medir la cobertura de código y verificar que se alcanza ≥ 70 % en modelos y servicios.

**Instrucciones:**

1. En Xcode, ve a **Product → Scheme → Edit Scheme…** (o presiona **⌘ + <**).
2. Selecciona la acción **Test** en la barra lateral izquierda.
3. Haz clic en la pestaña **Options**.
4. Marca la casilla **Code Coverage** y selecciona **Gather coverage for: some targets**.
5. Haz clic en el botón **+** y agrega el target `TaskManagerApp`.
6. Haz clic en **Close**.

7. Ejecuta **⌘ + U** para que Xcode recopile datos de cobertura.

8. Una vez completadas las pruebas, ve a **Report Navigator** (⌘ + 9).
9. Selecciona el reporte de pruebas más reciente.
10. Haz clic en la pestaña **Coverage** en la parte superior.

**Resultado esperado:** Verás un reporte con el porcentaje de cobertura por archivo:

| Archivo | Cobertura Esperada |
|---|---|
| `TaskItem.swift` | ≥ 90 % |
| `TaskManager.swift` | ≥ 80 % |
| `TaskService.swift` | ≥ 70 % |
| `TaskRepository.swift` (`InMemoryTaskRepository`, Lab 02) | ≥ 90 % |
| `TaskListViewController.swift` | Variable (las pruebas de UI contribuyen) |

**Verificación:**
- Los archivos de modelo y servicio (`TaskItem.swift`, `TaskManager.swift`, `TaskService.swift`, `TaskRepository.swift`) deben mostrar ≥ 70 % de cobertura.
- Si algún archivo está por debajo del 70 %, identifica los métodos no cubiertos y considera agregar pruebas adicionales.

> **Tip:** Haz clic en un archivo en el reporte de cobertura para ver línea por línea qué código fue ejecutado (verde) y cuál no (rojo). Las barras en el margen derecho del editor de código también muestran la cobertura cuando está habilitada.

### Paso 14 — Commit final de la Fase 3

```bash
cd ~/Developer/TaskManagerApp
git add -A
git commit -m "refactor: apply DIP and SRP principles with TaskRepository injection and TaskService

- Inject the existing TaskRepository protocol into TaskManager (DIP)
- Extract list logic from TaskListViewController to TaskService (SRP)
- Add 5 unit tests for InMemoryTaskRepository
- Enable code coverage reporting
- Document refactoring decisions in REFACTORING_LOG.md"
```

Opcionalmente, sube los cambios a GitHub:

```bash
git push origin develop
```

---

## Validación y Testing Final

Ejecuta la suite completa de pruebas una última vez para confirmar la integridad del proyecto:

1. **⌘ + U** para ejecutar todas las pruebas.
2. Abre **Test Navigator** (⌘ + 6) y verifica:

| Suite | Pruebas | Estado Esperado |
|---|---|---|
| `TaskModelTests` | 4 | ✅ Todas aprobadas |
| `TaskManagerTests` | 11 | ✅ Todas aprobadas |
| `TaskRepositoryTests` | 5 | ✅ Todas aprobadas |
| `TaskManagerUITests` | 5 | ✅ Todas aprobadas |
| **Total** | **25** | **25 aprobadas, 0 fallos** |

3. Abre **Report Navigator** (⌘ + 9) → último reporte → pestaña **Coverage**:
   - `TaskItem.swift`: ≥ 70 %
   - `TaskManager.swift`: ≥ 70 %
   - `TaskService.swift`: ≥ 70 %
   - `TaskRepository.swift`: ≥ 70 %

4. Verifica el historial de Git:

```bash
git log --oneline -5
```

**Salida esperada (aproximada):**

```
abc1234 refactor: apply DIP and SRP principles with TaskRepository injection and TaskService
def5678 feat(ui-tests): add 5 UI tests for task list, creation, deletion, search, and validation
ghi9012 feat(ui): add accessibility identifiers and search to TaskListViewController
jkl3456 feat(tests): add unit tests for TaskItem model and TaskManager (15 tests)
mno7890 feat: add TaskManager business logic
```

---

## Solución de Problemas

### Problema 1: Error `@testable import TaskManagerApp` — "No such module"

**Síntomas:** Al compilar el target de pruebas, Xcode muestra el error `No such module 'TaskManagerApp'` en la línea `@testable import TaskManagerApp`.

**Causa:** El target de pruebas no tiene configurado correctamente el **Host Application**, o el nombre del módulo no coincide con el nombre del producto del target principal.

**Solución:**

1. Selecciona el target `TaskManagerAppTests` en el Project Navigator → pestaña **General**.
2. Verifica que en **Host Application** esté seleccionado `TaskManagerApp`.
3. Si el campo está vacío, selecciónalo del menú desplegable.
4. Ve a **Build Settings** del target de pruebas y busca `TEST_HOST`. Debe apuntar a `$(BUILT_PRODUCTS_DIR)/TaskManagerApp.app/$(BUNDLE_EXECUTABLE_FOLDER_PATH)/TaskManagerApp`.
5. Limpia la build con **⌘ + Shift + K** y vuelve a compilar con **⌘ + U**.

### Problema 2: Las pruebas de UI fallan con "Failed to find element" o timeout

**Síntomas:** Las pruebas de XCUITest fallan con mensajes como `Failed to find matching element` o `waitForExistence timed out` al buscar elementos por `accessibilityIdentifier`.

**Causa:** Los `accessibilityIdentifier` definidos en el código de la app no coinciden con los strings usados en las pruebas, o la jerarquía de vistas no está cargada cuando la prueba intenta interactuar.

**Solución:**

1. Verifica que cada `accessibilityIdentifier` en el ViewController coincida **exactamente** (case-sensitive) con el string usado en la prueba. Por ejemplo, `"taskTableView"` en la app debe coincidir con `app.tables["taskTableView"]` en la prueba.
2. Usa `waitForExistence(timeout: 5)` antes de interactuar con cualquier elemento para dar tiempo a que la UI se cargue.
3. Si la app no precarga datos de ejemplo, verifica que `app.launchArguments.append("-UITesting")` esté presente en `setUp()` y que `AppConfiguration.makeRepository(modelContext:)` lo lea con `ProcessInfo.processInfo.arguments.contains("-UITesting")` (Paso 6).
4. Para depurar, agrega un breakpoint en la prueba de UI y usa `po app.debugDescription` en la consola LLDB para ver la jerarquía completa de elementos accesibles.

---

## Limpieza

No se requiere limpieza destructiva. Los targets de pruebas y el código refactorizado son parte permanente del proyecto. Si deseas limpiar artefactos de compilación:

```bash
# Limpiar derived data (opcional, si experimentas problemas de caché)
rm -rf ~/Library/Developer/Xcode/DerivedData/TaskManagerApp-*

# Verificar estado del repositorio
cd ~/Developer/TaskManagerApp
git status
```

---

## Reto Opcional (KMP) — Pruebas en commonTest

> Este reto es **opcional** y no forma parte de la evaluación. Se trabaja sobre el proyecto KMP de los labs anteriores.

**Objetivo:** Probar el repositorio compartido en `commonTest` con kotlin.test, de modo que las mismas pruebas corran en Android y en iOS.

**Instrucciones:**

1. Agrega `kotlin("test")` y `kotlinx-coroutines-test` a las dependencias de `commonTest`.
2. Crea un `FakeTareasRepositorio` que implemente la interfaz `TareasRepositorio` con una lista en memoria (en código compartido no se pueden usar bibliotecas de mocks de Android).
3. Escribe pruebas con `@Test` y `assertEquals` para agregar, listar y eliminar tareas; usa `runTest` para probar funciones `suspend` y `Flow`.
4. Ejecuta las pruebas con `./gradlew :shared:allTests`.

**Resultado esperado:** las pruebas de `commonTest` pasan en Android y en iOS al ejecutar `./gradlew :shared:allTests`.

---

## Resumen

En esta práctica incremental de 180 minutos completaste las tres fases del cierre del curso:

| Fase | Logro | Archivos Creados |
|---|---|---|
| **Fase 1 — Unit Testing** | 15 pruebas unitarias con XCTestCase, patrón AAA, `setUp`/`tearDown` (20 con las del repositorio de la Fase 3) | `TaskManager.swift`, `TaskModelTests.swift`, `TaskManagerTests.swift` |
| **Fase 2 — UI Testing** | 5 pruebas de interfaz con XCUITest, accessibility identifiers, simulación de interacciones | `TaskManagerUITests.swift`, `TaskListViewController.swift` actualizado |
| **Fase 3 — Refactorización SOLID** | 3 violaciones identificadas (SRP, DIP, ISP); SRP y DIP corregidas, ISP documentada con su solución, cobertura ≥ 70 % | `TaskManager.swift` (refactorizado), `TaskService.swift`, `TaskRepositoryTests.swift`, `REFACTORING_LOG.md` |

**Conceptos clave aplicados:**

- **Patrón AAA** (Arrange – Act – Assert) para estructurar cada prueba unitaria.
- **`@testable import`** para acceder a tipos `internal` desde targets de prueba.
- **`sut` (System Under Test)** como convención de nomenclatura estándar.
- **`XCTAssertEqual`**, **`XCTAssertThrowsError`**, **`XCTAssertNil`**, **`XCTAssertTrue`** para cubrir diferentes escenarios de validación.
- **`XCUIApplication`**, **`XCUIElement`** y **accessibility identifiers** para localizar y manipular elementos de UI.
- **Principio de Responsabilidad Única (SRP):** Extraer `TaskService` del ViewController.
- **Principio de Inversión de Dependencias (DIP):** Inyectar en `TaskManager` el protocolo `TaskRepository` del Lab 02 en lugar de un array concreto.
- **Code Coverage** como métrica objetivo de calidad de las pruebas.
- **Commits convencionales** (`feat:`, `refactor:`) para documentar la evolución del proyecto.

### Recursos Adicionales

- [Documentación oficial de XCTest — Apple Developer](https://developer.apple.com/documentation/xctest)
- [Testing Your Apps in Xcode — Apple Developer](https://developer.apple.com/documentation/xcode/testing-your-apps-in-xcode)
- [UI Testing in Xcode — WWDC](https://developer.apple.com/videos/play/wwdc2015/406/)
- [SOLID Principles in Swift — Kodeco](https://www.kodeco.com/21503974-solid-principles-for-ios-apps)
- [Swift by Sundell: Unit Testing Best Practices](https://www.swiftbysundell.com/articles/unit-testing-best-practices-in-swift/)