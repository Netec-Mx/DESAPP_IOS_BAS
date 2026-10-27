# 6 Práctica incremental — Creación del Proyecto Base TaskManagerApp

| Campo | Detalle |
|---|---|
| **Duración** | 216 minutos (≈ 3 h 36 min) |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |

---

## Descripción General

En este laboratorio fundacional crearás desde cero el proyecto Xcode **TaskManagerApp** que servirá como base incremental para todos los módulos del curso. Configurarás la estructura de carpetas estándar, explorarás los archivos generados por Xcode para comprender el ciclo de vida de la app (AppDelegate/SceneDelegate), inicializarás un repositorio Git con flujo profesional hacia GitHub, e implementarás el modelo de datos inicial en Swift puro (`TaskItem`, protocolo `TaskRepository`) aplicando sintaxis Swift 5.10 con closures, structs, protocolos y control de flujo. Finalizarás usando breakpoints y el debugger de Xcode para inspeccionar el ciclo de vida en el simulador iPhone 15 Pro.

---

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Crear y configurar un proyecto Xcode 15.4 (UIKit, Storyboard, iOS 17.0) con la estructura de carpetas estándar del curso en `~/Developer/TaskManagerApp`.
- [ ] Explicar el rol de `AppDelegate.swift`, `SceneDelegate.swift`, `Info.plist` y `Assets.xcassets` dentro de la arquitectura en capas de iOS y el ciclo de vida de la aplicación.
- [ ] Configurar un repositorio Git local con `.gitignore` apropiado, crear ramas `main`/`develop` y realizar push al repositorio remoto `task-manager-ios` en GitHub.
- [ ] Implementar el struct `TaskItem`, el protocolo `TaskRepository` y funciones de filtrado con closures aplicando sintaxis Swift 5.10 (opcionales, `guard`, `switch`, parámetros etiquetados).
- [ ] Utilizar breakpoints, la consola LLDB y la vista de variables en Xcode para inspeccionar el ciclo de vida y depurar el modelo de datos en el simulador iPhone 15 Pro (iOS 17.5).

---

## Prerrequisitos

### Conocimientos Previos

| Requisito | Descripción |
|---|---|
| Swift básico | Familiaridad con variables, constantes, tipos de datos, funciones y control de flujo en Swift |
| Xcode básico | Haber abierto Xcode al menos una vez y aceptado la licencia de uso |
| Git básico | Saber ejecutar `git init`, `git add`, `git commit` desde Terminal |
| GitHub | Cuenta activa con autenticación SSH o HTTPS configurada |

### Acceso Requerido

| Recurso | Detalle |
|---|---|
| Mac | macOS Ventura 13.6.7 o Sonoma 14.5, Apple Silicon o Intel 8.ª gen+, 16 GB RAM, 50 GB libres |
| Xcode 15.4 | Instalado desde App Store o developer.apple.com, licencia aceptada |
| Simulador | iPhone 15 Pro con iOS 17.5 descargado en Xcode |
| Git 2.45.2 | `user.name` y `user.email` configurados globalmente |
| GitHub | Repositorio `task-manager-ios` creado (privado), acceso SSH/HTTPS funcional |

---

## Entorno del Laboratorio

### Software

| Herramienta | Versión | Verificación |
|---|---|---|
| macOS | Ventura 13.6.7+ / Sonoma 14.5+ | `sw_vers` |
| Xcode | 15.4 | `xcodebuild -version` |
| Swift | 5.10 | `swift --version` |
| Git | 2.45.2 | `git --version` |
| Simulador | iPhone 15 Pro — iOS 17.5 | Xcode → Window → Devices and Simulators |

### Preparación Inicial del Entorno

Abre **Terminal** y ejecuta los siguientes comandos para verificar tu entorno antes de comenzar:

```bash
# Verificar versiones
sw_vers
xcodebuild -version
swift --version
git --version

# Verificar configuración global de Git
git config --global user.name
git config --global user.email

# Crear directorio de trabajo del curso (si no existe)
mkdir -p ~/Developer && cd ~/Developer
pwd
# Salida esperada: /Users/estudiante/Developer
# (donde "estudiante" será reemplazado por tu nombre de usuario de macOS;
#  puedes verificarlo ejecutando: whoami)
```

> **Nota:** Si `git config --global user.name` no muestra tu nombre, configúralo ahora:
> ```bash
> git config --global user.name "Tu Nombre"
> git config --global user.email "tu@email.com"
> ```

---

## Pasos del Laboratorio

### Paso 1 — Crear el Proyecto Xcode TaskManagerApp

**Objetivo:** Crear un nuevo proyecto Xcode con la plantilla UIKit/Storyboard, configurado con el deployment target iOS 17.0, en el directorio de trabajo del curso.

**Instrucciones:**

1. Abre **Xcode 15.4** desde el Dock o Spotlight (`⌘ + Espacio` → "Xcode").

2. En la ventana de bienvenida, selecciona **Create New Project…** (o menú **File → New → Project…**).

3. En la sección de plantillas, selecciona la pestaña **iOS** en la parte superior, luego elige **App** y haz clic en **Next**.

4. Configura las opciones del proyecto exactamente así:

   | Campo | Valor |
   |---|---|
   | Product Name | `TaskManagerApp` |
   | Team | Tu Apple ID personal (o "None") |
   | Organization Identifier | `com.cursounam` |
   | Bundle Identifier | `com.cursounam.taskmanagerapp` (se genera automáticamente) |
   | Interface | **Storyboard** |
   | Language | **Swift** |
   | Storage | **None** |
   | ☐ Include Tests | **Desmarcado** (los agregaremos manualmente en labs posteriores) |

   > **Importante:** Asegúrate de seleccionar **Storyboard** como Interface, NO SwiftUI. Esto genera los archivos `AppDelegate.swift` y `SceneDelegate.swift` que exploraremos.

5. Haz clic en **Next**. En el diálogo de ubicación:
   - Navega a `~/Developer` (puedes escribir `~/Developer` en la barra de ruta con `⌘ + Shift + G`).
   - **Desmarca** la casilla "Create Git repository on my Mac" (configuraremos Git manualmente).
   - Haz clic en **Create**.

