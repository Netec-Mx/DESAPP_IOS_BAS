# Práctica Incremental — Capa de Persistencia con UserDefaults, Core Data y SwiftData

## 1. Metadatos

| Campo | Valor |
|---|---|
| **Duración** | 288 minutos (~4 h 48 min) |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |

## 2. Descripción General

En este laboratorio reemplazarás el `InMemoryTaskRepository` construido en el Lab 01 con tres implementaciones de persistencia real: **UserDefaults** para preferencias del usuario, **Core Data** para un repositorio relacional completo y **SwiftData** como alternativa moderna declarativa. Al finalizar, dispondrás de un `enum RepositoryType` que permite conmutar entre las tres implementaciones, y habrás seleccionado SwiftData como repositorio principal de producción. Todo el trabajo se realizará en la rama `feature/persistence` y se integrará a `develop`.

## 3. Objetivos de Aprendizaje

- [ ] Implementar un servicio `UserPreferencesService` que persista preferencias de usuario (tema de color, ordenamiento, filtro de tareas completadas) usando `UserDefaults`.
- [ ] Construir un stack de Core Data completo con entidad `TaskEntity`, `NSPersistentContainer` y un `CoreDataTaskRepository` que conforme el protocolo `TaskRepository`.
- [ ] Crear un modelo `@Model SDTaskItem` con SwiftData e implementar `SwiftDataTaskRepository` usando `ModelContainer` y `ModelContext`.
- [ ] Comparar las tres tecnologías de persistencia y configurar un mecanismo de selección (`RepositoryType`) que permita cambiar la implementación activa.
- [ ] Gestionar el flujo Git completo: rama `feature/persistence`, commits con Conventional Commits y merge a `develop`.

## 4. Prerrequisitos

### Conocimientos

| Requisito | Detalle |
|---|---|
| Lab 02-00-01 completado | Interfaces UIKit y SwiftUI funcionales con `InMemoryTaskRepository` |
| Protocolo `TaskRepository` | Comprender la interfaz CRUD definida en labs anteriores |
| Swift básico | Closures, protocolos, enums, extensiones, `Codable` |
| Git básico | Ramas, commits, merge |

### Acceso

| Recurso | Detalle |
|---|---|
| Repositorio GitHub | `task-manager-ios` (privado), rama `develop` actualizada |
| Xcode 15.4 | Con simulador iPhone 15 Pro / iOS 17.5 instalado |
| Deployment Target | iOS 17.0 (configurado en Lab 01) |

## 5. Entorno del Laboratorio

### Hardware

| Componente | Mínimo |
|---|---|
| Procesador | Apple Silicon (M1/M2/M3) o Intel i5 8ª gen+ |
| RAM | 16 GB |
| Disco libre | 50 GB |
| Pantalla | 1440 × 900 (recomendado 1920 × 1080) |

### Software

| Herramienta | Versión |
|---|---|
| macOS | Ventura 13.6.7 o Sonoma 14.5 |
| Xcode | 15.4 |
| Swift | 5.10 |
| Simulador | iPhone 15 Pro — iOS 17.5 |
| Git | 2.45.2 |
| XCTest | 15.4 (incluido con Xcode) |

### Preparación inicial

Abre Terminal y ejecuta:

```bash
cd ~/Developer/TaskManagerApp
git checkout develop
git pull origin develop
git checkout -b feature/persistence
```

Verifica que el proyecto compila sin errores:

```bash
xcodebuild -scheme TaskManagerApp -destination 'platform=iOS Simulator,name=iPhone 15 Pro,OS=17.5' build 2>&1 | tail -5
```

**Salida esperada:** la última línea debe contener `** BUILD SUCCEEDED **`.

---

## 6. Pasos del Laboratorio

---

### PARTE 1 — UserDefaults: Preferencias del Usuario (45 min)

---

#### Paso 1: Crear el enum `SortOrder` y el modelo de preferencias

**Objetivo:** Definir los tipos de datos que representan las preferencias del usuario de forma segura y codificable.

**Instrucciones:**

1. En Xcode, haz clic derecho sobre el grupo `Models` en el navegador de archivos y selecciona **New File… → Swift File**. Nómbralo `UserPreferences.swift`.

2. Escribe el siguiente código:

```swift
// UserPreferences.swift
import Foundation

/// Orden de clasificación de la lista de tareas.
enum SortOrder: String, Codable, CaseIterable {
    case createdAtAscending  = "createdAtAsc"
    case createdAtDescending = "createdAtDesc"
    case titleAscending      = "titleAsc"
    case titleDescending     = "titleDesc"

    var displayName: String {
        switch self {
        case .createdAtAscending:  return "Fecha ↑"
        case .createdAtDescending: return "Fecha ↓"
        case .titleAscending:      return "Título A-Z"
        case .titleDescending:     return "Título Z-A"
        }
    }
}

/// Temas de color disponibles en la aplicación.
enum ColorTheme: String, Codable, CaseIterable {
    case system  = "system"
    case light   = "light"
    case dark    = "dark"
    case blue    = "blue"

    var displayName: String {
        switch self {
        case .system: return "Sistema"
        case .light:  return "Claro"
        case .dark:   return "Oscuro"
        case .blue:   return "Azul"
        }
    }
}
```

3. Guarda el archivo con **⌘ S**.

**Salida esperada:** El archivo aparece bajo el grupo `Models` sin errores de compilación.

**Verificación:** Compila con **⌘ B**. No deben aparecer errores ni advertencias.

---

#### Paso 2: Implementar `UserPreferencesService`

**Objetivo:** Crear un servicio que encapsule toda la interacción con `UserDefaults` usando claves tipadas y propiedades computadas.

**Instrucciones:**

1. Crea un nuevo grupo llamado `Services` en el navegador de Xcode (clic derecho sobre `TaskManagerApp` → **New Group**).

2. Dentro de `Services`, crea un archivo Swift llamado `UserPreferencesService.swift`.

3. Implementa el servicio:

```swift
// UserPreferencesService.swift
import Foundation

final class UserPreferencesService {

    // MARK: - Singleton
    static let shared = UserPreferencesService()

    private let defaults: UserDefaults

    // Inyección para testing; producción usa .standard
    init(defaults: UserDefaults = .standard) {
        self.defaults = defaults
        registerDefaults()
    }

    // MARK: - Claves privadas
    private enum Keys {
        static let sortOrder          = "userPref_sortOrder"
        static let colorTheme         = "userPref_colorTheme"
        static let showCompletedTasks = "userPref_showCompletedTasks"
    }

    // MARK: - Valores por defecto
    private func registerDefaults() {
        defaults.register(defaults: [
            Keys.sortOrder:          SortOrder.createdAtDescending.rawValue,
            Keys.colorTheme:         ColorTheme.system.rawValue,
            Keys.showCompletedTasks: true
        ])
    }

    // MARK: - Sort Order
    var sortOrder: SortOrder {
        get {
            guard let raw = defaults.string(forKey: Keys.sortOrder),
                  let value = SortOrder(rawValue: raw) else {
                return .createdAtDescending
            }
            return value
        }
        set {
            defaults.set(newValue.rawValue, forKey: Keys.sortOrder)
        }
    }

    // MARK: - Color Theme
    var colorTheme: ColorTheme {
        get {
            guard let raw = defaults.string(forKey: Keys.colorTheme),
                  let value = ColorTheme(rawValue: raw) else {
                return .system
            }
            return value
        }
        set {
            defaults.set(newValue.rawValue, forKey: Keys.colorTheme)
        }
    }

    // MARK: - Show Completed Tasks
    var showCompletedTasks: Bool {
        get { defaults.bool(forKey: Keys.showCompletedTasks) }
        set { defaults.set(newValue, forKey: Keys.showCompletedTasks) }
    }

    // MARK: - Reset
    func resetToDefaults() {
        defaults.removeObject(forKey: Keys.sortOrder)
        defaults.removeObject(forKey: Keys.colorTheme)
        defaults.removeObject(forKey: Keys.showCompletedTasks)
        registerDefaults()
    }
}
```

