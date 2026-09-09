# 5 Práctica incremental — Interfaces UIKit y SwiftUI para TaskManagerApp

## Metadatos

| Campo | Valor |
|---|---|
| **Duración** | 252 minutos (UIKit ≈ 140 min · SwiftUI ≈ 112 min) |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |

---

## Descripción General

En esta práctica construirás **dos flujos de interfaz de usuario completos y paralelos** para la aplicación TaskManagerApp: uno con **UIKit** (Storyboards, UITableViewController, UINavigationController, Auto Layout) y otro con **SwiftUI** (List, NavigationStack, @State, @Binding). Ambos flujos compartirán la capa de modelo (`Task` y `TaskRepository`) creada en el Lab 01. Trabajarás en una rama Git dedicada (`feature/ui-module`) y realizarás commits atómicos siguiendo Conventional Commits.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio serás capaz de:

- [ ] Construir la interfaz principal de la app con UIKit: lista de tareas usando `UITableViewController` con Auto Layout y el simulador iPhone 15 Pro.
- [ ] Implementar navegación jerárquica con `UINavigationController` y segues (show/modal) para la pantalla de detalle y creación de tareas.
- [ ] Crear una segunda interfaz equivalente usando SwiftUI con `List`, `NavigationStack`, `VStack`, `HStack` y Modifiers.
- [ ] Gestionar estado en SwiftUI con `@State` y `@Binding` para el formulario de creación de tareas.
- [ ] Integrar la capa de modelo (`struct Task` y protocolo `TaskRepository`) del Lab 01 con ambas interfaces.

---

## Prerrequisitos

### Conocimiento

- Lab 01-00-01 completado: proyecto TaskManagerApp con `struct Task`, protocolo `TaskRepository` e `InMemoryTaskRepository`.
- Rama `develop` activa en el repositorio local con los commits del Lab 01.
- Conocimiento básico de Auto Layout, Storyboards y vistas SwiftUI (cursos prerrequisito).

### Acceso

- Mac con macOS Ventura 13.5+ o Sonoma 14.5.
- Xcode 15.4 instalado con simulador iOS 17.5 (iPhone 15 Pro).
- Repositorio Git local del proyecto TaskManagerApp en `~/Developer/TaskManagerApp`.

---

## Entorno de Laboratorio

### Hardware

| Requisito | Mínimo |
|---|---|
| Procesador | Apple Silicon (M1/M2/M3) o Intel Core i5 8.ª gen+ |
| RAM | 16 GB |
| Disco libre | 50 GB |
| Resolución | 1440 × 900 (recomendado 1920 × 1080+) |

### Software

| Herramienta | Versión |
|---|---|
| macOS | Ventura 13.6.7 o Sonoma 14.5 |
| Xcode | 15.4 |
| Swift | 5.10 |
| Simulador | iOS 17.5 — iPhone 15 Pro |
| Git | 2.45.2 |

### Preparación Inicial

Abre Terminal y ejecuta los siguientes comandos para verificar el estado del proyecto:

```bash
cd ~/Developer/TaskManagerApp
git status
git branch
```

Debes estar en la rama `develop` con un directorio de trabajo limpio. Si hay cambios sin confirmar, haz commit o stash antes de continuar.

---

## Paso 1 — Crear la Rama de Trabajo `feature/ui-module`

**Objetivo:** Crear una rama de feature desde `develop` para aislar todo el trabajo de UI.

### Instrucciones

1. En Terminal, asegúrate de estar en `develop`:

```bash
cd ~/Developer/TaskManagerApp
git checkout develop
```

2. Crea la nueva rama y cámbiate a ella:

```bash
git checkout -b feature/ui-module
```

3. Verifica la rama activa:

```bash
git branch
```

### Salida Esperada

```
  develop
* feature/ui-module
  main
```

### Verificación

El asterisco (`*`) debe estar junto a `feature/ui-module`. Todos los commits de este lab se harán sobre esta rama.

---

## Paso 2 — Verificar el Modelo de Datos del Lab 01

**Objetivo:** Confirmar que los archivos de modelo existen y compilar el proyecto para partir de una base estable.

### Instrucciones

1. Abre el proyecto en Xcode:

```bash
open TaskManagerApp.xcodeproj
```

2. En el **Project Navigator** (⌘1), verifica que existan los siguientes archivos dentro del grupo `Models/` (o la carpeta donde los creaste en el Lab 01):

   - `Task.swift` — contiene `struct Task` con propiedades `id`, `title`, `description`, `isCompleted` y `createdAt`.
   - `TaskRepository.swift` — contiene el protocolo `TaskRepository` con métodos `getAll()`, `add(_:)`, `update(_:)`, `delete(id:)`.
   - `InMemoryTaskRepository.swift` — implementación en memoria del protocolo.

3. Si el archivo `InMemoryTaskRepository.swift` no existe aún, créalo ahora. Haz clic derecho en el grupo `Models/` → **New File…** → **Swift File** → nombre: `InMemoryTaskRepository`. Añade el siguiente contenido:

```swift
import Foundation

class InMemoryTaskRepository: TaskRepository {
    
    private var tasks: [Task] = []
    
    init() {
        // Datos de ejemplo para desarrollo
        tasks = [
            Task(
                id: UUID(),
                title: "Comprar víveres",
                description: "Leche, pan, huevos y frutas",
                isCompleted: false,
                createdAt: Date()
            ),
            Task(
                id: UUID(),
                title: "Estudiar Swift",
                description: "Repasar protocolos y genéricos",
                isCompleted: true,
                createdAt: Date().addingTimeInterval(-86400)
            ),
            Task(
                id: UUID(),
                title: "Hacer ejercicio",
                description: "Correr 5 km en el parque",
                isCompleted: false,
                createdAt: Date().addingTimeInterval(-172800)
            )
        ]
    }
    
    func getAll() -> [Task] {
        return tasks
    }
    
    func add(_ task: Task) {
        tasks.append(task)
    }
    
    func update(_ task: Task) {
        if let index = tasks.firstIndex(where: { $0.id == task.id }) {
            tasks[index] = task
        }
    }
    
    func delete(id: UUID) {
        tasks.removeAll { $0.id == id }
    }
}
```