6. Xcode creará la carpeta `~/Developer/TaskManagerApp/` con el proyecto dentro.

7. Una vez abierto el proyecto, configura el **Deployment Target**:
   - En el **Project Navigator** (panel izquierdo), haz clic en el icono azul del proyecto `TaskManagerApp` (primer elemento).
   - Selecciona el **Target** `TaskManagerApp` en la lista central.
   - Ve a la pestaña **General**.
   - En la sección **Minimum Deployments**, establece **iOS 17.0**.

8. Selecciona el simulador **iPhone 15 Pro** en la barra de esquemas (parte superior de Xcode, junto al botón de Play). Si no aparece, ve a **Window → Devices and Simulators** y verifica que el simulador con iOS 17.5 esté disponible.

9. Compila y ejecuta el proyecto presionando `⌘ + R`. El simulador debe abrirse mostrando una pantalla blanca con el texto predeterminado del Storyboard.

10. Detén la ejecución con `⌘ + .` (punto).

**Salida esperada:**

- El simulador iPhone 15 Pro muestra una pantalla blanca.
- La consola de Xcode no muestra errores (puede mostrar advertencias del sistema que son normales).
- El esquema muestra `TaskManagerApp > iPhone 15 Pro`.

**Verificación:**

```bash
# En Terminal, verifica que la estructura del proyecto existe
ls -la ~/Developer/TaskManagerApp/
# Debes ver: TaskManagerApp/ (carpeta de código), TaskManagerApp.xcodeproj
ls ~/Developer/TaskManagerApp/TaskManagerApp/
# Debes ver: AppDelegate.swift, SceneDelegate.swift, ViewController.swift,
#            Main.storyboard, Assets.xcassets, LaunchScreen.storyboard, Info.plist
```

---

### Paso 2 — Explorar la Arquitectura del Proyecto y el Ciclo de Vida

**Objetivo:** Comprender el rol de cada archivo generado por Xcode y su relación con las capas de la arquitectura iOS y el ciclo de vida de la aplicación.

**Instrucciones:**

1. En Xcode, expande la carpeta `TaskManagerApp` en el **Project Navigator** (⌘ + 1). Observa los archivos generados:

   | Archivo | Capa iOS | Rol |
   |---|---|---|
   | `AppDelegate.swift` | Cocoa Touch (UIKit) | Punto de entrada de la app; gestiona el ciclo de vida a nivel de aplicación |
   | `SceneDelegate.swift` | Cocoa Touch (UIKit) | Gestiona el ciclo de vida de cada escena/ventana (iOS 13+) |
   | `ViewController.swift` | Cocoa Touch (UIKit) | Controlador de la vista principal |
   | `Main.storyboard` | Cocoa Touch (UIKit) | Interfaz visual de la pantalla principal |
   | `Assets.xcassets` | Recurso del proyecto | Catálogo de imágenes, colores y recursos |
   | `LaunchScreen.storyboard` | Cocoa Touch (UIKit) | Pantalla de carga al iniciar la app |
   | `Info.plist` | Configuración del sistema | Metadatos de la app (permisos, configuraciones) |

2. Abre **`AppDelegate.swift`**. Observa que la clase hereda de `UIResponder` e implementa `UIApplicationDelegate`. Este archivo corresponde a la capa **Cocoa Touch** y gestiona las transiciones del ciclo de vida que estudiamos en la lección 1.1.

3. Agrega `print()` en los métodos del ciclo de vida de `AppDelegate.swift` para observar las transiciones. Reemplaza todo el contenido del archivo con:

   ```swift
   //
   //  AppDelegate.swift
   //  TaskManagerApp
   //
   //  Laboratorio 01-00-01: Exploración del ciclo de vida
   //

   import UIKit

   @main
   class AppDelegate: UIResponder, UIApplicationDelegate {

       func application(
           _ application: UIApplication,
           didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
       ) -> Bool {
           print("📱 [AppDelegate] didFinishLaunchingWithOptions — La app terminó de lanzarse")
           return true
       }

       // MARK: UISceneSession Lifecycle

       func application(
           _ application: UIApplication,
           configurationForConnecting connectingSceneSession: UISceneSession,
           options: UIScene.ConnectionOptions
       ) -> UISceneConfiguration {
           print("📱 [AppDelegate] configurationForConnecting — Configurando nueva escena")
           return UISceneConfiguration(
               name: "Default Configuration",
               sessionRole: connectingSceneSession.role
           )
       }

       func application(
           _ application: UIApplication,
           didDiscardSceneSessions sceneSessions: Set<UISceneSession>
       ) {
           print("📱 [AppDelegate] didDiscardSceneSessions — Escenas descartadas")
       }
   }
   ```

4. Abre **`SceneDelegate.swift`** y reemplaza su contenido con la versión instrumentada:

   ```swift
   //
   //  SceneDelegate.swift
   //  TaskManagerApp
   //
   //  Laboratorio 01-00-01: Exploración del ciclo de vida
   //

   import UIKit

   class SceneDelegate: UIResponder, UIWindowSceneDelegate {

       var window: UIWindow?

       func scene(
           _ scene: UIScene,
           willConnectTo session: UISceneSession,
           options connectionOptions: UIScene.ConnectionOptions
       ) {
           print("🪟 [SceneDelegate] willConnectTo — La escena se conectará a la sesión")
           guard let _ = (scene as? UIWindowScene) else { return }
       }

       func sceneDidDisconnect(_ scene: UIScene) {
           print("🪟 [SceneDelegate] sceneDidDisconnect — La escena se desconectó")
       }

       func sceneDidBecomeActive(_ scene: UIScene) {
           print("🪟 [SceneDelegate] sceneDidBecomeActive — La escena está ACTIVA")
       }

       func sceneWillResignActive(_ scene: UIScene) {
           print("🪟 [SceneDelegate] sceneWillResignActive — La escena dejará de estar activa")
       }

       func sceneWillEnterForeground(_ scene: UIScene) {
           print("🪟 [SceneDelegate] sceneWillEnterForeground — La escena entrará al primer plano")
       }

       func sceneDidEnterBackground(_ scene: UIScene) {
           print("🪟 [SceneDelegate] sceneDidEnterBackground — La escena entró al segundo plano")
       }
   }
   ```

