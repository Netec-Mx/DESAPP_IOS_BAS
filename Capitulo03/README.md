# Práctica Incremental — Capa de Persistencia con UserDefaults, Core Data y SwiftData

## 1. Metadatos

| Campo | Valor |
|---|---|
| **Duración** | 288 minutos (~4 h 48 min) |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |

## 2. Descripción General

En este laboratorio reemplazarás el `InMemoryTaskRepository` construido en el Lab 02 con tres implementaciones de persistencia real: **UserDefaults** para preferencias del usuario, **Core Data** para un repositorio relacional completo y **SwiftData** como alternativa moderna declarativa. Al finalizar, dispondrás de un `enum RepositoryType` que permite conmutar entre las tres implementaciones, y habrás seleccionado SwiftData como repositorio principal de producción. Todo el trabajo se realizará en la rama `feature/persistence` y se integrará a `develop`.

## 3. Objetivos de Aprendizaje

- [ ] Implementar un servicio `UserPreferencesService` que persista preferencias de usuario (tema de color, ordenamiento, filtro de tareas completadas) usando `UserDefaults`.
- [ ] Construir un stack de Core Data completo con entidad `TaskEntity`, `NSPersistentContainer` y un `CoreDataTaskRepository` que conforme el protocolo `TaskRepository`.
- [ ] Crear un modelo `@Model TaskItem` con SwiftData e implementar `SwiftDataTaskRepository` usando `ModelContainer` y `ModelContext`.
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

**Verificación:** Compila con **⌘ B**. Verifica que `UserPreferencesService.shared.sortOrder` devuelve `.createdAtDescending` por defecto añadiendo temporalmente un `print` en `AppDelegate` o en el `init` de tu `App` struct (elimínalo después de verificar).

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

1. En tu vista principal de SwiftUI (por ejemplo, `TaskListView` o `ContentView`), añade un botón de navegación a preferencias en la toolbar:

```swift
.toolbar {
    ToolbarItem(placement: .navigationBarTrailing) {
        NavigationLink(destination: PreferencesView()) {
            Image(systemName: "gearshape")
                .accessibilityLabel("Preferencias")
        }
    }
}
```

2. Si usas UIKit con un `UINavigationController`, añade un `UIBarButtonItem` que presente `PreferencesView` dentro de un `UIHostingController`:

```swift
// En tu UIKit ViewController — viewDidLoad()
let gearButton = UIBarButtonItem(
    image: UIImage(systemName: "gearshape"),
    style: .plain,
    target: self,
    action: #selector(openPreferences)
)
navigationItem.rightBarButtonItem = gearButton

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

1. Primero, verifica que tu protocolo `TaskRepository` del Lab 02 tiene esta forma (o similar). Si difiere, adapta los nombres de método. Referencia esperada:

```swift
// TaskRepository.swift (ya existente del Lab 02)
import Foundation

protocol TaskRepository {
    func fetchAll() -> [TaskItem]
    func add(_ task: TaskItem)
    func update(_ task: TaskItem)
    func delete(id: UUID)
}
```

> **Importante:** Si en tu Lab 02 el modelo se llama `Task` en lugar de `TaskItem`, usa el nombre que tengas. En este lab usaremos `TaskItem` para evitar conflictos con `Swift.Task` (concurrencia). Si necesitas renombrar, hazlo ahora globalmente con **Edit → Find and Replace in Workspace**.

2. Asegúrate de que tu modelo `TaskItem` (struct del Lab 02) tiene al menos estas propiedades:

```swift
// TaskItem.swift (ya existente, verificar)
import Foundation

struct TaskItem: Identifiable, Equatable {
    let id: UUID
    var title: String
    var taskDescription: String
    var isCompleted: Bool
    var createdAt: Date

    init(
        id: UUID = UUID(),
        title: String,
        taskDescription: String = "",
        isCompleted: Bool = false,
        createdAt: Date = Date()
    ) {
        self.id = id
        self.title = title
        self.taskDescription = taskDescription
        self.isCompleted = isCompleted
        self.createdAt = createdAt
    }
}
```

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
                    taskDescription: entity.taskDescription ?? "",
                    isCompleted: entity.isCompleted,
                    createdAt: entity.createdAt ?? Date()
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

    // MARK: - Update
    func update(_ task: TaskItem) {
        let request: NSFetchRequest<TaskEntity> = TaskEntity.fetchRequest()
        request.predicate = NSPredicate(format: "id == %@", task.id as CVarArg)

        do {
            let results = try context.fetch(request)
            if let entity = results.first {
                entity.title = task.title
                entity.taskDescription = task.taskDescription
                entity.isCompleted = task.isCompleted
                coreDataStack.saveContext()
            }
        } catch {
            print("Error al actualizar tarea en Core Data: \(error)")
        }
    }

    // MARK: - Delete
    func delete(id: UUID) {
        let request: NSFetchRequest<TaskEntity> = TaskEntity.fetchRequest()
        request.predicate = NSPredicate(format: "id == %@", id as CVarArg)

        do {
            let results = try context.fetch(request)
            if let entity = results.first {
                context.delete(entity)
                coreDataStack.saveContext()
            }
        } catch {
            print("Error al eliminar tarea de Core Data: \(error)")
        }
    }
}
```