4. Asegúrate de que `Task.swift` tenga al menos esta estructura (ajusta si tu Lab 01 difiere ligeramente):

```swift
import Foundation

struct Task: Identifiable {
    let id: UUID
    var title: String
    var description: String
    var isCompleted: Bool
    var createdAt: Date
}
```

5. Asegúrate de que `TaskRepository.swift` contenga:

```swift
import Foundation

protocol TaskRepository {
    func getAll() -> [Task]
    func add(_ task: Task)
    func update(_ task: Task)
    func delete(id: UUID)
}
```

6. Compila el proyecto con **⌘B**. Debe compilar sin errores.

### Salida Esperada

En la barra de estado de Xcode: **Build Succeeded**.

### Verificación

Si hay errores de compilación, revisa que los nombres de tipos y métodos coincidan exactamente entre los tres archivos. Corrige antes de continuar.

---

## Paso 3 — Configurar el Storyboard Principal con UINavigationController

**Objetivo:** Establecer `UINavigationController` como controlador raíz y crear la escena de lista de tareas.

### Instrucciones

1. En el Project Navigator, abre `Main.storyboard`.

2. Si ya existe un `ViewController` genérico en el lienzo, selecciónalo y elimínalo (tecla **Delete**).

3. Abre la **Object Library** con **⇧⌘L**. Busca `Table View Controller` y arrástralo al lienzo.

4. Con el Table View Controller seleccionado, ve al menú **Editor → Embed In → Navigation Controller**. Xcode creará un `UINavigationController` conectado al Table View Controller.

5. Selecciona el **Navigation Controller** recién creado. En el **Attributes Inspector** (⌥⌘4), marca la casilla **Is Initial View Controller** (verás la flecha gris apuntando al Navigation Controller).

6. Selecciona el **Table View Controller**. En el **Identity Inspector** (⌥⌘3), establece la **Class** como `TaskListViewController` (la crearemos en el siguiente paso).

7. Haz clic en la barra de navegación del Table View Controller (la barra superior dentro de la escena). En el **Attributes Inspector**, escribe `Mis Tareas` en el campo **Title**.

8. En el **Document Outline** del Storyboard, selecciona la **Table View Cell** que aparece dentro del Table View Controller. En el **Attributes Inspector**, establece el **Identifier** (Reuse Identifier) como `TaskCell` y el **Style** como `Subtitle`.

9. Guarda el Storyboard con **⌘S**.

### Salida Esperada

El lienzo del Storyboard muestra: **Navigation Controller** (con flecha de inicio) → **Table View Controller** con título "Mis Tareas" y una celda prototipo con identificador `TaskCell`.

### Verificación

En el Document Outline, la jerarquía debe ser:
- Navigation Controller Scene
  - Navigation Controller
- Task List View Controller Scene (con Class = `TaskListViewController`)
  - Table View
    - Table View Cell (Identifier = `TaskCell`)

---

## Paso 4 — Crear TaskListViewController (UIKit)

**Objetivo:** Implementar el controlador de la lista de tareas que muestra datos del repositorio en un UITableView.

### Instrucciones

1. Crea un grupo para los controladores UIKit. En el Project Navigator, haz clic derecho en el grupo raíz `TaskManagerApp` → **New Group** → nómbralo `UIKitViews`.

2. Haz clic derecho en `UIKitViews/` → **New File…** → **Cocoa Touch Class**.
   - Class: `TaskListViewController`
   - Subclass of: `UITableViewController`
   - Language: Swift
   - Desmarca "Also create XIB file"
   - Haz clic en **Next** → **Create**.

3. Reemplaza todo el contenido del archivo con:

```swift
import UIKit

class TaskListViewController: UITableViewController {
    
    // MARK: - Properties
    
    private let repository: TaskRepository = InMemoryTaskRepository()
    private var tasks: [Task] = []
    
    // MARK: - Lifecycle
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        title = "Mis Tareas"
        navigationController?.navigationBar.prefersLargeTitles = true
        
        configureNavigationBar()
        loadTasks()
    }
    
    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        loadTasks()
    }
    
    // MARK: - Configuration
    
    private func configureNavigationBar() {
        let addButton = UIBarButtonItem(
            barButtonSystemItem: .add,
            target: self,
            action: #selector(addTaskTapped)
        )
        navigationItem.rightBarButtonItem = addButton
    }
    
    private func loadTasks() {
        tasks = repository.getAll()
        tableView.reloadData()
    }
    
    // MARK: - Actions
    
    @objc private func addTaskTapped() {
        performSegue(withIdentifier: "presentAddTask", sender: nil)
    }
    
    // MARK: - UITableViewDataSource
    
    override func numberOfSections(in tableView: UITableView) -> Int {
        return 1
    }
    
    override func tableView(
        _ tableView: UITableView,
        numberOfRowsInSection section: Int
    ) -> Int {
        return tasks.count
    }
    
    override func tableView(
        _ tableView: UITableView,
        cellForRowAt indexPath: IndexPath
    ) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(
            withIdentifier: "TaskCell",
            for: indexPath
        )
        
        let task = tasks[indexPath.row]
        
        // Configuración de contenido (iOS 14+)
        var content = cell.defaultContentConfiguration()
        content.text = task.title
        content.secondaryText = task.description
        content.image = UIImage(
            systemName: task.isCompleted
                ? "checkmark.circle.fill"
                : "circle"
        )
        content.imageProperties.tintColor = task.isCompleted
            ? .systemGreen
            : .systemGray
        
        cell.contentConfiguration = content
        cell.accessoryType = .disclosureIndicator
        
        return cell
    }
    
    // MARK: - UITableViewDelegate
    
    override func tableView(
        _ tableView: UITableView,
        didSelectRowAt indexPath: IndexPath
    ) {
        tableView.deselectRow(at: indexPath, animated: true)
        performSegue(
            withIdentifier: "showTaskDetail",
            sender: tasks[indexPath.row]
        )
    }
    
    override func tableView(
        _ tableView: UITableView,
        trailingSwipeActionsConfigurationForRowAt indexPath: IndexPath
    ) -> UISwipeActionsConfiguration? {
        let deleteAction = UIContextualAction(
            style: .destructive,
            title: "Eliminar"
        ) { [weak self] _, _, completionHandler in
            guard let self = self else { return }
            let task = self.tasks[indexPath.row]
            self.repository.delete(id: task.id)
            self.tasks.remove(at: indexPath.row)
            tableView.deleteRows(
                at: [indexPath],
                with: .automatic
            )
            completionHandler(true)
        }
        deleteAction.image = UIImage(systemName: "trash")
        
        return UISwipeActionsConfiguration(actions: [deleteAction])
    }
    
    // MARK: - Navigation
    
    override func prepare(for segue: UIStoryboardSegue, sender: Any?) {
        if segue.identifier == "showTaskDetail",
           let detailVC = segue.destination as? TaskDetailViewController,
           let task = sender as? Task {
            detailVC.task = task
            detailVC.repository = repository
        } else if segue.identifier == "presentAddTask",
                  let navController = segue.destination as? UINavigationController,
                  let addVC = navController.topViewController as? AddTaskViewController {
            addVC.repository = repository
        }
    }
}
```

4. Compila con **⌘B** para verificar. Habrá errores porque `TaskDetailViewController` y `AddTaskViewController` aún no existen — los crearemos en los siguientes pasos. Esto es esperado.

### Salida Esperada

El archivo `TaskListViewController.swift` se crea dentro de `UIKitViews/`. Los errores de compilación referentes a los controladores faltantes son normales en este punto.

### Verificación

Confirma que la clase hereda de `UITableViewController` y que el Storyboard tiene la clase `TaskListViewController` asignada en el Identity Inspector.

---

## Paso 5 — Crear TaskDetailViewController (UIKit)

**Objetivo:** Implementar la pantalla de detalle que muestra y permite editar el estado de una tarea, conectada mediante un segue tipo **Show**.

### Instrucciones

1. En `Main.storyboard`, abre la **Object Library** (⇧⌘L) y arrastra un nuevo **View Controller** al lienzo, a la derecha del Table View Controller.

2. Mantén presionada la tecla **Control** y arrastra desde la **celda** `TaskCell` del Table View Controller hasta el nuevo View Controller. En el menú emergente, selecciona **Show** bajo "Selection Segue".

3. Selecciona el segue recién creado (la línea entre las escenas). En el **Attributes Inspector**, establece el **Identifier** como `showTaskDetail`.

4. Selecciona el nuevo View Controller. En el **Identity Inspector**, establece la **Class** como `TaskDetailViewController`.

5. Ahora diseña la interfaz del detalle en el Storyboard. Añade los siguientes elementos arrastrándolos desde la Object Library:

   - Un **UILabel** para el título de la tarea (parte superior).
   - Un **UITextView** para la descripción (zona media, no editable).
   - Un **UILabel** con texto "Completada" y un **UISwitch** a su derecha.
   - Un **UILabel** para la fecha de creación (parte inferior).

6. Aplica las siguientes restricciones de Auto Layout seleccionando cada elemento y usando el botón **Add New Constraints** (⌃⇧⌘= o el ícono en la barra inferior del lienzo):

   **UILabel (Título):**
   - Top: 20 pts al Safe Area
   - Leading: 16 pts
   - Trailing: 16 pts
   - Font: System Bold 22

   **UITextView (Descripción):**
   - Top: 16 pts debajo del título
   - Leading: 12 pts
   - Trailing: 12 pts
   - Height: ≥ 80 pts
   - Desmarca "Editable" en Attributes Inspector

   **UILabel ("Completada"):**
   - Top: 20 pts debajo del text view
   - Leading: 16 pts

   **UISwitch:**
   - CenterY alineado al label "Completada"
   - Trailing: 16 pts

   **UILabel (Fecha):**
   - Top: 20 pts debajo del switch
   - Leading: 16 pts
   - Trailing: 16 pts
   - Font: System 14, Color: Secondary Label

7. Crea el archivo del controlador. Haz clic derecho en `UIKitViews/` → **New File…** → **Cocoa Touch Class**:
   - Class: `TaskDetailViewController`
   - Subclass of: `UIViewController`
   - Haz clic en **Create**.

8. Reemplaza el contenido con:

```swift
import UIKit

class TaskDetailViewController: UIViewController {
    
    // MARK: - Outlets
    
    @IBOutlet weak var titleLabel: UILabel!
    @IBOutlet weak var descriptionTextView: UITextView!
    @IBOutlet weak var completedSwitch: UISwitch!
    @IBOutlet weak var completedLabel: UILabel!
    @IBOutlet weak var dateLabel: UILabel!
    
    // MARK: - Properties
    
    var task: Task?
    var repository: TaskRepository?
    
    private let dateFormatter: DateFormatter = {
        let formatter = DateFormatter()
        formatter.dateStyle = .long
        formatter.timeStyle = .short
        formatter.locale = Locale(identifier: "es_MX")
        return formatter
    }()
    
    // MARK: - Lifecycle
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        title = "Detalle"
        configureUI()
    }
    
    // MARK: - Configuration
    
    private func configureUI() {
        guard let task = task else { return }
        
        titleLabel.text = task.title
        descriptionTextView.text = task.description
        completedSwitch.isOn = task.isCompleted
        completedLabel.text = "Completada"
        dateLabel.text = "Creada: \(dateFormatter.string(from: task.createdAt))"
        
        // Estilo del text view
        descriptionTextView.font = UIFont.systemFont(ofSize: 16)
        descriptionTextView.textColor = .label
        descriptionTextView.backgroundColor = .secondarySystemBackground
        descriptionTextView.layer.cornerRadius = 8
        descriptionTextView.isEditable = false
    }
    
    // MARK: - Actions
    
    @IBAction func completedSwitchChanged(_ sender: UISwitch) {
        guard var task = task else { return }
        task.isCompleted = sender.isOn
        self.task = task
        repository?.update(task)
    }
}
```

9. **Conecta los outlets y la acción** en el Storyboard:
   - Abre el **Assistant Editor** (⌃⌥⌘↩) con el Storyboard a la izquierda y `TaskDetailViewController.swift` a la derecha.
   - **Control + arrastra** desde cada elemento visual al outlet correspondiente en el código.
   - **Control + arrastra** desde el UISwitch al método `completedSwitchChanged(_:)` seleccionando el evento **Value Changed**.

10. Guarda todo con **⌘S**.

### Salida Esperada

El Storyboard muestra el segue `showTaskDetail` conectando la celda del Table View Controller con el Detail View Controller. Todos los outlets tienen el círculo relleno en el gutter del editor.

### Verificación

Compila con **⌘B**. Verifica en el **Connections Inspector** (⌥⌘6) del View Controller que los 5 outlets y la 1 acción estén conectados correctamente.

---

## Paso 6 — Crear AddTaskViewController (UIKit — Presentación Modal)

**Objetivo:** Implementar un formulario de creación de tareas presentado modalmente con campos para título, descripción y estado.

### Instrucciones

1. En `Main.storyboard`, arrastra un nuevo **View Controller** al lienzo.

2. Selecciónalo y ve a **Editor → Embed In → Navigation Controller**. Esto crea un Navigation Controller intermedio que permitirá mostrar botones Cancel/Save en la barra de navegación.

3. Crea el segue: mantén **Control** y arrastra desde el **icono del View Controller** en el Scene Dock del `TaskListViewController` (no desde la celda) hasta el **nuevo Navigation Controller**. Selecciona **Present Modally**.

4. Selecciona el segue y en el Attributes Inspector establece el **Identifier** como `presentAddTask`.

5. En el View Controller destino (el que está dentro del nuevo Navigation Controller), establece la Class como `AddTaskViewController` en el Identity Inspector.

6. Diseña el formulario añadiendo estos elementos con Auto Layout:

   **UITextField (Título):**
   - Top: 20 pts al Safe Area
   - Leading: 16 pts, Trailing: 16 pts
   - Height: 44 pts
   - Placeholder: "Título de la tarea"
   - Border Style: Rounded Rect

   **UITextView (Descripción):**
   - Top: 16 pts debajo del text field
   - Leading: 16 pts, Trailing: 16 pts
   - Height: 120 pts
   - Font: System 16, Text: "" (vacío)

   **UILabel ("Marcar como completada"):**
   - Top: 20 pts debajo del text view
   - Leading: 16 pts

   **UISwitch:**
   - CenterY alineado al label
   - Trailing: 16 pts
   - Estado inicial: Off

7. Crea el archivo. Clic derecho en `UIKitViews/` → **New File…** → **Cocoa Touch Class**:
   - Class: `AddTaskViewController`
   - Subclass of: `UIViewController`
   - **Create**.

8. Reemplaza el contenido:

```swift
import UIKit

class AddTaskViewController: UIViewController {
    
    // MARK: - Outlets
    
    @IBOutlet weak var titleTextField: UITextField!
    @IBOutlet weak var descriptionTextView: UITextView!
    @IBOutlet weak var completedSwitch: UISwitch!
    
    // MARK: - Properties
    
    var repository: TaskRepository?
    
    // MARK: - Lifecycle
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        title = "Nueva Tarea"
        configureNavigationBar()
        configureUI()
    }
    
    // MARK: - Configuration
    
    private func configureNavigationBar() {
        navigationItem.leftBarButtonItem = UIBarButtonItem(
            barButtonSystemItem: .cancel,
            target: self,
            action: #selector(cancelTapped)
        )
        
        navigationItem.rightBarButtonItem = UIBarButtonItem(
            barButtonSystemItem: .save,
            target: self,
            action: #selector(saveTapped)
        )
    }
    
    private func configureUI() {
        // Estilo del text view para que parezca un campo de texto
        descriptionTextView.layer.borderColor = UIColor.systemGray4.cgColor
        descriptionTextView.layer.borderWidth = 1.0
        descriptionTextView.layer.cornerRadius = 8
        descriptionTextView.font = UIFont.systemFont(ofSize: 16)
        
        // Hacer que el teclado se cierre al tocar fuera
        let tapGesture = UITapGestureRecognizer(
            target: view,
            action: #selector(UIView.endEditing)
        )
        view.addGestureRecognizer(tapGesture)
    }
    
    // MARK: - Actions
    
    @objc private func cancelTapped() {
        dismiss(animated: true)
    }
    
    @objc private func saveTapped() {
        guard let title = titleTextField.text,
              !title.trimmingCharacters(in: .whitespaces).isEmpty else {
            showAlert(
                title: "Campo requerido",
                message: "El título de la tarea no puede estar vacío."
            )
            return
        }
        
        let newTask = Task(
            id: UUID(),
            title: title.trimmingCharacters(in: .whitespaces),
            description: descriptionTextView.text ?? "",
            isCompleted: completedSwitch.isOn,
            createdAt: Date()
        )
        
        repository?.add(newTask)
        dismiss(animated: true)
    }
    
    private func showAlert(title: String, message: String) {
        let alert = UIAlertController(
            title: title,
            message: message,
            preferredStyle: .alert
        )
        alert.addAction(UIAlertAction(title: "OK", style: .default))
        present(alert, animated: true)
    }
}
```