5. Ejecuta la app (`⌘ + R`). Observa la consola de Xcode (panel inferior). Deberías ver los mensajes del lanzamiento en orden.

6. **Simula transiciones del ciclo de vida** en el simulador:
   - Presiona `⌘ + Shift + H` (Home) para enviar la app a segundo plano → observa los mensajes `sceneWillResignActive` y `sceneDidEnterBackground`.
   - Haz clic en el icono de la app en el simulador para volver → observa `sceneWillEnterForeground` y `sceneDidBecomeActive`.

7. Detén la ejecución (`⌘ + .`).

**Salida esperada en consola (al lanzar la app):**

```
📱 [AppDelegate] didFinishLaunchingWithOptions — La app terminó de lanzarse
📱 [AppDelegate] configurationForConnecting — Configurando nueva escena
🪟 [SceneDelegate] willConnectTo — La escena se conectará a la sesión
🪟 [SceneDelegate] sceneWillEnterForeground — La escena entrará al primer plano
🪟 [SceneDelegate] sceneDidBecomeActive — La escena está ACTIVA
```

**Salida esperada en consola (al presionar Home y volver):**

```
🪟 [SceneDelegate] sceneWillResignActive — La escena dejará de estar activa
🪟 [SceneDelegate] sceneDidEnterBackground — La escena entró al segundo plano
🪟 [SceneDelegate] sceneWillEnterForeground — La escena entrará al primer plano
🪟 [SceneDelegate] sceneDidBecomeActive — La escena está ACTIVA
```

**Verificación:**

- [ ] Los 5 mensajes de lanzamiento aparecen en la consola en el orden mostrado.
- [ ] Las transiciones Home → App producen los 4 mensajes esperados.
- [ ] Puedes identificar qué archivo (`AppDelegate` vs `SceneDelegate`) gestiona cada evento.

---

### Paso 3 — Crear la Estructura de Carpetas Estándar

**Objetivo:** Organizar el proyecto con la estructura de carpetas que seguiremos durante todo el curso, separando modelos, vistas, controladores y servicios.

**Instrucciones:**

1. En Xcode, haz clic derecho sobre la carpeta `TaskManagerApp` (la carpeta amarilla, no el proyecto azul) en el **Project Navigator**.

2. Selecciona **New Group** y crea las siguientes carpetas (grupos) una por una:

   | Carpeta | Propósito |
   |---|---|
   | `Models` | Structs y clases del modelo de datos |
   | `Views` | Archivos de Storyboard y vistas personalizadas |
   | `Controllers` | ViewControllers de la app |
   | `Services` | Protocolos y clases de lógica de negocio/repositorios |
   | `Resources` | Assets, archivos de configuración y recursos adicionales |
   | `Utilities` | Extensiones, helpers y funciones auxiliares |

3. Reorganiza los archivos existentes arrastrándolos a las carpetas correspondientes:
   - Mueve `ViewController.swift` → `Controllers/`
   - Mueve `Main.storyboard` → `Views/`
   - Mueve `LaunchScreen.storyboard` → `Views/`
   - Mueve `Assets.xcassets` → `Resources/`

   > **Importante:** Al arrastrar archivos en Xcode, asegúrate de que la opción "Move" (no "Copy") esté seleccionada si aparece un diálogo. Xcode actualiza las referencias automáticamente.

4. Verifica que la estructura en el Project Navigator se vea así:

   ```
   TaskManagerApp/
   ├── AppDelegate.swift
   ├── SceneDelegate.swift
   ├── Models/
   ├── Views/
   │   ├── Main.storyboard
   │   └── LaunchScreen.storyboard
   ├── Controllers/
   │   └── ViewController.swift
   ├── Services/
   ├── Resources/
   │   └── Assets.xcassets
   ├── Utilities/
   └── Info.plist
   ```

5. Compila el proyecto (`⌘ + B`) para verificar que la reorganización no rompió ninguna referencia.

**Salida esperada:**

- Build Succeeded (barra superior de Xcode muestra ✓).
- No hay errores rojos en el Project Navigator.

**Verificación:**

```bash
# Verifica la estructura de carpetas en el sistema de archivos
find ~/Developer/TaskManagerApp/TaskManagerApp -type d | head -20
# Debes ver las carpetas Models, Views, Controllers, Services, Resources, Utilities
```

---

### Paso 4 — Inicializar Git y Configurar el Repositorio Remoto

**Objetivo:** Crear un repositorio Git local con `.gitignore` apropiado para Xcode, establecer las ramas `main` y `develop`, y conectar con el repositorio remoto en GitHub.

**Instrucciones:**

1. Abre **Terminal** y navega al directorio del proyecto:

   ```bash
   cd ~/Developer/TaskManagerApp
   ```

2. Inicializa el repositorio Git:

   ```bash
   git init
   ```

3. Crea el archivo `.gitignore` con las exclusiones estándar para proyectos Xcode/Swift:

   ```bash
   cat > .gitignore << 'EOF'
   # Xcode
   build/
   DerivedData/
   *.xcodeproj/xcuserdata/
   *.xcworkspace/xcuserdata/
   *.xcodeproj/project.xcworkspace/xcshareddata/IDEWorkspaceChecks.plist

   # Swift Package Manager
   .build/
   Packages/

   # CocoaPods (si se usa en el futuro)
   Pods/

   # macOS
   .DS_Store
   *.swp
   *~

   # Archivos temporales
   *.playground/timeline.xctimeline
   *.playground/playground.xcworkspace
   EOF
   ```

4. Verifica que el `.gitignore` se creó correctamente:

   ```bash
   cat .gitignore
   ```

