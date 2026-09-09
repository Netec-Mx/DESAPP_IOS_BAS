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

Verifica la compilación con **⌘ + B** en Xcode. Si no tienes el proyecto de módulos anteriores, utiliza el starter project proporcionado por el instructor y clónalo en `~/Developer/TaskManagerApp`.

---

## Código Base Requerido (Starter)

Antes de iniciar las fases de testing, necesitas asegurarte de que tu proyecto contiene los archivos de modelo y lógica de negocio sobre los que escribirás pruebas. Si tu proyecto ya los tiene de módulos anteriores, verifica que coincidan en estructura. Si partes del starter project, crea los siguientes archivos.

### Paso 0 — Verificar o crear el modelo y el gestor de tareas

**Objetivo:** Garantizar que existen `Task.swift` y `TaskManager.swift` en el target principal con la estructura que las pruebas ejercitarán.

**Instrucciones:**

1. En Xcode, selecciona la carpeta `TaskManagerApp` en el Project Navigator.

2. Verifica que exista el archivo `Task.swift`. Si no existe, créalo con **File → New → File → Swift File** y nómbralo `Task`. Asegúrate de que el target `TaskManagerApp` esté marcado. Escribe el siguiente contenido:

```swift
// Task.swift — Target: TaskManagerApp

import Foundation

enum TaskPriority: Int, Comparable {
    case low = 0
    case medium = 1
    case high = 2

    static func < (lhs: TaskPriority, rhs: TaskPriority) -> Bool {
        lhs.rawValue < rhs.rawValue
    }
}

enum TaskError: Error, Equatable {
    case emptyTitle
    case invalidDate
    case duplicateTask
}

struct Task: Identifiable, Equatable {
    let id: UUID
    var title: String
    var description: String
    var isCompleted: Bool
    var dueDate: Date?
    var priority: TaskPriority

    init(
        id: UUID = UUID(),
        title: String,
        description: String = "",
        isCompleted: Bool = false,
        dueDate: Date? = nil,
        priority: TaskPriority = .medium
    ) {
        self.id = id
        self.title = title
        self.description = description
        self.isCompleted = isCompleted
        self.dueDate = dueDate
        self.priority = priority
    }
}
```

3. Verifica que exista `TaskManager.swift`. Si no existe, créalo de la misma forma con el siguiente contenido:

```swift
// TaskManager.swift — Target: TaskManagerApp

import Foundation

class TaskManager {
    private(set) var tasks: [Task] = []

    // MARK: - Agregar tarea

    func addTask(title: String, description: String = "", dueDate: Date? = nil, priority: TaskPriority = .medium) throws -> Task {
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

        let task = Task(title: trimmedTitle, description: description, dueDate: dueDate, priority: priority)
        tasks.append(task)
        return task
    }

    // MARK: - Eliminar tarea

    func removeTask(by id: UUID) -> Bool {
        guard let index = tasks.firstIndex(where: { $0.id == id }) else {
            return false
        }
        tasks.remove(at: index)
        return true
    }

    // MARK: - Completar tarea

    func toggleComplete(for id: UUID) -> Bool {
        guard let index = tasks.firstIndex(where: { $0.id == id }) else {
            return false
        }
        tasks[index].isCompleted.toggle()
        return true
    }

    // MARK: - Filtrar tareas

    func pendingTasks() -> [Task] {
        tasks.filter { !$0.isCompleted }
    }

    func completedTasks() -> [Task] {
        tasks.filter { $0.isCompleted }
    }

    func searchTasks(query: String) -> [Task] {
        let lowered = query.lowercased()
        return tasks.filter {
            $0.title.lowercased().contains(lowered) ||
            $0.description.lowercased().contains(lowered)
        }
    }

    func tasksSorted(by priority: Bool = true) -> [Task] {
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

4. Compila con **⌘ + B** para confirmar que no hay errores.

5. Haz un commit inicial si estos archivos son nuevos:

```bash
cd ~/Developer/TaskManagerApp
git add -A
git commit -m "feat: add Task model and TaskManager business logic"
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