9. Conecta los outlets en el Storyboard usando el Assistant Editor:
   - `titleTextField` → UITextField
   - `descriptionTextView` → UITextView
   - `completedSwitch` → UISwitch

10. Guarda con **⌘S** y compila con **⌘B**.

### Salida Esperada

**Build Succeeded.** El Storyboard muestra tres escenas conectadas:
- Navigation Controller (inicial) → TaskListViewController
- TaskListViewController → (Show) → TaskDetailViewController
- TaskListViewController → (Present Modally) → Navigation Controller → AddTaskViewController

### Verificación

Ejecuta la app en el simulador **iPhone 15 Pro** (⌘R):
1. Debes ver la lista "Mis Tareas" con 3 tareas de ejemplo.
2. Toca una celda → navega al detalle con título, descripción, switch y fecha.
3. Toca el botón **+** → se presenta modalmente el formulario de nueva tarea.
4. Escribe un título, toca **Save** → el modal se cierra y la nueva tarea aparece en la lista.
5. Desliza una celda a la izquierda → aparece la opción "Eliminar".

---

## Paso 7 — Commit del Flujo UIKit

**Objetivo:** Registrar todo el trabajo de UIKit en un commit atómico.

### Instrucciones

1. En Terminal:

```bash
cd ~/Developer/TaskManagerApp
git add -A
git status
```

2. Verifica que los archivos añadidos incluyan:
   - `UIKitViews/TaskListViewController.swift`
   - `UIKitViews/TaskDetailViewController.swift`
   - `UIKitViews/AddTaskViewController.swift`
   - Cambios en `Main.storyboard`
   - `InMemoryTaskRepository.swift` (si fue creado en este lab)

3. Haz el commit:

```bash
git commit -m "feat: implement UIKit task list, detail and add views with navigation"
```

### Salida Esperada

```
[feature/ui-module xxxxxxx] feat: implement UIKit task list, detail and add views with navigation
 X files changed, Y insertions(+)
```

### Verificación

```bash
git log --oneline -3
```

El commit más reciente debe mostrar el mensaje del commit.

---

## Paso 8 — Crear la Estructura de Carpetas para SwiftUI

**Objetivo:** Organizar los archivos de la interfaz SwiftUI en un grupo dedicado.

### Instrucciones

1. En Xcode, haz clic derecho en el grupo raíz `TaskManagerApp` → **New Group** → nómbralo `SwiftUIViews`.

2. Dentro de `SwiftUIViews/`, crearemos cuatro archivos en los siguientes pasos:
   - `TaskListView.swift`
   - `TaskRowView.swift`
   - `TaskDetailView.swift`
   - `AddTaskView.swift`

### Salida Esperada

El Project Navigator muestra el grupo `SwiftUIViews/` vacío, listo para recibir los archivos.

### Verificación

El grupo aparece en el Project Navigator bajo `TaskManagerApp`.

---

## Paso 9 — Crear TaskRowView (SwiftUI)

**Objetivo:** Construir una vista reutilizable que represente una fila de tarea con icono SF Symbols, título y descripción.

### Instrucciones

1. Clic derecho en `SwiftUIViews/` → **New File…** → **SwiftUI View** → nombre: `TaskRowView`.

2. Reemplaza el contenido completo:

```swift
import SwiftUI

struct TaskRowView: View {
    
    let task: Task
    
    var body: some View {
        HStack(spacing: 12) {
            Image(systemName: task.isCompleted
                  ? "checkmark.circle.fill"
                  : "circle")
                .foregroundColor(task.isCompleted ? .green : .gray)
                .font(.title2)
            
            VStack(alignment: .leading, spacing: 4) {
                Text(task.title)
                    .font(.headline)
                    .strikethrough(task.isCompleted, color: .gray)
                
                Text(task.description)
                    .font(.subheadline)
                    .foregroundColor(.secondary)
                    .lineLimit(2)
            }
            
            Spacer()
        }
        .padding(.vertical, 4)
    }
}

#Preview {
    TaskRowView(task: Task(
        id: UUID(),
        title: "Tarea de ejemplo",
        description: "Descripción de prueba para la vista previa",
        isCompleted: false,
        createdAt: Date()
    ))
    .padding()
}
```

3. Guarda con **⌘S**. Abre el **Canvas** (⌥⌘↩) para ver la vista previa.

### Salida Esperada

El Canvas muestra una fila con un icono de círculo gris, el título "Tarea de ejemplo" y la descripción debajo.

### Verificación

Cambia `isCompleted` a `true` en el `#Preview` y verifica que el icono cambie a un checkmark verde y el título muestre tachado.

---

## Paso 10 — Crear TaskListView (SwiftUI)

**Objetivo:** Implementar la pantalla principal de lista con `NavigationStack` y `List`, usando el repositorio compartido.

### Instrucciones

1. Clic derecho en `SwiftUIViews/` → **New File…** → **SwiftUI View** → nombre: `TaskListView`.