5. Agrega todos los archivos al staging area y realiza el commit inicial:

   ```bash
   git add .
   git status
   git commit -m "feat: proyecto base TaskManagerApp con estructura de carpetas estándar

   - Proyecto Xcode 15.4 con UIKit/Storyboard, deployment target iOS 17.0
   - Estructura de carpetas: Models, Views, Controllers, Services, Resources, Utilities
   - AppDelegate y SceneDelegate instrumentados con print() para ciclo de vida
   - .gitignore configurado para Xcode/Swift"
   ```

6. Renombra la rama principal a `main` (si no lo está ya):

   ```bash
   git branch -M main
   ```

7. Conecta con el repositorio remoto en GitHub. Ejecuta el comando correspondiente según tu método de autenticación:

   **Si usas SSH:**
   ```bash
   # Reemplaza "estudiante-unam" con tu nombre de usuario real de GitHub
   git remote add origin git@github.com:estudiante-unam/task-manager-ios.git
   ```

   **Si usas HTTPS:**
   ```bash
   # Reemplaza "estudiante-unam" con tu nombre de usuario real de GitHub
   git remote add origin https://github.com/estudiante-unam/task-manager-ios.git
   ```

   > **Nota:** Debes haber creado previamente el repositorio `task-manager-ios` en tu cuenta de GitHub (vacío, sin README ni .gitignore). Reemplaza `estudiante-unam` con tu nombre de usuario real de GitHub.

8. Realiza el push inicial:

   ```bash
   git push -u origin main
   ```

9. Crea la rama `develop` y haz push:

   ```bash
   git checkout -b develop
   git push -u origin develop
   ```

10. Verifica el estado del repositorio:

    ```bash
    git log --oneline --all --graph
    git remote -v
    git branch -a
    ```

**Salida esperada:**

```
* 1a2b3c4 (HEAD -> develop, origin/develop, origin/main, main) feat: proyecto base TaskManagerApp con estructura de carpetas estándar
```

```
origin  git@github.com:estudiante-unam/task-manager-ios.git (fetch)
origin  git@github.com:estudiante-unam/task-manager-ios.git (push)
```

```
* develop
  main
  remotes/origin/develop
  remotes/origin/main
```

**Verificación:**

- [ ] `git status` muestra "nothing to commit, working tree clean".
- [ ] `git branch` muestra `main` y `* develop` (estás en develop).
- [ ] El repositorio es visible en GitHub con el commit inicial.

---

### Paso 5 — Implementar el Modelo de Datos TaskItem

> **Nota:** En este curso el modelo se llama `TaskItem` (y no `Task`) para evitar el conflicto con `Swift.Task`, el tipo de Swift Concurrency que usarás en el Lab 04.

**Objetivo:** Crear el struct `TaskItem` con propiedades tipadas, un enum `Priority`, y aplicar sintaxis Swift 5.10 incluyendo opcionales, inicializadores y conformancia a protocolos estándar.

**Instrucciones:**

1. En Xcode, haz clic derecho sobre la carpeta **`Models`** en el Project Navigator.

2. Selecciona **New File…** → **Swift File** → **Next**.

3. Nombra el archivo `TaskItem.swift` y asegúrate de que el target `TaskManagerApp` esté marcado. Haz clic en **Create**.

4. Reemplaza el contenido de `TaskItem.swift` con el siguiente código:

   ```swift
   //
   //  TaskItem.swift
   //  TaskManagerApp
   //
   //  Laboratorio 01-00-01: Modelo de datos inicial
   //

   import Foundation

   // MARK: - Priority Enum

   /// Nivel de prioridad de una tarea.
   /// Conforma a `Int` para permitir ordenamiento y a `CaseIterable` para iterar sobre todos los casos.
   enum Priority: Int, CaseIterable, CustomStringConvertible {
       case low = 0
       case medium = 1
       case high = 2
       case critical = 3

       var description: String {
           switch self {
           case .low:
               return "Baja"
           case .medium:
               return "Media"
           case .high:
               return "Alta"
           case .critical:
               return "Crítica"
           }
       }

       /// Emoji representativo de la prioridad.
       var emoji: String {
           switch self {
           case .low:      return "🟢"
           case .medium:   return "🟡"
           case .high:     return "🟠"
           case .critical: return "🔴"
           }
       }
   }

   // MARK: - TaskItem Struct

   /// Modelo principal de tarea para TaskManagerApp.
   /// Usa `struct` (tipo por valor) siguiendo las mejores prácticas de Swift.
   struct TaskItem: Identifiable, CustomStringConvertible {

       // MARK: Properties

       let id: UUID
       var title: String
       var taskDescription: String?  // Opcional: no todas las tareas necesitan descripción
       var isCompleted: Bool
       var priority: Priority
       let createdAt: Date
       var dueDate: Date?            // Opcional: no todas las tareas tienen fecha límite

       // MARK: Initializer

       /// Inicializador con valores por defecto para facilitar la creación de tareas.
       /// - Parameters:
       ///   - title: Título de la tarea (obligatorio).
       ///   - taskDescription: Descripción detallada (opcional, por defecto `nil`).
       ///   - isCompleted: Estado de completitud (por defecto `false`).
       ///   - priority: Nivel de prioridad (por defecto `.medium`).
       ///   - dueDate: Fecha límite (opcional, por defecto `nil`).
       init(
           title: String,
           taskDescription: String? = nil,
           isCompleted: Bool = false,
           priority: Priority = .medium,
           dueDate: Date? = nil
       ) {
           self.id = UUID()
           self.title = title
           self.taskDescription = taskDescription
           self.isCompleted = isCompleted
           self.priority = priority
           self.createdAt = Date()
           self.dueDate = dueDate
       }

       // MARK: CustomStringConvertible

       var description: String {
           let status = isCompleted ? "✅" : "⬜️"
           let dueDateText: String
           if let dueDate = dueDate {
               let formatter = DateFormatter()
               formatter.dateStyle = .short
               formatter.timeStyle = .none
               formatter.locale = Locale(identifier: "es_MX")
               dueDateText = " | Vence: \(formatter.string(from: dueDate))"
           } else {
               dueDateText = ""
           }
           return "\(status) \(priority.emoji) [\(priority)] \(title)\(dueDateText)"
       }

       // MARK: Methods

       /// Verifica si la tarea está vencida.
       /// - Returns: `true` si la tarea tiene fecha límite y esta ya pasó; `false` en caso contrario.
       func isOverdue() -> Bool {
           guard let dueDate = dueDate else {
               return false
           }
           return !isCompleted && dueDate < Date()
       }

       /// Retorna una copia de la tarea con el estado de completitud invertido.
       /// Usa el patrón de inmutabilidad de structs en Swift.
       func toggleCompleted() -> TaskItem {
           var copy = self
           copy.isCompleted = !copy.isCompleted
           return copy
       }
   }
   ```