4. Guarda con **⌘ S**.

**Salida esperada:** El archivo compila sin errores.

**Verificación:** Compila con **⌘ B**. Verifica que `UserPreferencesService.shared.sortOrder` devuelve `.createdAtDescending` por defecto añadiendo temporalmente un `print` en `application(_:didFinishLaunchingWithOptions:)` de `AppDelegate` (elimínalo después de verificar).

---

#### Paso 3: Crear la vista de preferencias en SwiftUI

**Objetivo:** Construir un `PreferencesView` que lea y escriba preferencias a través de `UserPreferencesService` y de `@AppStorage`.

**Instrucciones:**

1. Dentro del grupo `Views` (o `SwiftUIViews` si así lo nombraste en Lab 02), crea un archivo `PreferencesView.swift`.

2. Implementa la vista:

```swift
// PreferencesView.swift
import SwiftUI

struct PreferencesView: View {

    // @AppStorage sincroniza directamente con UserDefaults
    @AppStorage("userPref_colorTheme")
    private var colorThemeRaw: String = ColorTheme.system.rawValue

    @AppStorage("userPref_showCompletedTasks")
    private var showCompletedTasks: Bool = true

    @AppStorage("userPref_sortOrder")
    private var sortOrderRaw: String = SortOrder.createdAtDescending.rawValue

    // Propiedades computadas para trabajar con enums
    private var colorTheme: ColorTheme {
        get { ColorTheme(rawValue: colorThemeRaw) ?? .system }
    }

    private var sortOrder: SortOrder {
        get { SortOrder(rawValue: sortOrderRaw) ?? .createdAtDescending }
    }

    var body: some View {
        NavigationStack {
            Form {
                // MARK: - Apariencia
                Section("Apariencia") {
                    Picker("Tema de color", selection: $colorThemeRaw) {
                        ForEach(ColorTheme.allCases, id: \.rawValue) { theme in
                            Text(theme.displayName).tag(theme.rawValue)
                        }
                    }
                    .pickerStyle(.menu)
                }

                // MARK: - Lista de tareas
                Section("Lista de tareas") {
                    Toggle("Mostrar tareas completadas", isOn: $showCompletedTasks)

                    Picker("Ordenar por", selection: $sortOrderRaw) {
                        ForEach(SortOrder.allCases, id: \.rawValue) { order in
                            Text(order.displayName).tag(order.rawValue)
                        }
                    }
                    .pickerStyle(.menu)
                }

                // MARK: - Acciones
                Section {
                    Button("Restablecer preferencias", role: .destructive) {
                        UserPreferencesService.shared.resetToDefaults()
                        // Sincronizar @AppStorage manualmente tras reset
                        colorThemeRaw = ColorTheme.system.rawValue
                        sortOrderRaw = SortOrder.createdAtDescending.rawValue
                        showCompletedTasks = true
                    }
                }
            }
            .navigationTitle("Preferencias")
        }
    }
}

#Preview {
    PreferencesView()
}
```

3. Guarda con **⌘ S**.

4. Abre la vista en el Canvas de Xcode (**⌘ ⌥ ↩**) o ejecuta en el simulador para verificar que los controles aparecen correctamente.

**Salida esperada:** Un formulario con tres secciones: Apariencia (picker de tema), Lista de tareas (toggle + picker de orden) y un botón de restablecer.

**Verificación:**
- Cambia el tema a "Oscuro", cierra la app en el simulador (desliza hacia arriba desde Home), vuelve a abrirla. El picker debe mostrar "Oscuro".
- Pulsa "Restablecer preferencias". Todos los controles deben volver a sus valores por defecto.

---

#### Paso 4: Integrar `PreferencesView` en la navegación existente

**Objetivo:** Hacer accesible la pantalla de preferencias desde la interfaz principal de la app.

**Instrucciones:**

1. En la vista SwiftUI `TaskListView` del Lab 02, añade un botón de navegación a preferencias dentro del `.toolbar` existente (junto al botón **+**):

```swift
.toolbar {
    // ... ToolbarItem del botón "+" del Lab 02 ...
    ToolbarItem(placement: .navigationBarLeading) {
        NavigationLink(destination: PreferencesView()) {
            Image(systemName: "gearshape")
                .accessibilityLabel("Preferencias")
        }
    }
}
```

2. En la interfaz UIKit, abre `UIKitViews/TaskListViewController.swift` (Lab 02). Agrega `import SwiftUI` si no lo tiene y reemplaza `configureNavigationBar()` para que la barra muestre el botón **+** y el de preferencias (si asignaras solo `rightBarButtonItem`, el engranaje reemplazaría al **+**):

```swift
private func configureNavigationBar() {
    let addButton = UIBarButtonItem(
        barButtonSystemItem: .add,
        target: self,
        action: #selector(addTaskTapped)
    )
    let gearButton = UIBarButtonItem(
        image: UIImage(systemName: "gearshape"),
        style: .plain,
        target: self,
        action: #selector(openPreferences)
    )
    navigationItem.rightBarButtonItems = [addButton, gearButton]
}

@objc private func openPreferences() {
    let preferencesVC = UIHostingController(rootView: PreferencesView())
    navigationController?.pushViewController(preferencesVC, animated: true)
}
```

3. Compila y ejecuta (**⌘ R**). Navega a Preferencias desde ambas interfaces (UIKit y SwiftUI).

**Salida esperada:** El icono de engranaje aparece en la barra de navegación y al pulsarlo se muestra `PreferencesView`.

**Verificación:** Modifica una preferencia, regresa a la lista de tareas, vuelve a Preferencias. El valor debe haberse conservado.

---

#### Paso 5: Commit de la Parte 1

**Objetivo:** Registrar el progreso de UserDefaults en Git.

**Instrucciones:**

```bash
cd ~/Developer/TaskManagerApp
git add -A
git commit -m "feat: add UserPreferencesService with UserDefaults persistence"
```

**Salida esperada:** Commit creado en la rama `feature/persistence`.

---

### PARTE 2 — Core Data: Repositorio Persistente (100 min)

---

#### Paso 6: Crear el modelo de datos Core Data

**Objetivo:** Definir la entidad `TaskEntity` en el editor de modelos de Xcode.

**Instrucciones:**

1. En Xcode, selecciona **File → New → File… → Data Model** (bajo la categoría Core Data). Nómbralo `TaskManagerApp` (se creará `TaskManagerApp.xcdatamodeld`).

2. En el editor de modelos, haz clic en **Add Entity** (botón inferior). Renombra la entidad a `TaskEntity`.

3. Añade los siguientes atributos haciendo clic en el botón **+** de la sección Attributes:

| Atributo | Tipo |
|---|---|
| `id` | UUID |
| `title` | String |
| `taskDescription` | String |
| `isCompleted` | Boolean |
| `createdAt` | Date |

4. Selecciona la entidad `TaskEntity` y en el **Data Model Inspector** (panel derecho):
   - **Codegen:** selecciona **Manual/None** (escribiremos la clase manualmente para tener control total).
   - **Module:** selecciona **Current Product Module**.

5. Guarda con **⌘ S**.