2. Reemplaza el contenido:

```swift
import SwiftUI

struct TaskListView: View {
    
    // MARK: - Properties
    
    let repository: TaskRepository
    
    @State private var tasks: [Task] = []
    @State private var showingAddTask = false
    
    // MARK: - Body
    
    var body: some View {
        NavigationStack {
            List {
                ForEach(tasks) { task in
                    NavigationLink(value: task.id) {
                        TaskRowView(task: task)
                    }
                }
                .onDelete(perform: deleteTask)
            }
            .navigationTitle("Mis Tareas")
            .navigationDestination(for: UUID.self) { taskId in
                if let task = tasks.first(where: { $0.id == taskId }) {
                    TaskDetailView(
                        task: task,
                        repository: repository,
                        onUpdate: { loadTasks() }
                    )
                }
            }
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    Button {
                        showingAddTask = true
                    } label: {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $showingAddTask) {
                AddTaskView(
                    repository: repository,
                    onSave: { loadTasks() }
                )
            }
            .onAppear {
                loadTasks()
            }
        }
    }
    
    // MARK: - Methods
    
    private func loadTasks() {
        tasks = repository.getAll()
    }
    
    private func deleteTask(at offsets: IndexSet) {
        for index in offsets {
            repository.delete(id: tasks[index].id)
        }
        loadTasks()
    }
}

#Preview {
    TaskListView(repository: InMemoryTaskRepository())
}
```

3. Guarda con **⌘S**.

### Salida Esperada

El Canvas muestra una lista con barra de navegación "Mis Tareas", las 3 tareas de ejemplo y un botón **+** en la esquina superior derecha.

### Verificación

Compila con **⌘B**. Habrá errores porque `TaskDetailView` y `AddTaskView` no existen aún — se crean en los siguientes pasos.

---

## Paso 11 — Crear TaskDetailView (SwiftUI)

**Objetivo:** Construir la vista de detalle con toggle para marcar completada y callback de actualización.

### Instrucciones

1. Clic derecho en `SwiftUIViews/` → **New File…** → **SwiftUI View** → nombre: `TaskDetailView`.

2. Reemplaza el contenido:

```swift
import SwiftUI

struct TaskDetailView: View {
    
    // MARK: - Properties
    
    @State var task: Task
    let repository: TaskRepository
    var onUpdate: () -> Void
    
    private let dateFormatter: DateFormatter = {
        let formatter = DateFormatter()
        formatter.dateStyle = .long
        formatter.timeStyle = .short
        formatter.locale = Locale(identifier: "es_MX")
        return formatter
    }()
    
    // MARK: - Body
    
    var body: some View {
        Form {
            Section("Información") {
                VStack(alignment: .leading, spacing: 8) {
                    Text(task.title)
                        .font(.title2)
                        .fontWeight(.bold)
                    
                    Text(task.description)
                        .font(.body)
                        .foregroundColor(.secondary)
                }
                .padding(.vertical, 4)
            }
            
            Section("Estado") {
                Toggle("Completada", isOn: $task.isCompleted)
                    .onChange(of: task.isCompleted) { _, _ in
                        repository.update(task)
                        onUpdate()
                    }
            }
            
            Section("Fecha de creación") {
                Text(dateFormatter.string(from: task.createdAt))
                    .font(.subheadline)
                    .foregroundColor(.secondary)
            }
        }
        .navigationTitle("Detalle")
        .navigationBarTitleDisplayMode(.inline)
    }
}

#Preview {
    NavigationStack {
        TaskDetailView(
            task: Task(
                id: UUID(),
                title: "Tarea de ejemplo",
                description: "Una descripción detallada de la tarea",
                isCompleted: false,
                createdAt: Date()
            ),
            repository: InMemoryTaskRepository(),
            onUpdate: {}
        )
    }
}
```

3. Guarda con **⌘S**.

### Salida Esperada

El Canvas muestra un formulario con secciones: información (título y descripción), estado (toggle) y fecha de creación.

### Verificación

En el Canvas interactivo (botón ▶ en la esquina inferior izquierda del preview), activa el toggle y verifica que cambia de estado visualmente.

---

## Paso 12 — Crear AddTaskView (SwiftUI con @State y @Binding)

**Objetivo:** Implementar el formulario de creación de tareas usando `@State` para gestión de estado local y `@Environment` para cerrar la vista.

### Instrucciones

1. Clic derecho en `SwiftUIViews/` → **New File…** → **SwiftUI View** → nombre: `AddTaskView`.

2. Reemplaza el contenido:

```swift
import SwiftUI

struct AddTaskView: View {
    
    // MARK: - Environment
    
    @Environment(\.dismiss) private var dismiss
    
    // MARK: - State
    
    @State private var title: String = ""
    @State private var description: String = ""
    @State private var isCompleted: Bool = false
    @State private var showingValidationAlert = false
    
    // MARK: - Properties
    
    let repository: TaskRepository
    var onSave: () -> Void
    
    // MARK: - Body
    
    var body: some View {
        NavigationStack {
            Form {
                Section("Título") {
                    TextField(
                        "Título de la tarea",
                        text: $title
                    )
                    .autocorrectionDisabled()
                }
                
                Section("Descripción") {
                    TextEditor(text: $description)
                        .frame(minHeight: 100)
                }
                
                Section("Estado") {
                    Toggle("Marcar como completada", isOn: $isCompleted)
                }
            }
            .navigationTitle("Nueva Tarea")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .navigationBarLeading) {
                    Button("Cancelar") {
                        dismiss()
                    }
                }
                
                ToolbarItem(placement: .navigationBarTrailing) {
                    Button("Guardar") {
                        saveTask()
                    }
                    .fontWeight(.bold)
                }
            }
            .alert(
                "Campo requerido",
                isPresented: $showingValidationAlert
            ) {
                Button("OK", role: .cancel) {}
            } message: {
                Text("El título de la tarea no puede estar vacío.")
            }
        }
    }
    
    // MARK: - Methods
    
    private func saveTask() {
        let trimmedTitle = title.trimmingCharacters(in: .whitespaces)
        
        guard !trimmedTitle.isEmpty else {
            showingValidationAlert = true
            return
        }
        
        let newTask = Task(
            id: UUID(),
            title: trimmedTitle,
            description: description,
            isCompleted: isCompleted,
            createdAt: Date()
        )
        
        repository.add(newTask)
        onSave()
        dismiss()
    }
}

#Preview {
    AddTaskView(
        repository: InMemoryTaskRepository(),
        onSave: {}
    )
}
```