**Objetivo:** Implementar pruebas unitarias para validar la creación y propiedades del modelo `Task`.

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
        let task = Task(title: "Comprar leche")

        // Assert
        XCTAssertEqual(task.title, "Comprar leche")
        XCTAssertEqual(task.description, "")
        XCTAssertFalse(task.isCompleted)
        XCTAssertNil(task.dueDate)
        XCTAssertEqual(task.priority, .medium)
    }

    func testTaskCreation_WithAllFields_SetsPropertiesCorrectly() {
        // Arrange
        let date = Date().addingTimeInterval(86400) // mañana

        // Act
        let task = Task(
            title: "Estudiar Swift",
            description: "Leer capítulo 6",
            isCompleted: true,
            dueDate: date,
            priority: .high
        )

        // Assert
        XCTAssertEqual(task.title, "Estudiar Swift")
        XCTAssertEqual(task.description, "Leer capítulo 6")
        XCTAssertTrue(task.isCompleted)
        XCTAssertEqual(task.dueDate, date)
        XCTAssertEqual(task.priority, .high)
    }

    func testTaskCreation_GeneratesUniqueIDs() {
        let task1 = Task(title: "Tarea 1")
        let task2 = Task(title: "Tarea 2")

        XCTAssertNotEqual(task1.id, task2.id, "Cada tarea debe tener un UUID único")
    }

    // MARK: - Pruebas de prioridad

    func testTaskPriority_HighIsGreaterThanLow() {
        XCTAssertTrue(TaskPriority.high > TaskPriority.low)
        XCTAssertTrue(TaskPriority.medium > TaskPriority.low)
        XCTAssertTrue(TaskPriority.high > TaskPriority.medium)
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

    // MARK: - Pruebas: Filtrado

    func testPendingTasks_ReturnsOnlyIncompleteTasks() throws {
        // Arrange
        let task1 = try sut.addTask(title: "Pendiente")
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
git commit -m "feat(tests): add unit tests for Task model and TaskManager (15 tests)"
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

**Objetivo:** Preparar la app para que los elementos de UI sean localizables por XCUITest.

**Instrucciones:**

Las pruebas de UI necesitan identificar elementos por `accessibilityIdentifier`. Debes agregar estos identificadores en el código de la interfaz. Crea o modifica el ViewController principal de la app.

1. Si tu app usa UIKit con Storyboard, localiza el ViewController que muestra la lista de tareas. Si no tienes uno funcional, crea `TaskListViewController.swift` en el target `TaskManagerApp`:

```swift
// TaskListViewController.swift — Target: TaskManagerApp

import UIKit

class TaskListViewController: UITableViewController {

    var taskManager = TaskManager()

    override func viewDidLoad() {
        super.viewDidLoad()
        title = "Mis Tareas"
        navigationItem.rightBarButtonItem = UIBarButtonItem(
            barButtonSystemItem: .add,
            target: self,
            action: #selector(addTaskTapped)
        )

        // Accessibility Identifiers
        tableView.accessibilityIdentifier = "taskTableView"
        navigationItem.rightBarButtonItem?.accessibilityIdentifier = "addTaskButton"

        // Configurar search controller
        let searchController = UISearchController(searchResultsController: nil)
        searchController.searchBar.accessibilityIdentifier = "taskSearchBar"
        searchController.searchBar.placeholder = "Buscar tareas..."
        searchController.searchResultsUpdater = self
        navigationItem.searchController = searchController

        tableView.register(UITableViewCell.self, forCellReuseIdentifier: "TaskCell")

        // Agregar tareas de ejemplo para UI Testing
        if ProcessInfo.processInfo.arguments.contains("-UITesting") {
            try? taskManager.addTask(title: "Tarea de ejemplo", description: "Descripción de prueba", priority: .high)
            try? taskManager.addTask(title: "Comprar víveres", description: "Ir al supermercado", priority: .medium)
            try? taskManager.addTask(title: "Estudiar iOS", description: "Practicar XCTest", priority: .low)
        }
    }

    @objc func addTaskTapped() {
        let alert = UIAlertController(title: "Nueva Tarea", message: nil, preferredStyle: .alert)
        alert.addTextField { textField in
            textField.placeholder = "Título de la tarea"
            textField.accessibilityIdentifier = "taskTitleTextField"
        }
        alert.addAction(UIAlertAction(title: "Cancelar", style: .cancel))
        alert.addAction(UIAlertAction(title: "Guardar", style: .default) { [weak self] _ in
            guard let title = alert.textFields?.first?.text else { return }
            do {
                try self?.taskManager.addTask(title: title)
                self?.tableView.reloadData()
            } catch TaskError.emptyTitle {
                self?.showError("El título no puede estar vacío")
            } catch TaskError.duplicateTask {
                self?.showError("Ya existe una tarea con ese nombre")
            } catch {
                self?.showError("Error desconocido")
            }
        })
        alert.view.accessibilityIdentifier = "addTaskAlert"
        present(alert, animated: true)
    }

    func showError(_ message: String) {
        let errorAlert = UIAlertController(title: "Error", message: message, preferredStyle: .alert)
        errorAlert.addAction(UIAlertAction(title: "OK", style: .default))
        errorAlert.view.accessibilityIdentifier = "errorAlert"
        present(errorAlert, animated: true)
    }

    // MARK: - Table View Data Source

    var filteredTasks: [Task]?

    var displayedTasks: [Task] {
        filteredTasks ?? taskManager.tasks
    }

    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        let count = displayedTasks.count
        if count == 0 {
            let label = UILabel()
            label.text = "No hay tareas"
            label.textAlignment = .center
            label.accessibilityIdentifier = "emptyStateLabel"
            tableView.backgroundView = label
        } else {
            tableView.backgroundView = nil
        }
        return count
    }

    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "TaskCell", for: indexPath)
        let task = displayedTasks[indexPath.row]
        cell.textLabel?.text = task.title
        cell.accessoryType = task.isCompleted ? .checkmark : .none
        cell.accessibilityIdentifier = "taskCell_\(indexPath.row)"
        return cell
    }

    override func tableView(_ tableView: UITableView, commit editingStyle: UITableViewCell.EditingStyle, forRowAt indexPath: IndexPath) {
        if editingStyle == .delete {
            let task = displayedTasks[indexPath.row]
            _ = taskManager.removeTask(by: task.id)
            filteredTasks = nil
            tableView.deleteRows(at: [indexPath], with: .automatic)
        }
    }

    override func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        let task = displayedTasks[indexPath.row]
        _ = taskManager.toggleComplete(for: task.id)
        tableView.reloadRows(at: [indexPath], with: .automatic)
    }
}

// MARK: - UISearchResultsUpdating

extension TaskListViewController: UISearchResultsUpdating {
    func updateSearchResults(for searchController: UISearchController) {
        if let query = searchController.searchBar.text, !query.isEmpty {
            filteredTasks = taskManager.searchTasks(query: query)
        } else {
            filteredTasks = nil
        }
        tableView.reloadData()
    }
}
```

2. Asegúrate de que el `SceneDelegate.swift` (o `AppDelegate.swift`) establece este ViewController como root. En `SceneDelegate.swift`:

```swift
// SceneDelegate.swift — dentro de scene(_:willConnectTo:options:)

func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options connectionOptions: UIScene.ConnectionOptions) {
    guard let windowScene = (scene as? UIWindowScene) else { return }
    let window = UIWindow(windowScene: windowScene)
    let taskListVC = TaskListViewController(style: .plain)
    let navController = UINavigationController(rootViewController: taskListVC)
    window.rootViewController = navController
    window.makeKeyAndVisible()
    self.window = window
}
```

> **Nota:** Si tu proyecto usa Storyboards, puedes configurar el ViewController en el Storyboard y agregar los `accessibilityIdentifier` en `viewDidLoad()`. Lo importante es que los identifiers coincidan con los que usarán las pruebas de UI.

3. Compila con **⌘ + B** y ejecuta en el simulador **iPhone 15 Pro (iOS 17.5)** con **⌘ + R** para verificar que la app muestra la lista de tareas.

4. Haz commit:

```bash
git add -A
git commit -m "feat(ui): add TaskListViewController with accessibility identifiers for UI testing"
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

        // Escribir título en el alert
        let titleTextField = app.textFields["taskTitleTextField"]
        XCTAssertTrue(titleTextField.waitForExistence(timeout: 3), "El campo de texto debe aparecer")
        titleTextField.tap()
        titleTextField.typeText("Tarea UI Test")

        // Guardar
        app.buttons["Guardar"].tap()

        // Assert: La nueva tarea aparece en la lista
        let newCount = table.cells.count
        XCTAssertEqual(newCount, initialCount + 1, "La lista debe tener una tarea más")
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

        // Pulsar botón Delete
        let deleteButton = table.buttons["Delete"]
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

        app.buttons["Guardar"].tap()

        // Assert: Debe aparecer un alert de error
        let errorAlert = app.alerts.firstMatch
        XCTAssertTrue(errorAlert.waitForExistence(timeout: 3), "Debe mostrarse una alerta de error")

        // Verificar el mensaje
        let errorMessage = errorAlert.staticTexts["El título no puede estar vacío"]
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
- **Problema:** El ViewController gestiona la lógica de presentación (tabla, celdas,
  navegación), la lógica de negocio (agregar/eliminar tareas a través de TaskManager)
  y la lógica de búsqueda. Tiene demasiadas responsabilidades.
- **Solución propuesta:** Extraer la lógica de negocio a una clase `TaskService`
  separada que actúe como intermediario.

### Violación 2: Principio de Inversión de Dependencias (DIP)
- **Archivo:** `TaskManager.swift`
- **Problema:** `TaskManager` almacena las tareas directamente en un array en
  memoria. Si en el futuro necesitamos persistir con CoreData, UserDefaults o una
  API remota, habría que modificar `TaskManager` directamente. Las clases de alto
  nivel dependen de una implementación concreta de almacenamiento.
- **Solución propuesta:** Crear un protocolo `TaskRepositoryProtocol` que defina
  las operaciones de persistencia, e inyectar la dependencia en `TaskManager`.

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

### Paso 10 — Refactorización 1: Extraer TaskService (SRP)

**Objetivo:** Separar la lógica de negocio del ViewController creando una clase `TaskService`.

**Instrucciones:**

1. Crea el archivo `TaskService.swift` en el target `TaskManagerApp`:

```swift
// TaskService.swift — Target: TaskManagerApp

import Foundation

class TaskService {

    private let taskManager: TaskManager

    init(taskManager: TaskManager = TaskManager()) {
        self.taskManager = taskManager
    }

    // MARK: - Operaciones de escritura

    @discardableResult
    func createTask(title: String, description: String = "", dueDate: Date? = nil, priority: TaskPriority = .medium) throws -> Task {
        return try taskManager.addTask(title: title, description: description, dueDate: dueDate, priority: priority)
    }

    func deleteTask(by id: UUID) -> Bool {
        return taskManager.removeTask(by: id)
    }

    func toggleTaskCompletion(for id: UUID) -> Bool {
        return taskManager.toggleComplete(for: id)
    }

    // MARK: - Operaciones de lectura

    var allTasks: [Task] {
        taskManager.tasks
    }

    func pendingTasks() -> [Task] {
        taskManager.pendingTasks()
    }

    func completedTasks() -> [Task] {
        taskManager.completedTasks()
    }

    func search(query: String) -> [Task] {
        taskManager.searchTasks(query: query)
    }

    func completionPercentage() -> Double {
        taskManager.completionPercentage()
    }
}
```

2. Actualiza `TaskListViewController.swift` para usar `TaskService` en lugar de `TaskManager` directamente. Reemplaza la propiedad:

```swift
// ANTES:
// var taskManager = TaskManager()

// DESPUÉS:
var taskService = TaskService()
```

3. Actualiza todas las referencias en el ViewController. Los cambios clave son:

```swift
// En viewDidLoad(), cambiar las llamadas de precarga:
if ProcessInfo.processInfo.arguments.contains("-UITesting") {
    try? taskService.createTask(title: "Tarea de ejemplo", description: "Descripción de prueba", priority: .high)
    try? taskService.createTask(title: "Comprar víveres", description: "Ir al supermercado", priority: .medium)
    try? taskService.createTask(title: "Estudiar iOS", description: "Practicar XCTest", priority: .low)
}

// En addTaskTapped(), cambiar:
try self?.taskService.createTask(title: title)

// En showError — sin cambios

// En displayedTasks:
var displayedTasks: [Task] {
    filteredTasks ?? taskService.allTasks
}

// En commit editingStyle (delete):
_ = taskService.deleteTask(by: task.id)

// En didSelectRowAt:
_ = taskService.toggleTaskCompletion(for: task.id)

// En updateSearchResults:
filteredTasks = taskService.search(query: query)
```

4. **Ejecuta ⌘ + U** para verificar que todas las pruebas (unitarias y de UI) siguen en verde.

**Resultado esperado:** 20 pruebas ejecutadas (15 unitarias + 5 UI), todas aprobadas. La refactorización no introdujo regresiones.

### Paso 11 — Refactorización 2: Crear TaskRepositoryProtocol (DIP)

**Objetivo:** Invertir la dependencia de almacenamiento mediante un protocolo.

**Instrucciones:**

1. Crea `TaskRepositoryProtocol.swift` en el target `TaskManagerApp`:

```swift
// TaskRepositoryProtocol.swift — Target: TaskManagerApp

import Foundation

protocol TaskRepositoryProtocol {
    var tasks: [Task] { get }
    func save(_ task: Task)
    func delete(by id: UUID) -> Bool
    func update(_ task: Task)
    func findAll() -> [Task]
    func find(by id: UUID) -> Task?
}
```

2. Crea `InMemoryTaskRepository.swift` como implementación concreta:

```swift
// InMemoryTaskRepository.swift — Target: TaskManagerApp

import Foundation

class InMemoryTaskRepository: TaskRepositoryProtocol {

    private(set) var tasks: [Task] = []

    func save(_ task: Task) {
        tasks.append(task)
    }

    func delete(by id: UUID) -> Bool {
        guard let index = tasks.firstIndex(where: { $0.id == id }) else {
            return false
        }
        tasks.remove(at: index)
        return true
    }

    func update(_ task: Task) {
        guard let index = tasks.firstIndex(where: { $0.id == task.id }) else { return }
        tasks[index] = task
    }

    func findAll() -> [Task] {
        return tasks
    }

    func find(by id: UUID) -> Task? {
        return tasks.first(where: { $0.id == id })
    }
}
```

3. Refactoriza `TaskManager.swift` para depender del protocolo en lugar de un array interno:

```swift
// TaskManager.swift — Refactorizado con inyección de dependencias

import Foundation

class TaskManager {

    private let repository: TaskRepositoryProtocol

    var tasks: [Task] {
        repository.findAll()
    }

    init(repository: TaskRepositoryProtocol = InMemoryTaskRepository()) {
        self.repository = repository
    }

    // MARK: - Agregar tarea

    func addTask(title: String, description: String = "", dueDate: Date? = nil, priority: TaskPriority = .medium) throws -> Task {
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

        let task = Task(title: trimmedTitle, description: description, dueDate: dueDate, priority: priority)
        repository.save(task)
        return task
    }

    // MARK: - Eliminar tarea

    func removeTask(by id: UUID) -> Bool {
        return repository.delete(by: id)
    }

    // MARK: - Completar tarea

    func toggleComplete(for id: UUID) -> Bool {
        guard var task = repository.find(by: id) else { return false }
        task.isCompleted.toggle()
        repository.update(task)
        return true
    }

    // MARK: - Filtrar tareas

    func pendingTasks() -> [Task] {
        tasks.filter { !$0.isCompleted }
    }

    func completedTasks() -> [Task] {
        tasks.filter { $0.isCompleted }
    }

    func searchTasks(query: String) -> [Task] {
        let lowered = query.lowercased()
        return tasks.filter {
            $0.title.lowercased().contains(lowered) ||
            $0.description.lowercased().contains(lowered)
        }
    }

    func tasksSorted(by priority: Bool = true) -> [Task] {
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

4. **Ejecuta ⌘ + U** inmediatamente.

**Resultado esperado:** Todas las pruebas siguen en verde. La inyección de dependencias no alteró el comportamiento porque `InMemoryTaskRepository` replica el almacenamiento original en array.

### Paso 12 — Agregar pruebas unitarias para el repositorio

**Objetivo:** Verificar la nueva capa de abstracción con pruebas específicas.

**Instrucciones:**

1. Crea `TaskRepositoryTests.swift` en el target `TaskManagerAppTests`:

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

    func testSave_AddsTaskToRepository() {
        let task = Task(title: "Test task")
        sut.save(task)
        XCTAssertEqual(sut.tasks.count, 1)
        XCTAssertEqual(sut.tasks.first?.title, "Test task")
    }

    func testDelete_RemovesTaskFromRepository() {
        let task = Task(title: "To delete")
        sut.save(task)
        let result = sut.delete(by: task.id)
        XCTAssertTrue(result)
        XCTAssertEqual(sut.tasks.count, 0)
    }

    func testUpdate_ModifiesExistingTask() {
        var task = Task(title: "Original")
        sut.save(task)
        task.title = "Modificada"
        sut.update(task)
        XCTAssertEqual(sut.tasks.first?.title, "Modificada")
    }

    func testFind_WithValidID_ReturnsTask() {
        let task = Task(title: "Findable")
        sut.save(task)
        let found = sut.find(by: task.id)
        XCTAssertNotNil(found)
        XCTAssertEqual(found?.title, "Findable")
    }

    func testFind_WithInvalidID_ReturnsNil() {
        XCTAssertNil(sut.find(by: UUID()))
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
| `Task.swift` | ≥ 90 % |
| `TaskManager.swift` | ≥ 80 % |
| `TaskService.swift` | ≥ 70 % |
| `InMemoryTaskRepository.swift` | ≥ 90 % |
| `TaskListViewController.swift` | Variable (las pruebas de UI contribuyen) |

**Verificación:**
- Los archivos de modelo y servicio (`Task.swift`, `TaskManager.swift`, `TaskService.swift`, `InMemoryTaskRepository.swift`) deben mostrar ≥ 70 % de cobertura.
- Si algún archivo está por debajo del 70 %, identifica los métodos no cubiertos y considera agregar pruebas adicionales.

> **Tip:** Haz clic en un archivo en el reporte de cobertura para ver línea por línea qué código fue ejecutado (verde) y cuál no (rojo). Las barras en el margen derecho del editor de código también muestran la cobertura cuando está habilitada.

### Paso 14 — Commit final de la Fase 3

```bash
cd ~/Developer/TaskManagerApp
git add -A
git commit -m "refactor: apply SRP and DIP principles, add TaskService and TaskRepositoryProtocol

- Extract business logic from ViewController to TaskService (SRP)
- Create TaskRepositoryProtocol for dependency inversion (DIP)
- Add InMemoryTaskRepository as concrete implementation
- Add 5 repository unit tests
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
   - `Task.swift`: ≥ 70 %
   - `TaskManager.swift`: ≥ 70 %
   - `TaskService.swift`: ≥ 70 %
   - `InMemoryTaskRepository.swift`: ≥ 70 %

4. Verifica el historial de Git:

```bash
git log --oneline -5
```

**Salida esperada (aproximada):**

```
abc1234 refactor: apply SRP and DIP principles, add TaskService and TaskRepositoryProtocol
def5678 feat(ui-tests): add 5 UI tests for task list, creation, deletion, search, and validation
ghi9012 feat(ui): add TaskListViewController with accessibility identifiers for UI testing
jkl3456 feat(tests): add unit tests for Task model and TaskManager (15 tests)
mno7890 feat: add Task model and TaskManager business logic
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
3. Si la app no precarga datos de ejemplo, verifica que `app.launchArguments.append("-UITesting")` esté presente en `setUp()` y que el ViewController lo lea con `ProcessInfo.processInfo.arguments.contains("-UITesting")`.
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

## Resumen

En esta práctica incremental de 180 minutos completaste las tres fases del cierre del curso:

| Fase | Logro | Archivos Creados |
|---|---|---|
| **Fase 1 — Unit Testing** | 20 pruebas unitarias con XCTestCase, patrón AAA, `setUp`/`tearDown` | `TaskModelTests.swift`, `TaskManagerTests.swift`, `TaskRepositoryTests.swift` |
| **Fase 2 — UI Testing** | 5 pruebas de interfaz con XCUITest, accessibility identifiers, simulación de interacciones | `TaskManagerUITests.swift`, `TaskListViewController.swift` actualizado |
| **Fase 3 — Refactorización SOLID** | 3 violaciones identificadas y corregidas (SRP, DIP, ISP), cobertura ≥ 70 % | `TaskService.swift`, `TaskRepositoryProtocol.swift`, `InMemoryTaskRepository.swift`, `REFACTORING_LOG.md` |

**Conceptos clave aplicados:**

- **Patrón AAA** (Arrange – Act – Assert) para estructurar cada prueba unitaria.
- **`@testable import`** para acceder a tipos `internal` desde targets de prueba.
- **`sut` (System Under Test)** como convención de nomenclatura estándar.
- **`XCTAssertEqual`**, **`XCTAssertThrowsError`**, **`XCTAssertNil`**, **`XCTAssertTrue`** para cubrir diferentes escenarios de validación.
- **`XCUIApplication`**, **`XCUIElement`** y **accessibility identifiers** para localizar y manipular elementos de UI.
- **Principio de Responsabilidad Única (SRP):** Extraer `TaskService` del ViewController.
- **Principio de Inversión de Dependencias (DIP):** Crear `TaskRepositoryProtocol` e inyectar implementaciones.
- **Code Coverage** como métrica objetiva de calidad de las pruebas.
- **Commits convencionales** (`feat:`, `refactor:`) para documentar la evolución del proyecto.

### Recursos Adicionales

- [Documentación oficial de XCTest — Apple Developer](https://developer.apple.com/documentation/xctest)
- [Testing Your Apps in Xcode — Apple Developer](https://developer.apple.com/documentation/xcode/testing-your-apps-in-xcode)
- [UI Testing in Xcode — WWDC](https://developer.apple.com/videos/play/wwdc2015/406/)
- [SOLID Principles in Swift — Kodeco](https://www.kodeco.com/21503974-solid-principles-for-ios-apps)
- [Swift by Sundell: Unit Testing Best Practices](https://www.swiftbysundell.com/articles/unit-testing-best-practices-in-swift/)