**Salida esperada:** El archivo `.xcdatamodeld` aparece en el navegador con la entidad `TaskEntity` y sus 5 atributos.

**Verificación:** Al seleccionar cada atributo, el tipo mostrado en el inspector coincide con la tabla anterior.

---

#### Paso 7: Crear la clase `TaskEntity` manualmente

**Objetivo:** Generar la subclase de `NSManagedObject` con propiedades tipadas.

**Instrucciones:**

1. Crea un nuevo grupo llamado `CoreData` en el navegador de Xcode.

2. Dentro de `CoreData`, crea un archivo `TaskEntity+CoreDataClass.swift`:

```swift
// TaskEntity+CoreDataClass.swift
import Foundation
import CoreData

@objc(TaskEntity)
public class TaskEntity: NSManagedObject {
}
```

3. Crea otro archivo `TaskEntity+CoreDataProperties.swift`:

```swift
// TaskEntity+CoreDataProperties.swift
import Foundation
import CoreData

extension TaskEntity {

    @nonobjc public class func fetchRequest() -> NSFetchRequest<TaskEntity> {
        return NSFetchRequest<TaskEntity>(entityName: "TaskEntity")
    }

    @NSManaged public var id: UUID?
    @NSManaged public var title: String?
    @NSManaged public var taskDescription: String?
    @NSManaged public var isCompleted: Bool
    @NSManaged public var createdAt: Date?
}

extension TaskEntity: Identifiable { }
```

4. Guarda ambos archivos con **⌘ S**.

**Salida esperada:** Ambos archivos compilan sin errores.

**Verificación:** Compila con **⌘ B**. Si aparece un error de "redeclaration", verifica que **Codegen** en el modelo está en **Manual/None**.

---

#### Paso 8: Implementar `CoreDataStack`

**Objetivo:** Crear un singleton que gestione el `NSPersistentContainer` y proporcione acceso al contexto de Core Data.

**Instrucciones:**

1. Dentro del grupo `CoreData`, crea `CoreDataStack.swift`:

```swift
// CoreDataStack.swift
import Foundation
import CoreData

final class CoreDataStack {

    // MARK: - Singleton
    static let shared = CoreDataStack()

    // MARK: - Persistent Container
    let persistentContainer: NSPersistentContainer

    // Contexto principal (main queue)
    var viewContext: NSManagedObjectContext {
        persistentContainer.viewContext
    }

    // MARK: - Init
    private init(inMemory: Bool = false) {
        persistentContainer = NSPersistentContainer(name: "TaskManagerApp")

        if inMemory {
            let description = NSPersistentStoreDescription()
            description.type = NSInMemoryStoreType
            persistentContainer.persistentStoreDescriptions = [description]
        }

        persistentContainer.loadPersistentStores { description, error in
            if let error = error as NSError? {
                fatalError("Error al cargar Core Data: \(error), \(error.userInfo)")
            }
        }

        // Merge automático de cambios desde background contexts
        persistentContainer.viewContext.automaticallyMergesChangesFromParent = true
        persistentContainer.viewContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
    }

    /// Inicializador para testing con store en memoria.
    static func forTesting() -> CoreDataStack {
        let stack = CoreDataStack(inMemory: true)
        return stack
    }

    // MARK: - Save
    func saveContext() {
        let context = viewContext
        if context.hasChanges {
            do {
                try context.save()
            } catch {
                let nsError = error as NSError
                print("Error al guardar Core Data: \(nsError), \(nsError.userInfo)")
            }
        }
    }
}
```

> **Nota:** El `private init` con parámetro `inMemory` permite crear un stack de pruebas sin tocar disco. El singleton `shared` usa disco por defecto.

2. Guarda con **⌘ S**.

**Salida esperada:** El archivo compila sin errores.

**Verificación:** Compila con **⌘ B**.

---

#### Paso 9: Implementar `CoreDataTaskRepository`

**Objetivo:** Crear un repositorio que conforme `TaskRepository` usando operaciones CRUD de Core Data.

**Instrucciones:**

1. Verifica que tu protocolo `TaskRepository` es el que dejaste en el Lab 02 (sin `mutating`). Los tres repositorios de este lab (Core Data, SwiftData y el de memoria) implementan exactamente estos métodos:

```swift
// Services/TaskRepository.swift (versión del Lab 02)
protocol TaskRepository {
    func fetchAll() -> [TaskItem]
    func fetchById(_ id: UUID) -> TaskItem?
    func add(_ task: TaskItem)
    @discardableResult func update(_ task: TaskItem) -> Bool
    @discardableResult func delete(byId id: UUID) -> Bool
    func filter(where predicate: (TaskItem) -> Bool) -> [TaskItem]
}
```

> **Importante:** Desde el Lab 01 el modelo de dominio se llama `TaskItem` (y no `Task`) para evitar conflictos con `Swift.Task` (concurrencia). No modifiques `TaskItem.swift`: este lab usa el struct del Lab 01 tal como está (`taskDescription` opcional, `priority`, `dueDate`, `createdAt`).

2. El inicializador del Lab 01 genera un `id` y un `createdAt` nuevos cada vez. Para reconstruir una tarea leída de la base de datos hay que conservar los valores guardados, así que agrega un inicializador completo en una **extensión** (así el inicializador original sigue disponible). Crea `Models/TaskItem+Persistence.swift`:

```swift
// TaskItem+Persistence.swift
import Foundation

extension TaskItem {
    /// Reconstruye una tarea con todos sus valores guardados.
    /// Lo usan los repositorios de Core Data y SwiftData.
    init(
        id: UUID,
        title: String,
        taskDescription: String?,
        isCompleted: Bool,
        priority: Priority,
        createdAt: Date,
        dueDate: Date?
    ) {
        self.id = id
        self.title = title
        self.taskDescription = taskDescription
        self.isCompleted = isCompleted
        self.priority = priority
        self.createdAt = createdAt
        self.dueDate = dueDate
    }
}
```

> **Nota:** Un inicializador declarado en una extensión de un struct puede asignar las propiedades `let` (`id`, `createdAt`) y no reemplaza al inicializador del Lab 01.

3. Dentro del grupo `CoreData`, crea `CoreDataTaskRepository.swift`:

```swift
// CoreDataTaskRepository.swift
import Foundation
import CoreData

final class CoreDataTaskRepository: TaskRepository {

    private let coreDataStack: CoreDataStack

    init(coreDataStack: CoreDataStack = .shared) {
        self.coreDataStack = coreDataStack
    }

    private var context: NSManagedObjectContext {
        coreDataStack.viewContext
    }

    // MARK: - Fetch All
    func fetchAll() -> [TaskItem] {
        let request: NSFetchRequest<TaskEntity> = TaskEntity.fetchRequest()
        request.sortDescriptors = [
            NSSortDescriptor(key: "createdAt", ascending: false)
        ]

        do {
            let entities = try context.fetch(request)
            return entities.map { entity in
                TaskItem(
                    id: entity.id ?? UUID(),
                    title: entity.title ?? "",
                    taskDescription: entity.taskDescription,
                    isCompleted: entity.isCompleted,
                    priority: .medium,      // la entidad TaskEntity no guarda prioridad
                    createdAt: entity.createdAt ?? Date(),
                    dueDate: nil            // ni fecha límite (ver nota al final del paso)
                )
            }
        } catch {
            print("Error al obtener tareas de Core Data: \(error)")
            return []
        }
    }

    // MARK: - Add
    func add(_ task: TaskItem) {
        let entity = TaskEntity(context: context)
        entity.id = task.id
        entity.title = task.title
        entity.taskDescription = task.taskDescription
        entity.isCompleted = task.isCompleted
        entity.createdAt = task.createdAt
        coreDataStack.saveContext()
    }

    // MARK: - Fetch by ID
    func fetchById(_ id: UUID) -> TaskItem? {
        fetchAll().first { $0.id == id }
    }

    // MARK: - Update
    @discardableResult
    func update(_ task: TaskItem) -> Bool {
        let request: NSFetchRequest<TaskEntity> = TaskEntity.fetchRequest()
        request.predicate = NSPredicate(format: "id == %@", task.id as CVarArg)

        do {
            guard let entity = try context.fetch(request).first else { return false }
            entity.title = task.title
            entity.taskDescription = task.taskDescription
            entity.isCompleted = task.isCompleted
            coreDataStack.saveContext()
            return true
        } catch {
            print("Error al actualizar tarea en Core Data: \(error)")
            return false
        }
    }

    // MARK: - Delete
    @discardableResult
    func delete(byId id: UUID) -> Bool {
        let request: NSFetchRequest<TaskEntity> = TaskEntity.fetchRequest()
        request.predicate = NSPredicate(format: "id == %@", id as CVarArg)

        do {
            guard let entity = try context.fetch(request).first else { return false }
            context.delete(entity)
            coreDataStack.saveContext()
            return true
        } catch {
            print("Error al eliminar tarea de Core Data: \(error)")
            return false
        }
    }

    // MARK: - Filter
    func filter(where predicate: (TaskItem) -> Bool) -> [TaskItem] {
        fetchAll().filter(predicate)
    }
}
```

> **Nota:** La entidad `TaskEntity` de este lab guarda solo los campos básicos; al leer, `priority` toma `.medium` y `dueDate` queda en `nil`. SwiftData (Parte 3) sí guarda todos los campos y será el repositorio de producción. Como ejercicio, puedes agregar a la entidad los atributos `dueDate` (Date, opcional) y `priorityRaw` (Integer 16).

4. Guarda con **⌘ S** y compila con **⌘ B**.

**Salida esperada:** Compilación exitosa sin errores.

**Verificación:** Revisa que `CoreDataTaskRepository` aparece en el autocompletado al escribir `let repo: TaskRepository = CoreData...`.

---

#### Paso 10: Probar Core Data en el simulador

**Objetivo:** Verificar que los datos persisten entre relanzamientos de la app.

**Instrucciones:**

1. Temporalmente, en `UIKitViews/TaskListViewController.swift` (Lab 02), reemplaza el repositorio en uso por `CoreDataTaskRepository()`:

```swift
// Antes:
// private let repository: TaskRepository = InMemoryTaskRepository(seedSampleData: true)
// Después:
private let repository: TaskRepository = CoreDataTaskRepository()
```

2. Ejecuta la app en el simulador (**⌘ R**).

3. Añade dos tareas de prueba desde la interfaz:
   - "Comprar leche"
   - "Estudiar Core Data"

4. Detén la app en Xcode (**⌘ .** o botón Stop).

5. Vuelve a ejecutar (**⌘ R**) **sin** limpiar datos.

6. Verifica que las dos tareas aparecen en la lista.

7. Marca "Comprar leche" como completada. Detén y relanza. La tarea debe seguir marcada.

8. Elimina "Estudiar Core Data". Detén y relanza. Solo debe quedar "Comprar leche".

**Salida esperada:** Los datos sobreviven a cierres y relanzamientos de la app.

**Verificación:** Tras cada relanzamiento, la lista refleja exactamente el estado guardado.

9. **Revierte** el cambio temporal de `TaskListViewController` (vuelve a `InMemoryTaskRepository(seedSampleData: true)` por ahora). Lo configuraremos con el selector en el paso final.

---

#### Paso 11: Commit de la Parte 2

```bash
cd ~/Developer/TaskManagerApp
git add -A
git commit -m "feat: add Core Data stack and CoreDataTaskRepository"
```

---

### PARTE 3 — SwiftData: Repositorio Moderno (80 min)

---

#### Paso 12: Crear el modelo SwiftData `SDTaskItem`

**Objetivo:** Definir un modelo SwiftData con la macro `@Model` como alternativa declarativa a Core Data.

**Instrucciones:**

1. Crea un nuevo grupo llamado `SwiftDataModel` en el navegador de Xcode.

2. Dentro de `SwiftDataModel`, crea `SDTaskItem.swift`:

```swift
// SDTaskItem.swift
import Foundation
import SwiftData

@Model
final class SDTaskItem {
    @Attribute(.unique) var id: UUID
    var title: String
    var taskDescription: String
    var dueDate: Date?
    var priorityRaw: Int          // SwiftData guarda el rawValue del enum
    var isCompleted: Bool
    var createdAt: Date

    /// Prioridad tipada; propiedad calculada, SwiftData no la almacena.
    var priority: Priority {
        get { Priority(rawValue: priorityRaw) ?? .medium }
        set { priorityRaw = newValue.rawValue }
    }

    init(
        id: UUID = UUID(),
        title: String,
        taskDescription: String = "",
        dueDate: Date? = nil,
        priority: Priority = .medium,
        isCompleted: Bool = false,
        createdAt: Date = Date()
    ) {
        self.id = id
        self.title = title
        self.taskDescription = taskDescription
        self.dueDate = dueDate
        self.priorityRaw = priority.rawValue
        self.isCompleted = isCompleted
        self.createdAt = createdAt
    }
}
```

> **Nota:** En `SDTaskItem`, `taskDescription` es `String` (vacío si no hay descripción); en `TaskItem` es `String?`. Las funciones de conversión hacen ese ajuste. Usamos el prefijo `SD` para evitar conflictos de nombre con la struct `TaskItem` existente. A partir de aquí, el proyecto tiene dos tipos con roles distintos: `TaskItem` es el modelo de dominio (lo usan los repositorios, la red y las pruebas) y `SDTaskItem` es el modelo de persistencia (lo usan `@Query`, `ModelContext` y `.modelContainer`). Los Labs 04 y 05 siguen esta misma regla. El `@Attribute(.unique)` garantiza que no se dupliquen registros por `id`.

3. Añade un método de conveniencia para convertir entre `SDTaskItem` y `TaskItem`:

```swift
// SDTaskItem+Conversion.swift (mismo grupo SwiftDataModel)
import Foundation

extension SDTaskItem {
    /// Crea el modelo SwiftData a partir de la struct TaskItem del dominio.
    convenience init(from task: TaskItem) {
        self.init(
            id: task.id,
            title: task.title,
            taskDescription: task.taskDescription ?? "",
            dueDate: task.dueDate,
            priority: task.priority,
            isCompleted: task.isCompleted,
            createdAt: task.createdAt
        )
    }

    /// Convierte el modelo SwiftData a la struct TaskItem del dominio.
    func toTaskItem() -> TaskItem {
        TaskItem(
            id: id,
            title: title,
            taskDescription: taskDescription.isEmpty ? nil : taskDescription,
            isCompleted: isCompleted,
            priority: priority,
            createdAt: createdAt,
            dueDate: dueDate
        )
    }

    /// Actualiza las propiedades desde una struct TaskItem.
    func update(from task: TaskItem) {
        title = task.title
        taskDescription = task.taskDescription ?? ""
        dueDate = task.dueDate
        priority = task.priority
        isCompleted = task.isCompleted
    }
}
```

4. Guarda con **⌘ S** y compila con **⌘ B**.

**Salida esperada:** Compilación exitosa. SwiftData genera automáticamente el esquema; no se necesita archivo `.xcdatamodeld`.