3. Guarda con **⌘S** y compila con **⌘B**.

### Salida Esperada

**Build Succeeded.** El Canvas muestra el formulario con campos de título, descripción y toggle, con botones "Cancelar" y "Guardar" en la barra de navegación.

### Verificación

Verifica en el Canvas interactivo:
1. Escribe texto en el campo de título.
2. Toca "Guardar" sin título → debe aparecer la alerta de validación.
3. Toca "Cancelar" → la vista se cierra.

---

## Paso 13 — Crear un Punto de Entrada para el Flujo SwiftUI

**Objetivo:** Permitir acceso al flujo SwiftUI desde la app para poder probarlo. Crearemos un `UIHostingController` que se pueda invocar opcionalmente.

### Instrucciones

1. Abre `Main.storyboard`. Arrastra un **Bar Button Item** a la barra de navegación del `TaskListViewController`, en el **lado izquierdo** (left bar button).

2. En el Attributes Inspector del Bar Button Item, establece:
   - System Item: **Bookmarks** (o cualquier ícono distintivo)
   - O bien, cambia el título a "SwiftUI".

3. Abre el Assistant Editor con `TaskListViewController.swift` visible. **Control + arrastra** desde el botón al código para crear una acción:

```swift
@IBAction func showSwiftUITapped(_ sender: UIBarButtonItem) {
    let swiftUIView = TaskListView(repository: repository)
    let hostingController = UIHostingController(rootView: swiftUIView)
    hostingController.title = "SwiftUI"
    navigationController?.pushViewController(hostingController, animated: true)
}
```

4. Añade el import de SwiftUI en la parte superior de `TaskListViewController.swift`:

```swift
import UIKit
import SwiftUI
```

5. Guarda y compila con **⌘B**.

### Salida Esperada

**Build Succeeded.** La barra de navegación del TaskListViewController ahora tiene un botón "SwiftUI" a la izquierda y el botón "+" a la derecha.

### Verificación

Ejecuta la app en el simulador **iPhone 15 Pro** (⌘R):
1. Toca el botón "SwiftUI" → navega a la lista SwiftUI con las mismas tareas.
2. Toca una tarea en la lista SwiftUI → navega al detalle.
3. Toca "+" → se presenta el formulario de nueva tarea.
4. Crea una tarea desde SwiftUI y regresa → la tarea aparece en la lista.

> **Nota:** Como ambas interfaces comparten la misma instancia de `InMemoryTaskRepository`, las tareas creadas en un flujo aparecen en el otro.

---

## Paso 14 — Commit del Flujo SwiftUI

**Objetivo:** Registrar todo el trabajo de SwiftUI en un commit atómico.

### Instrucciones

1. En Terminal:

```bash
cd ~/Developer/TaskManagerApp
git add -A
git status
```

2. Verifica que los archivos incluyan:
   - `SwiftUIViews/TaskListView.swift`
   - `SwiftUIViews/TaskRowView.swift`
   - `SwiftUIViews/TaskDetailView.swift`
   - `SwiftUIViews/AddTaskView.swift`
   - Cambios en `TaskListViewController.swift` y `Main.storyboard`

3. Haz el commit:

```bash
git commit -m "feat: implement SwiftUI task list, detail and add views with state management"
```

### Salida Esperada

```
[feature/ui-module xxxxxxx] feat: implement SwiftUI task list, detail and add views with state management
 X files changed, Y insertions(+)
```

### Verificación

```bash
git log --oneline -5
```

Deben verse al menos 2 commits de este lab (UIKit y SwiftUI).

---

## Paso 15 — Merge de la Rama a `develop`

**Objetivo:** Integrar todo el trabajo de UI en la rama principal de desarrollo.

### Instrucciones

1. Asegúrate de que no hay cambios pendientes:

```bash
git status
```

2. Cambia a `develop` y haz merge:

```bash
git checkout develop
git merge feature/ui-module
```

3. Verifica el historial:

```bash
git log --oneline --graph -10
```

### Salida Esperada

```
*   xxxxxxx (HEAD -> develop, feature/ui-module) feat: implement SwiftUI task list, detail and add views with state management
*   xxxxxxx feat: implement UIKit task list, detail and add views with navigation
*   xxxxxxx (commits previos del Lab 01)
```

### Verificación

```bash
git branch
```

Debes estar en `develop` con todos los cambios integrados. Opcionalmente, sube a GitHub:

```bash
git push origin develop
```

---

## Validación y Pruebas

Ejecuta la aplicación en el simulador **iPhone 15 Pro con iOS 17.5** y verifica cada uno de los siguientes escenarios:

### Flujo UIKit