4. Guarda con **⌘ S** y compila con **⌘ B**.

**Salida esperada:** Compilación exitosa sin errores.

**Verificación:** Revisa que `CoreDataTaskRepository` aparece en el autocompletado al escribir `let repo: TaskRepository = CoreData...`.

---

#### Paso 10: Probar Core Data en el simulador

**Objetivo:** Verificar que los datos persisten entre relanzamientos de la app.

**Instrucciones:**

1. Temporalmente, en tu vista principal o ViewModel, reemplaza el repositorio en uso por `CoreDataTaskRepository()`:

```swift
// Ejemplo en un ViewModel existente
// Antes:
// private let repository: TaskRepository = InMemoryTaskRepository()
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

9. **Revierte** el cambio temporal del ViewModel (vuelve a `InMemoryTaskRepository` por ahora). Lo configuraremos con el selector en el paso final.

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
    var isCompleted: Bool
    var createdAt: Date

    init(
        id: UUID = UUID(),
        title: String,
        taskDescription: String = "",
        isCompleted: Bool = false,
        createdAt: Date = Date()
    ) {
        self.id = id
        self.title = title
        self.taskDescription = taskDescription
        self.isCompleted = isCompleted
        self.createdAt = createdAt
    }
}
```

> **Nota:** Usamos el prefijo `SD` para evitar conflictos de nombre con la struct `TaskItem` existente. El `@Attribute(.unique)` garantiza que no se dupliquen registros por `id`.

3. Añade un método de conveniencia para convertir entre `SDTaskItem` y `TaskItem`:

```swift
// SDTaskItem+Conversion.swift (mismo grupo SwiftDataModel)
import Foundation

extension SDTaskItem {
    /// Convierte el modelo SwiftData a la struct TaskItem del dominio.
    func toTaskItem() -> TaskItem {
        TaskItem(
            id: id,
            title: title,
            taskDescription: taskDescription,
            isCompleted: isCompleted,
            createdAt: createdAt
        )
    }

    /// Actualiza las propiedades desde una struct TaskItem.
    func update(from task: TaskItem) {
        title = task.title
        taskDescription = task.taskDescription
        isCompleted = task.isCompleted
    }
}
```

4. Guarda con **⌘ S** y compila con **⌘ B**.

**Salida esperada:** Compilación exitosa. SwiftData genera automáticamente el esquema; no se necesita archivo `.xcdatamodeld`.

**Verificación:** No hay errores ni advertencias.

---

#### Paso 13: Configurar el `ModelContainer` en el punto de entrada de la app

**Objetivo:** Registrar el contenedor de SwiftData para que esté disponible en toda la jerarquía de vistas SwiftUI.

**Instrucciones:**

1. Abre el archivo del punto de entrada de tu app SwiftUI (por ejemplo, `TaskManagerAppApp.swift` o `TaskManagerApp.swift`):

```swift
// TaskManagerAppApp.swift
import SwiftUI
import SwiftData

@main
struct TaskManagerAppApp: App {

    init() {
        // Registrar preferencias por defecto
        UserDefaults.standard.register(defaults: [
            "userPref_sortOrder":          SortOrder.createdAtDescending.rawValue,
            "userPref_colorTheme":         ColorTheme.system.rawValue,
            "userPref_showCompletedTasks": true
        ])
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: SDTaskItem.self)
    }
}
```

> **Importante:** `.modelContainer(for: SDTaskItem.self)` crea automáticamente un `ModelContainer` con almacenamiento en disco y lo inyecta en el entorno de SwiftUI. Todas las vistas hijas pueden acceder a `@Environment(\.modelContext)`.

2. Si tu app usa `UIApplicationDelegate` (UIKit lifecycle), configura el contenedor en el `AppDelegate` o `SceneDelegate`:

```swift
// Solo si usas UIKit lifecycle
import SwiftData

// En scene(_:willConnectTo:options:)
let container = try! ModelContainer(for: SDTaskItem.self)
let contentView = ContentView()
    .modelContainer(container)
window.rootViewController = UIHostingController(rootView: contentView)
```

3. Guarda y compila (**⌘ B**).

**Salida esperada:** Compilación exitosa.

**Verificación:** Ejecuta la app (**⌘ R**). No debe haber crashes al iniciar. Si ves un error `Failed to find a currently active container`, revisa que `.modelContainer` esté aplicado en el nivel correcto de la jerarquía de vistas.

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
        let sdItem = SDTaskItem(
            id: task.id,
            title: task.title,
            taskDescription: task.taskDescription,
            isCompleted: task.isCompleted,
            createdAt: task.createdAt
        )
        modelContext.insert(sdItem)
        saveContext()
    }

    // MARK: - Update
    func update(_ task: TaskItem) {
        let predicate = #Predicate<SDTaskItem> { $0.id == task.id }
        let descriptor = FetchDescriptor<SDTaskItem>(predicate: predicate)

        do {
            let results = try modelContext.fetch(descriptor)
            if let sdItem = results.first {
                sdItem.update(from: task)
                saveContext()
            }
        } catch {
            print("Error al actualizar tarea en SwiftData: \(error)")
        }
    }

    // MARK: - Delete
    func delete(id: UUID) {
        let predicate = #Predicate<SDTaskItem> { $0.id == id }
        let descriptor = FetchDescriptor<SDTaskItem>(predicate: predicate)

        do {
            let results = try modelContext.fetch(descriptor)
            if let sdItem = results.first {
                modelContext.delete(sdItem)
                saveContext()
            }
        } catch {
            print("Error al eliminar tarea de SwiftData: \(error)")
        }
    }

    // MARK: - Private
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

2. Guarda y compila (**⌘ B**).

3. Ejecuta en el simulador (**⌘ R**). Añade tareas, márcalas como completadas, elimina alguna. Detén y relanza para verificar persistencia.

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
    /// - Parameter modelContext: Requerido solo cuando `activeRepository == .swiftData`.
    func makeRepository(modelContext: ModelContext? = nil) -> TaskRepository {
        switch activeRepository {
        case .inMemory:
            return InMemoryTaskRepository()

        case .coreData:
            return CoreDataTaskRepository()

        case .swiftData:
            guard let context = modelContext else {
                fatalError(
                    "SwiftData requiere un ModelContext. " +
                    "Pasa el modelContext del entorno al crear el repositorio."
                )
            }
            return SwiftDataTaskRepository(modelContext: context)
        }
    }
}
```

2. Guarda con **⌘ S**.

**Salida esperada:** El archivo compila sin errores.

**Verificación:** Compila con **⌘ B**.

---

#### Paso 18: Actualizar el ViewModel para usar `AppConfiguration`

**Objetivo:** Modificar el ViewModel principal para que obtenga su repositorio desde `AppConfiguration`.

**Instrucciones:**

1. Abre tu ViewModel principal (por ejemplo, `TaskListViewModel.swift`). Modifícalo para aceptar un `TaskRepository` inyectado:

```swift
// TaskListViewModel.swift
import Foundation
import Combine

final class TaskListViewModel: ObservableObject {

    @Published var tasks: [TaskItem] = []

    private let repository: TaskRepository

    init(repository: TaskRepository) {
        self.repository = repository
        loadTasks()
    }

    func loadTasks() {
        tasks = repository.fetchAll()

        // Aplicar filtro de preferencias
        let prefs = UserPreferencesService.shared
        if !prefs.showCompletedTasks {
            tasks = tasks.filter { !$0.isCompleted }
        }

        // Aplicar ordenamiento de preferencias
        switch prefs.sortOrder {
        case .createdAtAscending:
            tasks.sort { $0.createdAt < $1.createdAt }
        case .createdAtDescending:
            tasks.sort { $0.createdAt > $1.createdAt }
        case .titleAscending:
            tasks.sort { $0.title.localizedCaseInsensitiveCompare($1.title) == .orderedAscending }
        case .titleDescending:
            tasks.sort { $0.title.localizedCaseInsensitiveCompare($1.title) == .orderedDescending }
        }
    }

    func addTask(title: String, description: String = "") {
        let task = TaskItem(title: title, taskDescription: description)
        repository.add(task)
        loadTasks()
    }

    func toggleCompletion(for task: TaskItem) {
        var updated = task
        updated.isCompleted.toggle()
        repository.update(updated)
        loadTasks()
    }