**Verificación:** No hay errores ni advertencias.

---

#### Paso 13: Configurar el `ModelContainer` compartido

**Objetivo:** Crear un único contenedor de SwiftData para toda la app y ponerlo a disposición de UIKit y de las vistas SwiftUI.

**Instrucciones:**

1. `TaskManagerApp` usa el ciclo de vida de UIKit desde el Lab 01 (`AppDelegate` con `@main`). **No crees** una estructura `App` de SwiftUI con `@main`: el proyecto tendría dos puntos de entrada y no compilaría. En su lugar, dentro del grupo `SwiftDataModel`, crea `AppModelContainer.swift`:

```swift
// AppModelContainer.swift
import SwiftData

/// Contenedor único de SwiftData para toda la app (almacenamiento en disco).
enum AppModelContainer {
    static let shared: ModelContainer = {
        do {
            return try ModelContainer(for: SDTaskItem.self)
        } catch {
            fatalError("No se pudo crear el ModelContainer: \(error)")
        }
    }()
}
```

2. Abre `AppDelegate.swift` y, en `application(_:didFinishLaunchingWithOptions:)`, registra las preferencias por defecto de la Parte 1 (antes del `return true`):

```swift
UserDefaults.standard.register(defaults: [
    "userPref_sortOrder":          SortOrder.createdAtDescending.rawValue,
    "userPref_colorTheme":         ColorTheme.system.rawValue,
    "userPref_showCompletedTasks": true
])
```

3. Cada vez que muestres una vista SwiftUI que use SwiftData desde UIKit, aplica el contenedor con `.modelContainer(AppModelContainer.shared)` antes de envolverla en un `UIHostingController`. Lo harás en el Paso 15.

> **Importante:** `.modelContainer(...)` inyecta el contenedor en el entorno de SwiftUI; así las vistas hijas pueden usar `@Query` y `@Environment(\.modelContext)`. Para código UIKit (o para los repositorios) usa `AppModelContainer.shared.mainContext`.

4. Guarda y compila (**⌘ B**).

**Salida esperada:** Compilación exitosa.

**Verificación:** Ejecuta la app (**⌘ R**). No debe haber crashes al iniciar. Si más adelante ves el error `Failed to find a currently active container`, revisa que la vista SwiftUI tenga aplicado `.modelContainer(AppModelContainer.shared)`.

---

#### Paso 14: Implementar `SwiftDataTaskRepository`

**Objetivo:** Crear un repositorio que conforme `TaskRepository` usando `ModelContext` de SwiftData.

**Instrucciones:**

1. Dentro del grupo `SwiftDataModel`, crea `SwiftDataTaskRepository.swift`:

```swift
// SwiftDataTaskRepository.swift
import Foundation
import SwiftData

final class SwiftDataTaskRepository: TaskRepository {

    private let modelContext: ModelContext

    init(modelContext: ModelContext) {
        self.modelContext = modelContext
    }

    // MARK: - Fetch All
    func fetchAll() -> [TaskItem] {
        let sortDescriptor = SortDescriptor<SDTaskItem>(\.createdAt, order: .reverse)
        let fetchDescriptor = FetchDescriptor<SDTaskItem>(sortBy: [sortDescriptor])

        do {
            let sdItems = try modelContext.fetch(fetchDescriptor)
            return sdItems.map { $0.toTaskItem() }
        } catch {
            print("Error al obtener tareas de SwiftData: \(error)")
            return []
        }
    }

    // MARK: - Add
    func add(_ task: TaskItem) {
        let sdItem = SDTaskItem(from: task)
        modelContext.insert(sdItem)
        saveContext()
    }

    // MARK: - Fetch by ID
    func fetchById(_ id: UUID) -> TaskItem? {
        findItem(id: id)?.toTaskItem()
    }

    // MARK: - Update
    @discardableResult
    func update(_ task: TaskItem) -> Bool {
        guard let sdItem = findItem(id: task.id) else { return false }
        sdItem.update(from: task)
        saveContext()
        return true
    }

    // MARK: - Delete
    @discardableResult
    func delete(byId id: UUID) -> Bool {
        guard let sdItem = findItem(id: id) else { return false }
        modelContext.delete(sdItem)
        saveContext()
        return true
    }

    // MARK: - Filter
    func filter(where predicate: (TaskItem) -> Bool) -> [TaskItem] {
        fetchAll().filter(predicate)
    }

    // MARK: - Private
    /// Busca el registro SwiftData por id. La variable local con tipo explícito
    /// evita el error "Type of expression is ambiguous" del macro #Predicate.
    private func findItem(id: UUID) -> SDTaskItem? {
        let targetID: UUID = id
        let descriptor = FetchDescriptor<SDTaskItem>(
            predicate: #Predicate { $0.id == targetID }
        )
        do {
            return try modelContext.fetch(descriptor).first
        } catch {
            print("Error al buscar tarea en SwiftData: \(error)")
            return nil
        }
    }

    private func saveContext() {
        do {
            try modelContext.save()
        } catch {
            print("Error al guardar contexto SwiftData: \(error)")
        }
    }
}
```

2. Guarda y compila (**⌘ B**).

**Salida esperada:** Compilación exitosa.

**Verificación:** No hay errores. El tipo `SwiftDataTaskRepository` conforma `TaskRepository`.

---

#### Paso 15: Integrar `@Query` en la vista SwiftUI de tareas

**Objetivo:** Migrar la lista de tareas SwiftUI para que use `@Query` de SwiftData directamente, aprovechando la reactividad automática.

**Instrucciones:**

1. Crea (o modifica) una vista de lista que use `@Query`. Ejemplo con una vista nueva `SwiftDataTaskListView.swift` dentro del grupo de vistas SwiftUI:

```swift
// SwiftDataTaskListView.swift
import SwiftUI
import SwiftData

struct SwiftDataTaskListView: View {

    @Environment(\.modelContext) private var modelContext
    @Query(sort: \SDTaskItem.createdAt, order: .reverse) private var tasks: [SDTaskItem]

    @State private var showingAddTask = false
    @State private var newTaskTitle = ""
    @State private var newTaskDescription = ""

    var body: some View {
        NavigationStack {
            List {
                ForEach(tasks) { task in
                    HStack {
                        VStack(alignment: .leading, spacing: 4) {
                            Text(task.title)
                                .font(.headline)
                                .strikethrough(task.isCompleted)
                                .foregroundColor(task.isCompleted ? .secondary : .primary)

                            if !task.taskDescription.isEmpty {
                                Text(task.taskDescription)
                                    .font(.subheadline)
                                    .foregroundColor(.secondary)
                            }

                            Text(task.createdAt, style: .date)
                                .font(.caption2)
                                .foregroundColor(.gray)
                        }

                        Spacer()

                        Button {
                            task.isCompleted.toggle()
                            try? modelContext.save()
                        } label: {
                            Image(systemName: task.isCompleted ? "checkmark.circle.fill" : "circle")
                                .foregroundColor(task.isCompleted ? .green : .gray)
                                .font(.title2)
                        }
                        .buttonStyle(.plain)
                    }
                }
                .onDelete(perform: deleteTasks)
            }
            .navigationTitle("Tareas (SwiftData)")
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    Button {
                        showingAddTask = true
                    } label: {
                        Image(systemName: "plus")
                    }
                }
                ToolbarItem(placement: .navigationBarLeading) {
                    NavigationLink(destination: PreferencesView()) {
                        Image(systemName: "gearshape")
                    }
                }
            }
            .sheet(isPresented: $showingAddTask) {
                addTaskSheet
            }
            .overlay {
                if tasks.isEmpty {
                    ContentUnavailableView(
                        "Sin tareas",
                        systemImage: "checklist",
                        description: Text("Pulsa + para añadir tu primera tarea.")
                    )
                }
            }
        }
    }

    // MARK: - Add Task Sheet
    private var addTaskSheet: some View {
        NavigationStack {
            Form {
                TextField("Título", text: $newTaskTitle)
                TextField("Descripción (opcional)", text: $newTaskDescription)
            }
            .navigationTitle("Nueva Tarea")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("Cancelar") {
                        resetForm()
                        showingAddTask = false
                    }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("Guardar") {
                        addTask()
                        showingAddTask = false
                    }
                    .disabled(newTaskTitle.trimmingCharacters(in: .whitespaces).isEmpty)
                }
            }
        }
        .presentationDetents([.medium])
    }

    // MARK: - Actions
    private func addTask() {
        let task = SDTaskItem(
            title: newTaskTitle.trimmingCharacters(in: .whitespaces),
            taskDescription: newTaskDescription.trimmingCharacters(in: .whitespaces)
        )
        modelContext.insert(task)
        try? modelContext.save()
        resetForm()
    }

    private func deleteTasks(at offsets: IndexSet) {
        for index in offsets {
            modelContext.delete(tasks[index])
        }
        try? modelContext.save()
    }

    private func resetForm() {
        newTaskTitle = ""
        newTaskDescription = ""
    }
}

#Preview {
    SwiftDataTaskListView()
        .modelContainer(for: SDTaskItem.self, inMemory: true)
}
```