| # | Escenario | Resultado esperado | ✓ |
|---|---|---|---|
| 1 | Abrir la app | Lista "Mis Tareas" con 3 tareas, títulos grandes | ☐ |
| 2 | Tocar una celda | Navega al detalle con título, descripción, switch y fecha | ☐ |
| 3 | Cambiar el switch en detalle | El estado se actualiza; al regresar, el icono de la celda refleja el cambio | ☐ |
| 4 | Tocar botón "+" | Se presenta modalmente el formulario de nueva tarea | ☐ |
| 5 | Guardar sin título | Aparece alerta "Campo requerido" | ☐ |
| 6 | Guardar con título válido | Modal se cierra, nueva tarea aparece en la lista | ☐ |
| 7 | Deslizar celda a la izquierda | Aparece botón "Eliminar" rojo; al tocar, la celda se elimina con animación | ☐ |

### Flujo SwiftUI

| # | Escenario | Resultado esperado | ✓ |
|---|---|---|---|
| 8 | Tocar botón "SwiftUI" | Navega a la lista SwiftUI con las mismas tareas | ☐ |
| 9 | Tocar una fila | Navega al detalle con Form, toggle y fecha | ☐ |
| 10 | Cambiar toggle en detalle | Estado se actualiza en el repositorio | ☐ |
| 11 | Tocar "+" en SwiftUI | Sheet se presenta con formulario | ☐ |
| 12 | Guardar sin título | Alerta de validación | ☐ |
| 13 | Guardar con datos válidos | Sheet se cierra, tarea nueva en la lista | ☐ |
| 14 | Deslizar fila para eliminar | La tarea se elimina de la lista | ☐ |

### Auto Layout

| # | Escenario | Resultado esperado | ✓ |
|---|---|---|---|
| 15 | Rotar simulador (⌘←/⌘→) | Los elementos se reposicionan correctamente sin recortes | ☐ |
| 16 | Consola de Xcode | Sin warnings de "Unable to simultaneously satisfy constraints" | ☐ |

---

## Solución de Problemas

### Problema 1: La celda no muestra datos — aparece vacía o con texto genérico

**Síntomas:** Al ejecutar la app, las celdas del UITableView aparecen vacías o muestran "Title" / "Subtitle" estáticos.

**Causa:** El Reuse Identifier de la celda en el Storyboard no coincide con el string usado en `dequeueReusableCell(withIdentifier:for:)`, o la clase del View Controller no está asignada en el Identity Inspector.

**Solución:**
1. Abre `Main.storyboard` y selecciona la celda dentro del Table View Controller.
2. En el **Attributes Inspector**, verifica que el **Identifier** sea exactamente `TaskCell` (sensible a mayúsculas).
3. Selecciona el Table View Controller y verifica en el **Identity Inspector** que la Class sea `TaskListViewController`.
4. Limpia el build (⇧⌘K) y ejecuta de nuevo (⌘R).

---

### Problema 2: El segue `presentAddTask` lanza un crash con "has no segue with identifier"

**Síntomas:** Al tocar el botón "+", la app se detiene con la excepción: `NSInvalidArgumentException: Receiver (<TaskManagerApp.TaskListViewController>) has no segue with identifier 'presentAddTask'`.

**Causa:** El segue en el Storyboard no fue creado desde el icono del View Controller en el Scene Dock, o el identifier no fue establecido correctamente.

**Solución:**
1. En `Main.storyboard`, verifica que el segue existe entre el `TaskListViewController` y el Navigation Controller que contiene `AddTaskViewController`.
2. El segue debe originarse desde el **icono amarillo del View Controller** en el Scene Dock (no desde un botón o celda), ya que se invoca programáticamente con `performSegue`.
3. Selecciona el segue (la línea entre las escenas) y en el **Attributes Inspector** confirma que el Identifier sea exactamente `presentAddTask`.
4. Confirma que el tipo sea **Present Modally**.
5. Si el segue no existe, elimina cualquier segue incorrecto, luego **Control + arrastra** desde el icono amarillo del TaskListViewController hasta el Navigation Controller destino → selecciona "Present Modally" → establece el identifier.

---

## Limpieza

No se requiere limpieza especial para este laboratorio. Los archivos creados forman parte del proyecto incremental y se usarán en laboratorios posteriores.

Si deseas eliminar la rama de feature ya integrada:

```bash
git branch -d feature/ui-module
```

---

## Resumen

En este laboratorio has construido **dos flujos de interfaz de usuario completos** para TaskManagerApp:

| Aspecto | UIKit | SwiftUI |
|---|---|---|
| **Lista** | `UITableViewController` con celdas `Subtitle` | `List` con `ForEach` y `TaskRowView` |
| **Navegación** | `UINavigationController` + segue Show | `NavigationStack` + `NavigationLink` |
| **Detalle** | `TaskDetailViewController` con outlets | `TaskDetailView` con `Form` y `@State` |
| **Creación** | `AddTaskViewController` modal con segue | `AddTaskView` con `.sheet` y `@State` |
| **Eliminación** | `trailingSwipeActionsConfiguration` | `.onDelete(perform:)` |
| **Estado** | Propiedades + `reloadData()` | `@State`, `@Binding`, `@Environment` |

Ambos flujos comparten el mismo `InMemoryTaskRepository`, demostrando la separación entre lógica de datos e interfaz. Has aplicado Auto Layout con constraints en Interface Builder, SF Symbols para iconografía, y gestión de estado declarativa en SwiftUI.

### Recursos Adicionales

- [UIKit — Apple Developer Documentation](https://developer.apple.com/documentation/uikit)
- [SwiftUI — Apple Developer Documentation](https://developer.apple.com/documentation/swiftui)
- [Auto Layout Guide — Apple](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/)
- [SF Symbols 5 — Apple](https://developer.apple.com/sf-symbols/)
- [NavigationStack — Apple Developer](https://developer.apple.com/documentation/swiftui/navigationstack)