5. Compila el proyecto (`⌘ + B`) para verificar que no hay errores de sintaxis.

**Salida esperada:**

- Build Succeeded.
- El archivo `TaskItem.swift` aparece dentro de la carpeta `Models/` en el Project Navigator.

**Verificación:**

- [ ] El proyecto compila sin errores.
- [ ] El struct `TaskItem` tiene 7 propiedades (id, title, taskDescription, isCompleted, priority, createdAt, dueDate).
- [ ] El enum `Priority` tiene 4 casos y conforma a `Int`, `CaseIterable` y `CustomStringConvertible`.

---

### Paso 6 — Implementar el Protocolo TaskRepository y su Implementación en Memoria

**Objetivo:** Definir el protocolo `TaskRepository` como contrato de acceso a datos e implementar `InMemoryTaskRepository` con operaciones CRUD y filtrado usando closures.

**Instrucciones:**

1. Crea un nuevo archivo Swift en la carpeta **`Services`**: clic derecho → **New File…** → **Swift File** → nombre: `TaskRepository.swift`.

2. Reemplaza el contenido con:

   ```swift
   //
   //  TaskRepository.swift
   //  TaskManagerApp
   //
   //  Laboratorio 01-00-01: Protocolo de repositorio y filtrado con closures
   //

   import Foundation

   // MARK: - TaskRepository Protocol

   /// Contrato que define las operaciones de acceso a datos para tareas.
   /// Cualquier implementación (en memoria, Core Data, API) debe conformar a este protocolo.
   protocol TaskRepository {

       /// Retorna todas las tareas almacenadas.
       func fetchAll() -> [TaskItem]

       /// Busca una tarea por su identificador único.
       /// - Parameter id: UUID de la tarea a buscar.
       /// - Returns: La tarea si existe, `nil` en caso contrario.
       func fetchById(_ id: UUID) -> TaskItem?

       /// Agrega una nueva tarea al repositorio.
       /// - Parameter task: La tarea a agregar.
       mutating func add(_ task: TaskItem)

       /// Actualiza una tarea existente.
       /// - Parameter task: La tarea con los datos actualizados (se identifica por `id`).
       /// - Returns: `true` si la tarea fue encontrada y actualizada, `false` en caso contrario.
       @discardableResult
       mutating func update(_ task: TaskItem) -> Bool

       /// Elimina una tarea por su identificador.
       /// - Parameter id: UUID de la tarea a eliminar.
       /// - Returns: `true` si la tarea fue encontrada y eliminada, `false` en caso contrario.
       @discardableResult
       mutating func delete(byId id: UUID) -> Bool

       /// Filtra tareas usando un closure como predicado.
       /// - Parameter predicate: Closure que recibe una `TaskItem` y retorna `Bool`.
       /// - Returns: Array de tareas que cumplen el predicado.
       func filter(where predicate: (TaskItem) -> Bool) -> [TaskItem]
   }

   // MARK: - InMemoryTaskRepository

   /// Implementación en memoria del repositorio de tareas.
   /// Usa un array interno como almacenamiento temporal (los datos se pierden al cerrar la app).
   struct InMemoryTaskRepository: TaskRepository {

       // MARK: Properties

       private var tasks: [TaskItem] = []

       /// Número de tareas almacenadas.
       var count: Int {
           return tasks.count
       }

       // MARK: TaskRepository Conformance

       func fetchAll() -> [TaskItem] {
           return tasks
       }

       func fetchById(_ id: UUID) -> TaskItem? {
           return tasks.first { $0.id == id }
       }

       mutating func add(_ task: TaskItem) {
           tasks.append(task)
           print("📝 Tarea agregada: \(task.title) | Total: \(tasks.count)")
       }

       @discardableResult
       mutating func update(_ task: TaskItem) -> Bool {
           guard let index = tasks.firstIndex(where: { $0.id == task.id }) else {
               print("⚠️ Tarea no encontrada para actualizar: \(task.id)")
               return false
           }
           tasks[index] = task
           print("🔄 Tarea actualizada: \(task.title)")
           return true
       }

       @discardableResult
       mutating func delete(byId id: UUID) -> Bool {
           guard let index = tasks.firstIndex(where: { $0.id == id }) else {
               print("⚠️ Tarea no encontrada para eliminar: \(id)")
               return false
           }
           let removed = tasks.remove(at: index)
           print("🗑️ Tarea eliminada: \(removed.title) | Total: \(tasks.count)")
           return true
       }

       func filter(where predicate: (TaskItem) -> Bool) -> [TaskItem] {
           return tasks.filter(predicate)
       }

       // MARK: Convenience Methods

       /// Retorna tareas pendientes (no completadas).
       func pendingTasks() -> [TaskItem] {
           return filter { !$0.isCompleted }
       }

       /// Retorna tareas completadas.
       func completedTasks() -> [TaskItem] {
           return filter { $0.isCompleted }
       }

       /// Retorna tareas filtradas por prioridad.
       /// - Parameter priority: Nivel de prioridad a filtrar.
       func tasks(withPriority priority: Priority) -> [TaskItem] {
           return filter { $0.priority == priority }
       }

       /// Retorna tareas ordenadas por prioridad (de mayor a menor).
       func tasksSortedByPriority() -> [TaskItem] {
           return tasks.sorted { $0.priority.rawValue > $1.priority.rawValue }
       }

       /// Retorna tareas vencidas.
       func overdueTasks() -> [TaskItem] {
           return filter { $0.isOverdue() }
       }
   }
   ```