2. Para abrir esta vista desde la app UIKit, agrega en `TaskListViewController.swift` una acción (por ejemplo, conectada a un nuevo `UIBarButtonItem` "SwiftData" en el Storyboard, igual que el botón "SwiftUI" del Lab 02):

```swift
@IBAction func showSwiftDataTapped(_ sender: UIBarButtonItem) {
    let view = SwiftDataTaskListView()
        .modelContainer(AppModelContainer.shared)
    let hostingController = UIHostingController(rootView: view)
    hostingController.title = "SwiftData"
    navigationController?.pushViewController(hostingController, animated: true)
}
```

3. Guarda y compila (**⌘ B**).

4. Ejecuta en el simulador (**⌘ R**), toca **SwiftData**, añade tareas, márcalas como completadas y elimina alguna. Detén y relanza para verificar persistencia.

**Salida esperada:** La lista muestra las tareas almacenadas en SwiftData. Los cambios persisten entre relanzamientos. La vista `#Preview` funciona con un contenedor en memoria.

**Verificación:**
- Añade 3 tareas → detén → relanza → las 3 tareas aparecen.
- Marca 1 como completada → detén → relanza → sigue marcada.
- Elimina 1 con swipe → detén → relanza → solo quedan 2.

---

#### Paso 16: Commit de la Parte 3

```bash
cd ~/Developer/TaskManagerApp
git add -A
git commit -m "feat: add SwiftData model and SwiftDataTaskRepository"
```

---

### PARTE 4 — Integración: Selector de Repositorio (63 min)

---

#### Paso 17: Crear `AppConfiguration` con `RepositoryType`

**Objetivo:** Implementar un mecanismo centralizado para seleccionar la implementación de persistencia activa.

**Instrucciones:**

1. Crea un archivo `AppConfiguration.swift` en el grupo raíz del proyecto:

```swift
// AppConfiguration.swift
import Foundation
import SwiftData

/// Tipo de repositorio de persistencia activo.
enum RepositoryType: String, CaseIterable {
    case inMemory   = "En Memoria"
    case coreData   = "Core Data"
    case swiftData  = "SwiftData"
}

final class AppConfiguration {

    static let shared = AppConfiguration()

    /// Cambia este valor para seleccionar el repositorio activo.
    /// Para producción, se recomienda `.swiftData`.
    var activeRepository: RepositoryType = .swiftData

    private init() {}

    /// Crea una instancia de TaskRepository según la configuración activa.
    /// - Parameter modelContext: Contexto de SwiftData; si no se pasa, se usa
    ///   el contexto principal del contenedor compartido (Paso 13).
    @MainActor
    func makeRepository(modelContext: ModelContext? = nil) -> TaskRepository {
        switch activeRepository {
        case .inMemory:
            return InMemoryTaskRepository(seedSampleData: true)

        case .coreData:
            return CoreDataTaskRepository()

        case .swiftData:
            let context = modelContext ?? AppModelContainer.shared.mainContext
            return SwiftDataTaskRepository(modelContext: context)
        }
    }
}
```

2. Guarda con **⌘ S**.

**Salida esperada:** El archivo compila sin errores.

**Verificación:** Compila con **⌘ B**.

---

#### Paso 18: Usar `AppConfiguration` en `TaskListViewController`

**Objetivo:** Hacer que la lista principal (UIKit, del Lab 02) obtenga su repositorio desde `AppConfiguration` y aplique las preferencias de `UserDefaults`. Como el botón "SwiftUI" del Lab 02 pasa este mismo repositorio a `TaskListView`, ambas interfaces usarán la persistencia elegida.

**Instrucciones:**

1. Abre `UIKitViews/TaskListViewController.swift` y reemplaza la declaración del repositorio:

```swift
// Antes (Lab 02):
// private let repository: TaskRepository = InMemoryTaskRepository(seedSampleData: true)

// Después:
private lazy var repository: TaskRepository = AppConfiguration.shared.makeRepository()
```

> **Nota:** Se usa `lazy var` para que el repositorio se cree la primera vez que se usa (ya en el hilo principal), no al construir el controlador desde el Storyboard.

2. En el mismo archivo, reemplaza `loadTasks()` para aplicar el filtro y el orden guardados en `UserPreferencesService` (Parte 1):

```swift
private func loadTasks() {
    var result = repository.fetchAll()

    // Filtro de preferencias
    let prefs = UserPreferencesService.shared
    if !prefs.showCompletedTasks {
        result = result.filter { !$0.isCompleted }
    }

    // Ordenamiento de preferencias
    switch prefs.sortOrder {
    case .createdAtAscending:
        result.sort { $0.createdAt < $1.createdAt }
    case .createdAtDescending:
        result.sort { $0.createdAt > $1.createdAt }
    case .titleAscending:
        result.sort { $0.title.localizedCaseInsensitiveCompare($1.title) == .orderedAscending }
    case .titleDescending:
        result.sort { $0.title.localizedCaseInsensitiveCompare($1.title) == .orderedDescending }
    }

    tasks = result
    tableView.reloadData()
}
```

3. No hace falta cambiar nada más: `prepare(for:sender:)` y `showSwiftUITapped(_:)` ya pasan `repository` a las demás pantallas.

4. Guarda y compila (**⌘ B**).

**Salida esperada:** Compilación exitosa. La app usa el repositorio configurado en `AppConfiguration.shared.activeRepository`.

**Verificación:** Cambia `activeRepository` a `.coreData`, ejecuta, añade una tarea. Cambia a `.swiftData`, ejecuta — la tarea de Core Data no aparece (son stores separados). Cambia de vuelta a `.coreData` — la tarea reaparece. Esto confirma que cada repositorio opera independientemente. Con `.inMemory` verás las 3 tareas de ejemplo del Lab 02 y nada persiste al relanzar.

---

#### Paso 19: Crear tabla comparativa de tecnologías

**Objetivo:** Documentar la comparación entre las tres tecnologías como referencia del equipo.

**Instrucciones:**