    func deleteTask(id: UUID) {
        repository.delete(id: id)
        loadTasks()
    }
}
```

2. En la vista SwiftUI que crea el ViewModel, inyecta el repositorio usando el entorno:

```swift
// Ejemplo en ContentView.swift o la vista que instancie el ViewModel
import SwiftUI
import SwiftData

struct TaskListContainerView: View {

    @Environment(\.modelContext) private var modelContext

    @StateObject private var viewModel: TaskListViewModel

    init() {
        // Placeholder — se configura en onAppear
        _viewModel = StateObject(wrappedValue: TaskListViewModel(
            repository: InMemoryTaskRepository()
        ))
    }

    var body: some View {
        TaskListContentView(viewModel: viewModel)
            .onAppear {
                let repo = AppConfiguration.shared.makeRepository(
                    modelContext: modelContext
                )
                // Reemplazar repositorio si es necesario
                // Nota: Para una arquitectura más limpia, considera usar
                // un factory pattern en el nivel de la App struct.
            }
    }
}
```

> **Alternativa más limpia:** Si prefieres evitar la complejidad de reconfigurar el ViewModel después de `onAppear`, puedes crear el ViewModel directamente en la `App` struct pasando el `ModelContext`:

```swift
// Patrón recomendado: crear el ViewModel en el nivel de App
// usando un wrapper que tenga acceso al modelContext

struct RootView: View {
    @Environment(\.modelContext) private var modelContext

    var body: some View {
        let repository = AppConfiguration.shared.makeRepository(modelContext: modelContext)
        let viewModel = TaskListViewModel(repository: repository)

        // Usar .id() para forzar recreación si cambia la config
        TaskListMainView(viewModel: viewModel)
            .id(AppConfiguration.shared.activeRepository)
    }
}
```

3. Guarda y compila (**⌘ B**).

**Salida esperada:** Compilación exitosa. La app usa el repositorio configurado en `AppConfiguration.shared.activeRepository`.

**Verificación:** Cambia `activeRepository` a `.coreData`, ejecuta, añade una tarea. Cambia a `.swiftData`, ejecuta — la tarea de Core Data no aparece (son stores separados). Cambia de vuelta a `.coreData` — la tarea reaparece. Esto confirma que cada repositorio opera independientemente.

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
1. Declara explícitamente el tipo de la variable antes del predicado:
```swift
func update(_ task: TaskItem) {
    let taskID: UUID = task.id  // Tipo explícito
    let predicate = #Predicate<SDTaskItem> { item in
        item.id == taskID
    }
    let descriptor = FetchDescriptor<SDTaskItem>(predicate: predicate)
    // ... resto del código
}
```
2. Repite el mismo patrón en el método `delete(id:)`:
```swift
func delete(id: UUID) {
    let targetID: UUID = id
    let predicate = #Predicate<SDTaskItem> { item in
        item.id == targetID
    }
    // ...
}
```
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

## 10. Resumen

En este laboratorio implementaste una capa de persistencia completa para `TaskManagerApp`:

| Tecnología | Propósito | Archivos creados |
|---|---|---|
| **UserDefaults** | Preferencias del usuario | `UserPreferences.swift`, `UserPreferencesService.swift`, `PreferencesView.swift` |
| **Core Data** | Repositorio relacional | `TaskManagerApp.xcdatamodeld`, `TaskEntity+CoreDataClass.swift`, `TaskEntity+CoreDataProperties.swift`, `CoreDataStack.swift`, `CoreDataTaskRepository.swift` |
| **SwiftData** | Repositorio moderno declarativo | `SDTaskItem.swift`, `SDTaskItem+Conversion.swift`, `SwiftDataTaskRepository.swift`, `SwiftDataTaskListView.swift` |
| **Integración** | Selector de implementación | `AppConfiguration.swift`, `PERSISTENCE_COMPARISON.md` |

**Decisión de arquitectura:** SwiftData fue seleccionado como repositorio principal de producción por su menor boilerplate, integración nativa con SwiftUI (`@Query`) y soporte de predicados type-safe.

### Recursos adicionales

- [Apple Documentation — UserDefaults](https://developer.apple.com/documentation/foundation/userdefaults)
- [Apple Documentation — Core Data](https://developer.apple.com/documentation/coredata)
- [Apple Documentation — SwiftData](https://developer.apple.com/documentation/swiftdata)
- [WWDC23 — Meet SwiftData](https://developer.apple.com/videos/play/wwdc2023/10187/)
- [WWDC23 — Build an app with SwiftData](https://developer.apple.com/videos/play/wwdc2023/10154/)