3. Compila el proyecto (`⌘ + B`).

**Salida esperada:**

- Build Succeeded.
- El archivo `TaskRepository.swift` aparece dentro de `Services/`.

**Verificación:**

- [ ] El protocolo `TaskRepository` define 6 métodos (fetchAll, fetchById, add, update, delete, filter).
- [ ] `InMemoryTaskRepository` implementa todos los métodos del protocolo.
- [ ] Los métodos de conveniencia usan closures de forma concisa.

---

### Paso 7 — Probar el Modelo de Datos en ViewController

**Objetivo:** Verificar el funcionamiento del modelo de datos y el repositorio ejecutando pruebas directamente en `viewDidLoad()`, observando los resultados en la consola de Xcode.

**Instrucciones:**

1. Abre `Controllers/ViewController.swift` y reemplaza su contenido con:

   ```swift
   //
   //  ViewController.swift
   //  TaskManagerApp
   //
   //  Laboratorio 01-00-01: Prueba del modelo de datos
   //

   import UIKit

   class ViewController: UIViewController {

       override func viewDidLoad() {
           super.viewDidLoad()
           view.backgroundColor = .systemBackground

           print("\n" + String(repeating: "=", count: 60))
           print("🧪 INICIO — Pruebas del Modelo de Datos TaskManagerApp")
           print(String(repeating: "=", count: 60) + "\n")

           testTaskCreation()
           testRepository()
           testClosureFiltering()
           testPriorityEnum()

           print("\n" + String(repeating: "=", count: 60))
           print("✅ FIN — Todas las pruebas ejecutadas correctamente")
           print(String(repeating: "=", count: 60) + "\n")
       }

       // MARK: - Test Methods

       private func testTaskCreation() {
           print("--- Test 1: Creación de Tasks ---")

           // Tarea con valores por defecto
           let task1 = TaskItem(title: "Estudiar Swift")
           print("Tarea 1: \(task1)")
           print("  ID: \(task1.id)")
           print("  Completada: \(task1.isCompleted)")
           print("  Descripción: \(task1.taskDescription ?? "Sin descripción")")

           // Tarea con todos los parámetros
           let tomorrow = Calendar.current.date(byAdding: .day, value: 1, to: Date())!
           let task2 = TaskItem(
               title: "Entregar laboratorio",
               taskDescription: "Lab 01-00-01 del curso iOS",
               isCompleted: false,
               priority: .high,
               dueDate: tomorrow
           )
           print("Tarea 2: \(task2)")
           print("  Vencida: \(task2.isOverdue())")

           // Tarea vencida (fecha en el pasado)
           let yesterday = Calendar.current.date(byAdding: .day, value: -1, to: Date())!
           let task3 = TaskItem(
               title: "Tarea vencida de ejemplo",
               priority: .critical,
               dueDate: yesterday
           )
           print("Tarea 3: \(task3)")
           print("  Vencida: \(task3.isOverdue())")

           // Toggle de completitud (inmutabilidad de struct)
           let task1Completed = task1.toggleCompleted()
           print("\nToggle completitud:")
           print("  Original: \(task1.isCompleted) → Copia: \(task1Completed.isCompleted)")
           print("  (El original NO cambió — tipos por valor)")
           print("")
       }

       private func testRepository() {
           print("--- Test 2: Repositorio CRUD ---")

           var repo = InMemoryTaskRepository()

           // CREATE
           let task1 = TaskItem(title: "Configurar Xcode", priority: .high)
           let task2 = TaskItem(title: "Crear .gitignore", priority: .medium)
           let task3 = TaskItem(title: "Implementar modelo TaskItem", priority: .high)
           let task4 = TaskItem(title: "Revisar documentación", priority: .low)
           let task5 = TaskItem(
               title: "Entregar proyecto",
               priority: .critical,
               dueDate: Calendar.current.date(byAdding: .day, value: 3, to: Date())
           )

           repo.add(task1)
           repo.add(task2)
           repo.add(task3)
           repo.add(task4)
           repo.add(task5)

           print("\nTotal de tareas: \(repo.count)")

           // READ
           print("\nTodas las tareas:")
           for task in repo.fetchAll() {
               print("  \(task)")
           }

           // READ by ID
           if let found = repo.fetchById(task3.id) {
               print("\nBúsqueda por ID exitosa: \(found.title)")
           }

           // UPDATE
           var taskToUpdate = task2
           taskToUpdate.isCompleted = true
           let updated = repo.update(taskToUpdate)
           print("Actualización exitosa: \(updated)")

           // DELETE
           let deleted = repo.delete(byId: task4.id)
           print("Eliminación exitosa: \(deleted)")
           print("Total después de eliminar: \(repo.count)")
           print("")
       }

       private func testClosureFiltering() {
           print("--- Test 3: Filtrado con Closures ---")

           var repo = InMemoryTaskRepository()

           repo.add(TaskItem(title: "Tarea baja completada", isCompleted: true, priority: .low))
           repo.add(TaskItem(title: "Tarea media pendiente", priority: .medium))
           repo.add(TaskItem(title: "Tarea alta pendiente", priority: .high))
           repo.add(TaskItem(title: "Tarea crítica completada", isCompleted: true, priority: .critical))
           repo.add(TaskItem(title: "Tarea alta completada", isCompleted: true, priority: .high))

           // Filtrado con closure explícito
           let pending = repo.filter(where: { (task: TaskItem) -> Bool in
               return !task.isCompleted
           })
           print("\nPendientes (closure explícito): \(pending.count)")
           pending.forEach { print("  \($0)") }

           // Filtrado con closure simplificado (trailing closure + shorthand)
           let highPriority = repo.filter { $0.priority.rawValue >= Priority.high.rawValue }
           print("\nAlta prioridad o superior: \(highPriority.count)")
           highPriority.forEach { print("  \($0)") }

           // Usando métodos de conveniencia
           let completed = repo.completedTasks()
           print("\nCompletadas: \(completed.count)")
           completed.forEach { print("  \($0)") }

           // Ordenamiento por prioridad
           let sorted = repo.tasksSortedByPriority()
           print("\nOrdenadas por prioridad (mayor a menor):")
           sorted.forEach { print("  \($0)") }
           print("")
       }

       private func testPriorityEnum() {
           print("--- Test 4: Enum Priority ---")

           // Iterar sobre todos los casos (CaseIterable)
           print("\nTodos los niveles de prioridad:")
           for priority in Priority.allCases {
               print("  \(priority.emoji) \(priority) (rawValue: \(priority.rawValue))")
           }

           // Switch exhaustivo
           let testPriority: Priority = .high
           let recommendation: String
           switch testPriority {
           case .low:
               recommendation = "Puede esperar"
           case .medium:
               recommendation = "Planificar esta semana"
           case .high:
               recommendation = "Atender hoy"
           case .critical:
               recommendation = "¡Atender inmediatamente!"
           }
           print("\nPrioridad \(testPriority): \(recommendation)")
           print("")
       }
   }
   ```