1. Crea un archivo `PERSISTENCE_COMPARISON.md` en la raíz del proyecto:

```bash
cd ~/Developer/TaskManagerApp
cat > PERSISTENCE_COMPARISON.md << 'EOF'
# Comparación de Tecnologías de Persistencia

| Criterio                  | UserDefaults         | Core Data              | SwiftData              |
|---------------------------|----------------------|------------------------|------------------------|
| **Caso de uso**           | Preferencias simples | Modelos relacionales   | Modelos relacionales   |
| **Complejidad de setup**  | Mínima               | Alta (stack + modelo)  | Baja (macro @Model)    |
| **Tipos soportados**      | Plist-compatible     | Cualquiera (NSManaged) | Cualquiera (@Model)    |
| **Consultas**             | Clave-valor          | NSFetchRequest + NSPredicate | FetchDescriptor + #Predicate |
| **Reactividad SwiftUI**   | @AppStorage          | Manual / NSFetchedResultsController | @Query (automático)   |
| **Migraciones**           | N/A                  | Mapping models         | Automáticas (ligeras)  |
| **iOS mínimo**            | iOS 2+               | iOS 3+                 | iOS 17+               |
| **Líneas de código (CRUD)** | ~20                | ~120                   | ~70                    |

## Decisión

**SwiftData** es el repositorio principal de producción porque:
1. Menor código boilerplate que Core Data.
2. Integración nativa con SwiftUI via `@Query`.
3. Predicados type-safe con `#Predicate`.
4. Nuestro deployment target (iOS 17.0) lo soporta.

**UserDefaults** se mantiene exclusivamente para preferencias del usuario.
**Core Data** queda disponible como fallback y referencia educativa.
EOF
```

2. Verifica que el archivo se creó:

```bash
cat PERSISTENCE_COMPARISON.md | head -5
```

**Salida esperada:** Las primeras líneas del archivo Markdown.

---

#### Paso 20: Commit final y merge a develop

**Objetivo:** Consolidar todo el trabajo y fusionar la rama de feature.

**Instrucciones:**

1. Añade y confirma todos los cambios:

```bash
cd ~/Developer/TaskManagerApp
git add -A
git commit -m "feat: persistence layer with UserDefaults, CoreData and SwiftData

- UserPreferencesService for user preferences (sort, theme, filter)
- CoreDataStack + CoreDataTaskRepository with full CRUD
- SDTaskItem @Model + SwiftDataTaskRepository with full CRUD
- AppConfiguration with RepositoryType selector
- SwiftData selected as production repository
- PreferencesView integrated in both UIKit and SwiftUI navigation"
```

2. Empuja la rama al repositorio remoto:

```bash
git push -u origin feature/persistence
```

3. Fusiona a `develop`:

```bash
git checkout develop
git merge feature/persistence
git push origin develop
```

4. Verifica el historial:

```bash
git log --oneline -5
```

**Salida esperada:**

```
abc1234 feat: persistence layer with UserDefaults, CoreData and SwiftData
def5678 feat: add SwiftData model and SwiftDataTaskRepository
ghi9012 feat: add Core Data stack and CoreDataTaskRepository
jkl3456 feat: add UserPreferencesService with UserDefaults persistence
mno7890 (commits anteriores del Lab 02)
```

**Verificación:** Los 4 commits de este lab aparecen en `develop`. La rama `feature/persistence` puede verse en GitHub.

---

## 7. Validación y Pruebas

Ejecuta las siguientes verificaciones para confirmar que todo el laboratorio funciona correctamente:

### Verificación 1: UserDefaults

1. Ejecuta la app en el simulador (**⌘ R**).
2. Navega a **Preferencias**.
3. Cambia el tema a "Azul" y el orden a "Título A-Z".
4. Desactiva "Mostrar tareas completadas".
5. Detén la app (**⌘ .**) y relanza (**⌘ R**).
6. Navega a Preferencias: los valores deben ser "Azul", "Título A-Z" y toggle desactivado.

### Verificación 2: Core Data

1. En `AppConfiguration.swift`, cambia `activeRepository` a `.coreData`.
2. Ejecuta la app, añade 2 tareas, marca 1 como completada.
3. Detén y relanza. Las tareas deben persistir con su estado.

### Verificación 3: SwiftData

1. Cambia `activeRepository` a `.swiftData`.
2. Ejecuta la app, añade 3 tareas.
3. Detén y relanza. Las 3 tareas deben aparecer.
4. Elimina 1 tarea con swipe. Detén y relanza. Solo quedan 2.

### Verificación 4: Compilación limpia

```bash
cd ~/Developer/TaskManagerApp
xcodebuild -scheme TaskManagerApp \
  -destination 'platform=iOS Simulator,name=iPhone 15 Pro,OS=17.5' \
  clean build 2>&1 | tail -3
```

**Salida esperada:** `** BUILD SUCCEEDED **`

### Verificación 5: Git

```bash
git log --oneline --graph develop | head -10
```

Debe mostrar los commits del lab en la rama `develop`.

---

## 8. Solución de Problemas

### Problema 1: Error "Failed to load model named TaskManagerApp"

**Síntomas:** La app crashea al iniciar con el mensaje:
```
CoreData: error: Failed to load model named TaskManagerApp
```

**Causa:** El nombre del archivo `.xcdatamodeld` no coincide con el nombre pasado a `NSPersistentContainer(name:)`. Xcode puede haber creado el archivo con un nombre diferente (por ejemplo, `Model.xcdatamodeld`).

**Solución:**
1. En el navegador de Xcode, localiza el archivo `.xcdatamodeld` y verifica su nombre exacto.
2. En `CoreDataStack.swift`, asegúrate de que el nombre coincide:
```swift
// Si el archivo se llama "TaskManagerApp.xcdatamodeld":
persistentContainer = NSPersistentContainer(name: "TaskManagerApp")

// Si se llama "Model.xcdatamodeld":
persistentContainer = NSPersistentContainer(name: "Model")
```
3. Limpia el build (**⌘ ⇧ K**), borra datos del simulador (**Device → Erase All Content and Settings**) y vuelve a ejecutar.

---

### Problema 2: Error de compilación "Ambiguous use of 'id'" en `#Predicate` de SwiftData

**Síntomas:** Al compilar `SwiftDataTaskRepository`, Xcode muestra:
```
Type of expression is ambiguous without a type annotation
```
en la línea del `#Predicate`.

**Causa:** El macro `#Predicate` requiere que la variable capturada sea del mismo tipo exacto que la propiedad del modelo. Si `task.id` es `UUID` pero la variable capturada tiene un tipo inferido diferente, el compilador falla.

**Solución:**
1. Declara explícitamente el tipo de la variable antes del predicado, como en el método `findItem(id:)` del Paso 14:
```swift
private func findItem(id: UUID) -> SDTaskItem? {
    let targetID: UUID = id  // Tipo explícito
    let descriptor = FetchDescriptor<SDTaskItem>(
        predicate: #Predicate { $0.id == targetID }
    )
    return try? modelContext.fetch(descriptor).first
}
```
2. Usa `findItem(id:)` en `fetchById(_:)`, `update(_:)` y `delete(byId:)` en lugar de escribir un `#Predicate` en cada método.
3. Limpia (**⌘ ⇧ K**) y recompila (**⌘ B**).

---

## 9. Limpieza

No es necesario eliminar archivos ni recursos. Todo el código forma parte del proyecto incremental `TaskManagerApp`.

Si deseas limpiar los datos de prueba del simulador para empezar el siguiente lab con una base limpia:

1. En el simulador: **Device → Erase All Content and Settings**.
2. O desde Terminal:

```bash
xcrun simctl erase "iPhone 15 Pro"
```

Opcionalmente, establece el repositorio de producción definitivo:

```swift
// AppConfiguration.swift — configuración final
var activeRepository: RepositoryType = .swiftData
```

---

## Reto Opcional (KMP) — Persistencia Compartida con SQLDelight

> Este reto es **opcional** y no forma parte de la evaluación. Se trabaja sobre el proyecto KMP del Lab 01.

**Objetivo:** Guardar las tareas en el módulo `shared` con SQLDelight y mostrarlas en la app iOS.

**Instrucciones:**

1. Agrega el plugin y las dependencias de SQLDelight al `build.gradle.kts` del módulo `shared` (driver de Android y driver nativo para iOS).
2. Crea el archivo `Tarea.sq` en `shared/src/commonMain/sqldelight` con la tabla de tareas y las consultas para insertar, listar y eliminar.
3. Implementa `TareasRepositorio` en `commonMain` usando las consultas generadas por SQLDelight.
4. Crea el driver de cada plataforma con `expect`/`actual` (o con una fábrica de driver por plataforma).
5. Desde la app iOS, llama al repositorio y muestra la lista de tareas.

**Resultado esperado:** las tareas se guardan con SQLDelight en el módulo `shared`, la app iOS las muestra y se conservan al cerrar y volver a abrir la app.

---

## 10. Resumen

En este laboratorio implementaste una capa de persistencia completa para `TaskManagerApp`:

| Tecnología | Propósito | Archivos creados |
|---|---|---|
| **UserDefaults** | Preferencias del usuario | `UserPreferences.swift`, `UserPreferencesService.swift`, `PreferencesView.swift` |
| **Core Data** | Repositorio relacional | `TaskManagerApp.xcdatamodeld`, `TaskEntity+CoreDataClass.swift`, `TaskEntity+CoreDataProperties.swift`, `CoreDataStack.swift`, `CoreDataTaskRepository.swift`, `TaskItem+Persistence.swift` |
| **SwiftData** | Repositorio moderno declarativo | `SDTaskItem.swift`, `SDTaskItem+Conversion.swift`, `SwiftDataTaskRepository.swift`, `SwiftDataTaskListView.swift`, `AppModelContainer.swift` |
| **Integración** | Selector de implementación | `AppConfiguration.swift`, `TaskListViewController.swift` (actualizado), `PERSISTENCE_COMPARISON.md` |

**Decisión de arquitectura:** SwiftData fue seleccionado como repositorio principal de producción por su menor boilerplate, integración nativa con SwiftUI (`@Query`) y soporte de predicados type-safe.

### Recursos adicionales

- [Apple Documentation — UserDefaults](https://developer.apple.com/documentation/foundation/userdefaults)
- [Apple Documentation — Core Data](https://developer.apple.com/documentation/coredata)
- [Apple Documentation — SwiftData](https://developer.apple.com/documentation/swiftdata)
- [WWDC23 — Meet SwiftData](https://developer.apple.com/videos/play/wwdc2023/10187/)
- [WWDC23 — Build an app with SwiftData](https://developer.apple.com/videos/play/wwdc2023/10154/)

---

## Anexo — Notas sobre la presentación (diapositivas 214 y 215, práctica del Capítulo 3)

Los criterios de la diapositiva 214 siguen siendo válidos para este README. Las notas de abajo aclaran **cómo** se cumple cada uno en el proyecto de los Labs 01–02, y qué falta en el texto del reto KMP. Mientras la diapositiva no se actualice, **manda el contenido de este README**.

| Criterio en la diapositiva 214 | Cómo se cumple en este README | Por qué |
|---|---|---|
| **UserDefaults:** `UserPreferencesService`, `SortOrder` y `PreferencesView` | Sin cambios. Las preferencias por defecto se registran en `AppDelegate` (Paso 13). El botón de preferencias se **agrega** junto al **+** con `rightBarButtonItems` (Paso 4). | El proyecto arranca en UIKit desde el Lab 01. Si se asignara solo `rightBarButtonItem`, el engranaje reemplazaría al botón **+**. |
| **Core Data:** `TaskEntity`, `CoreDataStack` y `CoreDataTaskRepository` con el protocolo `TaskRepository` | El repositorio implementa el protocolo del Lab 02: `fetchAll()`, `fetchById(_:)`, `add(_:)`, `update(_:) -> Bool`, `delete(byId:) -> Bool` y `filter(where:)`. Se agrega `TaskItem+Persistence.swift`, con un inicializador que conserva `id` y `createdAt` (Paso 9). | El inicializador del Lab 01 genera un `id` y una fecha nuevos cada vez, así que no sirve para reconstruir tareas guardadas. |
| **SwiftData:** `@Model SDTaskItem`, `ModelContainer`, `SwiftDataTaskRepository` y lista con `@Query` | El contenedor es `AppModelContainer.shared` y **no** un `App` de SwiftUI con `@main` (Paso 13). La lista `SwiftDataTaskListView` se abre con un botón **SwiftData** desde la lista UIKit (Paso 15). | El proyecto ya tiene `@main` en `AppDelegate`; con dos puntos de entrada no compila. |
| *(implícito)* Relación entre los dos modelos | `TaskItem` es el modelo de dominio (`taskDescription` opcional) y `SDTaskItem` es el modelo de persistencia (`taskDescription` no opcional y `priorityRaw`). `init(from:)`, `toTaskItem()` y `update(from:)` convierten entre ambos (Paso 12). | Los Labs 04–06 siguen esta misma regla. |
| **Integración:** `AppConfiguration` elige la implementación; SwiftData es el repositorio principal | `makeRepository(modelContext:)` usa `AppModelContainer.shared.mainContext` cuando no recibe un contexto. El Paso 18 modifica **`TaskListViewController`** (Lab 02), que obtiene el repositorio con `lazy var repository = AppConfiguration.shared.makeRepository()`. | En ningún lab existe un `TaskListViewModel`. La pantalla principal es el ViewController UIKit del Lab 02, que además pasa el repositorio a la vista SwiftUI. |
| **Reto opcional (KMP):** "SQLDelight (`Tarea.sq` y un `TareasRepositorio`)" | `TareasRepositorio` se declara como **interfaz** en `commonMain` y se implementa en `SQLDelightTareasRepositorio`. La app iOS depende de la interfaz (ver `Retos_KMP_por_capitulo.md`). | El reto 6 crea un `FakeTareasRepositorio` que implementa esa interfaz, y la diapositiva «SOLID en el Módulo Compartido» muestra el mismo diseño. Si `TareasRepositorio` fuera una clase, el reto 6 no se podría hacer. |

**Reto KMP en la diapositiva 214:**

> **Reto opcional (KMP):** En el proyecto KMP, declara la interfaz `TareasRepositorio` en `commonMain`, impleméntala con SQLDelight (`Tarea.sq` y `SQLDelightTareasRepositorio`) y muestra las tareas en la app iOS.

**Reto KMP en la diapositiva 215 (Resultados esperados):**

> **Reto KMP (opcional):** la interfaz `TareasRepositorio` vive en el módulo `shared`, `SQLDelightTareasRepositorio` guarda las tareas y la app iOS las muestra sin depender de la implementación.

**Resultados esperados (diapositiva 215):** los demás puntos siguen siendo válidos. La lista con `@Query` está en la pantalla **SwiftData**, a la que se llega desde la lista UIKit. Al cambiar `RepositoryType`, cambia el repositorio que usan la lista UIKit y la vista SwiftUI del Lab 02.