3. Ejecuta la app (`⌘ + R`) y observa la consola de Xcode.

**Salida esperada en consola (fragmento):**

```
============================================================
🧪 INICIO — Pruebas del Modelo de Datos TaskManagerApp
============================================================

--- Test 1: Creación de Tasks ---
Tarea 1: ⬜️ 🟡 [Media] Estudiar Swift
  ID: A1B2C3D4-...
  Completada: false
  Descripción: Sin descripción
Tarea 2: ⬜️ 🟠 [Alta] Entregar laboratorio | Vence: 25/01/2025
  Vencida: false
Tarea 3: ⬜️ 🔴 [Crítica] Tarea vencida de ejemplo | Vence: 23/01/2025
  Vencida: true

Toggle completitud:
  Original: false → Copia: true
  (El original NO cambió — tipos por valor)

--- Test 2: Repositorio CRUD ---
📝 Tarea agregada: Configurar Xcode | Total: 1
📝 Tarea agregada: Crear .gitignore | Total: 2
📝 Tarea agregada: Implementar modelo TaskItem | Total: 3
📝 Tarea agregada: Revisar documentación | Total: 4
📝 Tarea agregada: Entregar proyecto | Total: 5

Total de tareas: 5
...

============================================================
✅ FIN — Todas las pruebas ejecutadas correctamente
============================================================
```

4. Detén la ejecución (`⌘ + .`).

**Verificación:**

- [ ] Los 4 tests se ejecutan sin errores.
- [ ] La salida de consola muestra los emojis y formato esperado.
- [ ] El toggle de completitud demuestra la inmutabilidad de structs (el original no cambia).
- [ ] El filtrado con closures produce los resultados correctos.

---

### Paso 8 — Depuración con Breakpoints y LLDB

**Objetivo:** Aprender a usar breakpoints, la consola LLDB y la vista de variables de Xcode para inspeccionar el estado de la aplicación en tiempo de ejecución.

**Instrucciones:**

1. Abre `Controllers/ViewController.swift`.

2. **Establece un breakpoint** en la línea que contiene `testRepository()` dentro de `viewDidLoad()`:
   - Haz clic en el número de línea (margen izquierdo del editor) junto a `testRepository()`. Aparecerá un indicador azul.

3. **Establece un segundo breakpoint** dentro del método `testRepository()`, en la línea `repo.add(task1)`.

4. Ejecuta la app (`⌘ + R`). La ejecución se detendrá en el primer breakpoint.

5. **Inspecciona el estado** usando las herramientas de depuración:

   a. **Vista de Variables** (panel inferior izquierdo): Expande `self` para ver las propiedades del ViewController.

   b. **Consola LLDB** (panel inferior derecho): Escribe los siguientes comandos:

   ```lldb
   po self
   ```

   Esto imprime la descripción del objeto actual.

   ```lldb
   expression print("Hola desde LLDB")
   ```

   Esto ejecuta código Swift arbitrario desde el debugger.

6. Presiona **Continue** (`⌃ + ⌘ + Y` o el botón ▶️ en la barra de depuración) para avanzar al siguiente breakpoint.

7. Cuando la ejecución se detenga en `repo.add(task1)`, inspecciona las variables locales:

   ```lldb
   po task1
   po task1.title
   po task1.priority
   po repo.count
   ```

8. Usa **Step Over** (`F6`) para ejecutar línea por línea y observar cómo cambia `repo.count` después de cada `add()`.

9. Usa **Step Into** (`F7`) en una llamada a `repo.add(task2)` para entrar al método `add()` de `InMemoryTaskRepository` y ver su implementación interna.

10. **Elimina los breakpoints** arrastrándolos fuera del margen (o clic derecho → Delete Breakpoint).

11. Ejecuta la app una vez más (`⌘ + R`) para verificar que corre sin interrupciones.

12. Detén la ejecución (`⌘ + .`).

**Salida esperada:**

- Al detenerse en el breakpoint, el panel de variables muestra las propiedades del ViewController.
- Los comandos `po` en LLDB muestran los valores de las variables.
- Step Over permite ver el incremento de `repo.count` después de cada `add()`.

**Verificación:**

- [ ] Sabes establecer y eliminar breakpoints haciendo clic en el margen del editor.
- [ ] Puedes usar `po` en LLDB para inspeccionar variables.
- [ ] Puedes usar Step Over (F6) y Step Into (F7) para navegar la ejecución.
- [ ] La vista de variables muestra el estado actual de los objetos.

---

### Paso 9 — Commit Final y Push a GitHub

**Objetivo:** Confirmar todos los cambios del laboratorio en Git y sincronizar con el repositorio remoto.

**Instrucciones:**

1. En Terminal, navega al proyecto y verifica los cambios:

   ```bash
   cd ~/Developer/TaskManagerApp
   git status
   ```

2. Revisa los archivos modificados y nuevos:

   ```bash
   git diff --stat
   ```

3. Agrega todos los cambios y realiza el commit:

   ```bash
   git add .
   git commit -m "feat: modelo de datos TaskItem, protocolo TaskRepository e InMemoryTaskRepository

   - Struct TaskItem con propiedades tipadas, opcionales y métodos (isOverdue, toggleCompleted)
   - Enum Priority con CaseIterable, CustomStringConvertible y rawValue Int
   - Protocolo TaskRepository con operaciones CRUD y filtrado con closures
   - InMemoryTaskRepository con métodos de conveniencia (pendingTasks, completedTasks, etc.)
   - ViewController con pruebas del modelo de datos en consola
   - AppDelegate/SceneDelegate instrumentados para observar ciclo de vida"
   ```

4. Haz push a la rama `develop`:

   ```bash
   git push origin develop
   ```

5. Verifica el log de commits:

   ```bash
   git log --oneline --all --graph
   ```

**Salida esperada:**

```
* a1b2c3d (HEAD -> develop, origin/develop) feat: modelo de datos TaskItem, protocolo TaskRepository e InMemoryTaskRepository
* 1a2b3c4 (origin/main, main) feat: proyecto base TaskManagerApp con estructura de carpetas estándar
```

**Verificación:**

- [ ] `git log` muestra 2 commits en `develop` y 1 en `main`.
- [ ] Los cambios son visibles en GitHub en la rama `develop`.
- [ ] `git status` muestra "nothing to commit, working tree clean".

---

## Reto Opcional (KMP) — Saludo desde el Módulo Compartido

> Este reto es **opcional** y no forma parte de la evaluación. Requiere **Android Studio** con el plugin **Kotlin Multiplatform**, además de Xcode.

**Objetivo:** Crear un proyecto Kotlin Multiplatform y mostrar en la app iOS un texto generado por el módulo `shared`.

**Instrucciones:**

1. En Android Studio, crea un proyecto con el asistente **Kotlin Multiplatform** y marca los destinos Android e iOS.
2. Revisa la estructura generada: `shared/src/commonMain` (código común), `androidApp` e `iosApp`.
3. Abre `iosApp/iosApp.xcodeproj` en Xcode y ejecuta la app en el simulador iPhone 15 Pro.
4. En la vista principal de `iosApp`, importa el framework con `import Shared` y muestra el resultado de `Greeting().greet()`.

**Resultado esperado:** la app iOS muestra el saludo que devuelve la clase `Greeting` del módulo `shared`, lo que confirma que el código Kotlin se usa desde Swift.

---

## Resumen del Laboratorio

En este laboratorio has completado las siguientes actividades:

| Paso | Actividad | Resultado |
|---|---|---|
| 1 | Creación del proyecto Xcode | `TaskManagerApp` con UIKit/Storyboard, iOS 17.0 |
| 2 | Exploración del ciclo de vida | AppDelegate y SceneDelegate instrumentados con `print()` |
| 3 | Estructura de carpetas | Models, Views, Controllers, Services, Resources, Utilities |
| 4 | Configuración de Git | Repositorio local + remoto con ramas `main`/`develop` |
| 5 | Modelo de datos | Struct `TaskItem` + Enum `Priority` con sintaxis Swift 5.10 |
| 6 | Protocolo y repositorio | `TaskRepository` + `InMemoryTaskRepository` con closures |
| 7 | Pruebas en consola | CRUD, filtrado y validación del modelo en ViewController |
| 8 | Depuración | Breakpoints, LLDB y vista de variables en Xcode |
| 9 | Commit y push | 2 commits en `develop`, sincronizado con GitHub |

---

## Criterios de Evaluación

| Criterio | Ponderación |
|---|---|
| El proyecto compila y ejecuta sin errores en el simulador iPhone 15 Pro | 20% |
| La estructura de carpetas sigue el estándar del curso (6 carpetas) | 10% |
| Los mensajes del ciclo de vida aparecen correctamente en consola | 15% |
| El repositorio Git tiene 2 commits, ramas `main`/`develop` y está en GitHub | 15% |
| El struct `TaskItem` y enum `Priority` están correctamente implementados | 15% |
| El protocolo `TaskRepository` y `InMemoryTaskRepository` funcionan con CRUD y closures | 15% |
| Se demuestra el uso de breakpoints y LLDB para inspeccionar variables | 10% |

---

## Solución de Problemas Comunes

| Problema | Causa Probable | Solución |
|---|---|---|
| "No such module 'UIKit'" | Target incorrecto o archivo no agregado al target | Selecciona el archivo → File Inspector → marca `TaskManagerApp` en Target Membership |
| El simulador no aparece | Runtime de iOS 17.5 no descargado | Xcode → Settings → Platforms → descarga iOS 17.5 |
| `git push` falla con "Permission denied" | SSH no configurado o clave no agregada a GitHub | Verifica con `ssh -T git@github.com`; si falla, configura SSH keys |
| `git push` falla con "repository not found" | El repositorio no existe en GitHub o el nombre es incorrecto | Crea el repositorio `task-manager-ios` en GitHub; verifica la URL con `git remote -v` |
| Build falla después de mover archivos | Referencias rotas en el proyecto | Elimina el archivo del proyecto (Remove Reference), luego agrégalo de nuevo con Add Files |
| La consola no muestra mensajes de `print()` | La consola está filtrada o cerrada | Abre la consola con `⌘ + Shift + C`; verifica que no hay filtros activos |
| Breakpoint no se detiene | El breakpoint está deshabilitado (gris) | Haz clic en el breakpoint para habilitarlo (azul); verifica que la línea se ejecuta |

---

## Próximo Laboratorio

En el **Lab 01-00-02** extenderás el modelo de datos con validaciones, implementarás extensiones de Swift para formateo de fechas y cadenas, y crearás tests unitarios básicos para verificar el comportamiento del repositorio.