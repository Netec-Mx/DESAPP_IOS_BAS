# Capa de Red con URLSession, Codable y Swift Concurrency

| Campo | Detalle |
|---|---|
| **Duración** | 252 minutos (~4 h 12 min) |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |

---

## Descripción General

En esta práctica incremental añadirás la **capa de networking** al proyecto **TaskManagerApp** construido en los Labs 01–03. Implementarás un servicio de red con `URLSession` que consume la API pública JSONPlaceholder para obtener y crear tareas remotas, parsearás las respuestas JSON con `Codable`, gestionarás la concurrencia con `async/await` y `@MainActor`, y sincronizarás los datos remotos con el repositorio local SwiftData. Al finalizar, la aplicación mostrará estados de carga, éxito y error tanto en la interfaz SwiftUI como en UIKit.

---

## Objetivos de Aprendizaje

- [ ] Implementar un servicio de red con `URLSession` que realice peticiones GET y POST a JSONPlaceholder.
- [ ] Parsear respuestas JSON usando `Codable` (`Decodable`/`Encodable`) y mapear datos remotos al modelo de dominio `TaskItem`, que se persiste en SwiftData como `SDTaskItem`.
- [ ] Gestionar la concurrencia con `async/await`, `Task` y `@MainActor` para actualizar la UI de forma segura.
- [ ] Implementar manejo de errores de red con un `enum NetworkError` y mostrar estados de carga/éxito/error en la interfaz.
- [ ] Sincronizar tareas remotas con el repositorio SwiftData local y confirmar los cambios con Git.

---

## Prerrequisitos

### Conocimiento Previo

| Requisito | Detalle |
|---|---|
| Lab 03 completado | Persistencia con SwiftData (`SwiftDataTaskRepository`) funcional |
| Rama `develop` actualizada | Commits de Labs 01, 02 y 03 en GitHub |
| HTTP básico | Métodos GET/POST, códigos de estado, formato JSON |
| Swift Concurrency básico | Comprensión conceptual de `async/await` |

### Acceso Requerido

| Recurso | Verificación |
|---|---|
| Conexión a Internet | `curl -s https://jsonplaceholder.typicode.com/todos/1` devuelve JSON |
| Repositorio GitHub | `task-manager-ios` con rama `develop` |

---

## Entorno del Laboratorio

### Hardware y Software

| Componente | Requisito |
|---|---|
| macOS | Ventura 13.5+ o Sonoma 14.5 |
| Xcode | 15.4 con iOS SDK 17.5 |
| Swift | 5.10 |
| Simulador | iPhone 15 Pro – iOS 17.5 |
| Git | 2.45.2 |
| RAM | 16 GB mínimo |
| Disco libre | 50 GB mínimo |

### Preparación Inicial del Entorno

Abre **Terminal** y verifica la conectividad con la API y el estado del repositorio:

```bash
# Verificar conectividad con JSONPlaceholder
curl -s https://jsonplaceholder.typicode.com/todos/1 | python3 -m json.tool
```

**Salida esperada:**

```json
{
    "userId": 1,
    "id": 1,
    "title": "delectus aut autem",
    "completed": false
}
```

```bash
# Navegar al proyecto y verificar rama
cd ~/Developer/TaskManagerApp
git branch --show-current
```

**Salida esperada:**

```
develop
```

```bash
# Verificar que no hay cambios pendientes
git status
```

**Salida esperada:**

```
On branch develop
nothing to commit, working tree clean
```

---

## Instrucciones Paso a Paso

### Paso 1 — Crear la rama de feature y la estructura de carpetas (10 min)

**Objetivo:** Crear la rama `feature/networking` y organizar la estructura de directorios para la capa de red.

**Instrucciones:**

1. Abre **Terminal** y navega al directorio del proyecto:

```bash
cd ~/Developer/TaskManagerApp
```

2. Crea y cambia a la nueva rama de feature:

```bash
git checkout -b feature/networking
```

3. Crea la estructura de carpetas para los componentes de red:

```bash
mkdir -p TaskManagerApp/Services/Network
mkdir -p TaskManagerApp/ViewModels
```

4. Verifica la estructura creada:

```bash
find TaskManagerApp/Services -type d
```

**Salida esperada:**

```
TaskManagerApp/Services
TaskManagerApp/Services/Network
```

5. Abre el proyecto en Xcode:

```bash
open TaskManagerApp.xcodeproj
```

6. En el **Project Navigator** de Xcode, haz clic derecho sobre el grupo `TaskManagerApp` → **New Group** y nómbralo `Services`. Dentro de `Services`, crea otro grupo llamado `Network`. Si ya existe un grupo `ViewModels`, úsalo; de lo contrario, crea uno al mismo nivel que `Services`.

> **Nota:** Es importante que los grupos en Xcode correspondan a las carpetas físicas creadas en el paso 3. Al crear los grupos, verifica en el **File Inspector** (panel derecho) que la ruta apunta a la carpeta correcta.

**Verificación:**

```bash
git checkout feature/networking 2>&1 | grep -c "Already on"
```

Debe devolver `1` (ya estás en la rama).

---

### Paso 2 — Definir el modelo remoto `RemoteTask` (30 min)

**Objetivo:** Crear el struct `RemoteTask` que mapea la respuesta JSON de JSONPlaceholder y la función de conversión a `TaskItem`.

**Instrucciones:**

1. En Xcode, haz clic derecho sobre el grupo `Services/Network` → **New File…** → **Swift File** → nómbralo `RemoteTask.swift`. Asegúrate de que el target `TaskManagerApp` esté seleccionado.

2. Reemplaza el contenido del archivo con lo siguiente:

```swift
//
//  RemoteTask.swift
//  TaskManagerApp
//
//  Modelo que representa una tarea remota de JSONPlaceholder.
//  Endpoint: GET https://jsonplaceholder.typicode.com/todos
//

import Foundation

/// Estructura que mapea la respuesta JSON del endpoint /todos de JSONPlaceholder.
///
/// Ejemplo de respuesta JSON:
/// ```json
/// {
///     "userId": 1,
///     "id": 1,
///     "title": "delectus aut autem",
///     "completed": false
/// }
/// ```
struct RemoteTask: Decodable, Identifiable, Sendable {
    let userId: Int
    let id: Int
    let title: String
    let completed: Bool
}
```

3. Ahora crea la extensión con la función de conversión. En el mismo archivo, añade debajo del struct:

```swift
// MARK: - Conversión a TaskItem (modelo de dominio)

extension RemoteTask {
    /// Convierte un `RemoteTask` remoto al modelo de dominio `TaskItem`.
    ///
    /// - Returns: Una instancia de `TaskItem` con los datos mapeados.
    ///
    /// Mapeo de campos:
    /// - `title` → `TaskItem.title`
    /// - `completed` → `TaskItem.isCompleted`
    /// - Se genera una `taskDescription` a partir del `userId` y `id` remoto.
    /// - La `dueDate` se establece a 7 días desde la fecha actual.
    /// - La `priority` se asigna como `.medium` por defecto.
    func toTaskItem() -> TaskItem {
        // Mismo orden de argumentos que el inicializador del Lab 01:
        // title, taskDescription, isCompleted, priority, dueDate
        let item = TaskItem(
            title: title,
            taskDescription: "Importada de JSONPlaceholder (userId: \(userId), remoteId: \(id))",
            isCompleted: completed,
            priority: .medium,
            dueDate: Calendar.current.date(byAdding: .day, value: 7, to: Date())
        )
        return item
    }
}
```

> **Importante:** Desde el Lab 03 hay dos modelos: `TaskItem` (struct de dominio, la usan la red y los repositorios) y `SDTaskItem` (clase `@Model` de SwiftData). `RemoteTask` convierte a `TaskItem`; para guardar en SwiftData se usa `SDTaskItem(from:)`. El inicializador de `TaskItem` del Lab 01 ya acepta `priority` y `dueDate`; en Swift los argumentos con valor por defecto se pueden omitir, pero los que se pasan deben ir en el orden declarado (`title`, `taskDescription`, `isCompleted`, `priority`, `dueDate`).

4. Crea un segundo archivo para el modelo de respuesta POST. Haz clic derecho sobre `Services/Network` → **New File…** → **Swift File** → nómbralo `RemoteTaskRequest.swift`:

```swift
//
//  RemoteTaskRequest.swift
//  TaskManagerApp
//
//  Modelo Encodable para enviar tareas al servidor via POST.
//

import Foundation

/// Estructura que representa el cuerpo de una petición POST a /todos.
///
/// JSONPlaceholder espera este formato:
/// ```json
/// {
///     "title": "Nueva tarea",
///     "completed": false,
///     "userId": 1
/// }
/// ```
struct RemoteTaskRequest: Encodable, Sendable {
    let title: String
    let completed: Bool
    let userId: Int
    
    /// Crea un `RemoteTaskRequest` a partir de un `TaskItem` local.
    ///
    /// - Parameter taskItem: La tarea del modelo de dominio.
    init(from taskItem: TaskItem) {
        self.title = taskItem.title
        self.completed = taskItem.isCompleted
        self.userId = 1 // Usuario fijo para JSONPlaceholder
    }
}
```

5. Compila el proyecto con **⌘B** para verificar que no hay errores.

**Salida esperada:** Build Succeeded sin errores ni warnings.

**Verificación:**

Abre el archivo `RemoteTask.swift` y confirma que Xcode no muestra errores de compilación. Si ves un error de tipo `Cannot find type 'TaskItem' in scope`, verifica que el archivo del modelo `TaskItem` pertenece al mismo target `TaskManagerApp`.

---

### Paso 3 — Implementar el enum `NetworkError` (15 min)

**Objetivo:** Crear un enum de errores de red tipado que permita un manejo de errores claro y descriptivo.

**Instrucciones:**

1. En el grupo `Services/Network`, crea un nuevo archivo Swift llamado `NetworkError.swift`.

2. Añade el siguiente contenido:

```swift
//
//  NetworkError.swift
//  TaskManagerApp
//
//  Enum que define los posibles errores de la capa de red.
//

import Foundation

/// Errores específicos de la capa de networking de TaskManagerApp.
enum NetworkError: LocalizedError, Equatable {
    /// La URL proporcionada no es válida.
    case invalidURL
    
    /// El servidor no devolvió datos.
    case noData
    
    /// Error al decodificar la respuesta JSON.
    case decodingError(String)
    
    /// El servidor devolvió un código de estado HTTP de error.
    case serverError(Int)
    
    /// No hay conexión de red disponible.
    case networkUnavailable
    
    /// Descripción legible del error para mostrar en la UI.
    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "La URL de la petición no es válida."
        case .noData:
            return "El servidor no devolvió datos."
        case .decodingError(let detail):
            return "Error al procesar la respuesta: \(detail)"
        case .serverError(let code):
            return "Error del servidor (código \(code))."
        case .networkUnavailable:
            return "No hay conexión a Internet disponible."
        }
    }
    
    // Conformancia Equatable para testing
    static func == (lhs: NetworkError, rhs: NetworkError) -> Bool {
        switch (lhs, rhs) {
        case (.invalidURL, .invalidURL),
             (.noData, .noData),
             (.networkUnavailable, .networkUnavailable):
            return true
        case (.decodingError(let a), .decodingError(let b)):
            return a == b
        case (.serverError(let a), .serverError(let b)):
            return a == b
        default:
            return false
        }
    }
}
```

3. Compila con **⌘B**.

**Salida esperada:** Build Succeeded.

**Verificación:** El enum `NetworkError` aparece sin errores en el Project Navigator dentro de `Services/Network/`.

---

### Paso 4 — Implementar `TaskNetworkService` con `async/await` (90 min)

**Objetivo:** Crear la clase principal del servicio de red con métodos para obtener tareas (GET) y crear tareas (POST) usando `URLSession` con Swift Concurrency.

**Instrucciones:**

1. En el grupo `Services/Network`, crea un nuevo archivo Swift llamado `TaskNetworkService.swift`.

2. Añade la definición del protocolo y la clase:

```swift
//
//  TaskNetworkService.swift
//  TaskManagerApp
//
//  Servicio de red para comunicación con JSONPlaceholder API.
//

import Foundation

// MARK: - Protocolo del servicio de red

/// Protocolo que define las operaciones de red para tareas.
/// Facilita la inyección de dependencias y el testing con mocks.
protocol TaskNetworkServiceProtocol: Sendable {
    /// Obtiene una lista de tareas remotas del servidor.
    /// - Returns: Array de `RemoteTask` decodificados.
    /// - Throws: `NetworkError` si la petición falla.
    func fetchRemoteTasks() async throws -> [RemoteTask]
    
    /// Crea una tarea en el servidor remoto.
    /// - Parameter task: El `TaskItem` local a enviar.
    /// - Returns: El `RemoteTask` devuelto por el servidor con el ID asignado.
    /// - Throws: `NetworkError` si la petición falla.
    func createRemoteTask(_ task: TaskItem) async throws -> RemoteTask
}
```

3. Ahora implementa la clase concreta debajo del protocolo en el mismo archivo:

```swift
// MARK: - Implementación del servicio de red

/// Servicio de red que consume la API de JSONPlaceholder.
///
/// Usa `URLSession.shared` con `async/await` para realizar peticiones HTTP.
/// Todos los métodos lanzan `NetworkError` en caso de fallo.
final class TaskNetworkService: TaskNetworkServiceProtocol {
    
    // MARK: - Propiedades
    
    /// URL base de la API de JSONPlaceholder.
    private let baseURL = "https://jsonplaceholder.typicode.com"
    
    /// Sesión de URLSession utilizada para las peticiones.
    /// Se inyecta para facilitar testing.
    private let session: URLSession
    
    // MARK: - Inicialización
    
    /// Inicializa el servicio con una sesión de URLSession.
    /// - Parameter session: Sesión a utilizar. Por defecto `.shared`.
    init(session: URLSession = .shared) {
        self.session = session
    }
    
    // MARK: - GET /todos
    
    func fetchRemoteTasks() async throws -> [RemoteTask] {
        // 1. Construir la URL con query parameter para limitar resultados
        guard let url = URL(string: "\(baseURL)/todos?_limit=20") else {
            throw NetworkError.invalidURL
        }
        
        // 2. Crear el URLRequest configurado para GET
        var request = URLRequest(url: url)
        request.httpMethod = "GET"
        request.setValue("application/json", forHTTPHeaderField: "Accept")
        request.timeoutInterval = 15
        
        // 3. Ejecutar la petición con async/await
        let data: Data
        let response: URLResponse
        
        do {
            (data, response) = try await session.data(for: request)
        } catch let urlError as URLError {
            // Detectar errores de conectividad específicos
            if urlError.code == .notConnectedToInternet ||
               urlError.code == .networkConnectionLost {
                throw NetworkError.networkUnavailable
            }
            throw NetworkError.networkUnavailable
        }
        
        // 4. Validar la respuesta HTTP
        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.noData
        }
        
        // 5. Verificar código de estado exitoso (200-299)
        guard (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.serverError(httpResponse.statusCode)
        }
        
        // 6. Decodificar el JSON a [RemoteTask]
        do {
            let decoder = JSONDecoder()
            let remoteTasks = try decoder.decode([RemoteTask].self, from: data)
            return remoteTasks
        } catch {
            throw NetworkError.decodingError(error.localizedDescription)
        }
    }
    
    // MARK: - POST /todos
    
    func createRemoteTask(_ task: TaskItem) async throws -> RemoteTask {
        // 1. Construir la URL
        guard let url = URL(string: "\(baseURL)/todos") else {
            throw NetworkError.invalidURL
        }
        
        // 2. Crear el cuerpo de la petición usando Encodable
        let requestBody = RemoteTaskRequest(from: task)
        
        let bodyData: Data
        do {
            bodyData = try JSONEncoder().encode(requestBody)
        } catch {
            throw NetworkError.decodingError("Error al codificar la tarea: \(error.localizedDescription)")
        }
        
        // 3. Configurar el URLRequest para POST
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue("application/json", forHTTPHeaderField: "Accept")
        request.httpBody = bodyData
        request.timeoutInterval = 15
        
        // 4. Ejecutar la petición
        let data: Data
        let response: URLResponse
        
        do {
            (data, response) = try await session.data(for: request)
        } catch let urlError as URLError {
            if urlError.code == .notConnectedToInternet ||
               urlError.code == .networkConnectionLost {
                throw NetworkError.networkUnavailable
            }
            throw NetworkError.networkUnavailable
        }
        
        // 5. Validar respuesta HTTP (201 Created esperado para POST)
        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.noData
        }
        
        guard (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.serverError(httpResponse.statusCode)
        }
        
        // 6. Decodificar la respuesta
        //    JSONPlaceholder devuelve el objeto creado con un ID asignado
        do {
            let decoder = JSONDecoder()
            let createdTask = try decoder.decode(RemoteTask.self, from: data)
            return createdTask
        } catch {
            throw NetworkError.decodingError(error.localizedDescription)
        }
    }
}
```

4. Compila con **⌘B**.

**Salida esperada:** Build Succeeded sin errores.

5. Para verificar rápidamente que el servicio funciona, crea un archivo de prueba temporal. En el grupo `Services/Network`, crea `NetworkServiceTest_Temp.swift` (lo eliminarás al final):

```swift
//
//  NetworkServiceTest_Temp.swift
//  TaskManagerApp
//
//  Verificación temporal de conectividad. ELIMINAR después del lab.
//

import Foundation

/// Función temporal para verificar la conectividad con JSONPlaceholder.
/// Ejecutar desde un punto de entrada temporal o Playground.
func verificarConectividad() async {
    let service = TaskNetworkService()
    
    do {
        let tasks = try await service.fetchRemoteTasks()
        print("✅ Conectividad OK - Se recibieron \(tasks.count) tareas")
        if let primera = tasks.first {
            print("   Primera tarea: '\(primera.title)' (completada: \(primera.completed))")
        }
    } catch let error as NetworkError {
        print("❌ Error de red: \(error.errorDescription ?? "Desconocido")")
    } catch {
        print("❌ Error inesperado: \(error)")
    }
}
```

> **Nota:** Este archivo es solo para verificación. Lo eliminarás en el Paso 9 (Cleanup).

**Verificación:**

Confirma que el grupo `Services/Network/` contiene exactamente estos archivos:
- `RemoteTask.swift`
- `RemoteTaskRequest.swift`
- `NetworkError.swift`
- `TaskNetworkService.swift`
- `NetworkServiceTest_Temp.swift` (temporal)

Compila con **⌘B** — Build Succeeded.

---

### Paso 5 — Implementar `SyncViewModel` con `@Observable` y `@MainActor` (50 min)

**Objetivo:** Crear el ViewModel que orquesta la sincronización entre la API remota y el repositorio local SwiftData, gestionando los estados de la UI de forma reactiva.

**Instrucciones:**

1. En el grupo `ViewModels`, crea un nuevo archivo Swift llamado `SyncViewModel.swift`.

2. Primero, define el enum de estados de sincronización:

```swift
//
//  SyncViewModel.swift
//  TaskManagerApp
//
//  ViewModel que gestiona la sincronización de tareas remotas.
//

import Foundation
import SwiftData

// MARK: - Estado de sincronización

/// Representa los posibles estados del proceso de sincronización.
enum SyncState: Equatable {
    /// Sin actividad de sincronización.
    case idle
    
    /// Sincronización en progreso.
    case loading
    
    /// Sincronización completada exitosamente.
    /// - Parameter count: Número de tareas sincronizadas.
    case success(Int)
    
    /// Error durante la sincronización.
    /// - Parameter message: Descripción del error.
    case error(String)
}
```

3. Debajo del enum, añade la implementación del ViewModel:

```swift
// MARK: - SyncViewModel

/// ViewModel que coordina la obtención de tareas remotas y su persistencia local.
///
/// Usa `@Observable` para notificar cambios a las vistas SwiftUI
/// y `@MainActor` para garantizar que las actualizaciones de estado
/// ocurran en el hilo principal.
@MainActor
@Observable
final class SyncViewModel {
    
    // MARK: - Propiedades observables
    
    /// Estado actual del proceso de sincronización.
    var syncState: SyncState = .idle
    
    /// Mensaje informativo para el usuario.
    var statusMessage: String = ""
    
    /// Controla la visibilidad de la alerta de error.
    var showErrorAlert: Bool = false
    
    /// Mensaje de error para mostrar en la alerta.
    var errorAlertMessage: String = ""
    
    // MARK: - Dependencias
    
    /// Servicio de red para comunicación con la API.
    private let networkService: TaskNetworkServiceProtocol
    
    /// Contexto de SwiftData para persistencia local.
    private let modelContext: ModelContext?
    
    // MARK: - Inicialización
    
    /// Inicializa el ViewModel con sus dependencias.
    /// - Parameters:
    ///   - networkService: Servicio de red a utilizar.
    ///   - modelContext: Contexto de SwiftData para persistencia.
    init(
        networkService: TaskNetworkServiceProtocol = TaskNetworkService(),
        modelContext: ModelContext? = nil
    ) {
        self.networkService = networkService
        self.modelContext = modelContext
    }
    
    // MARK: - Acciones públicas
    
    /// Obtiene tareas remotas de JSONPlaceholder y las persiste en SwiftData.
    ///
    /// Flujo:
    /// 1. Establece estado `.loading`
    /// 2. Llama a `fetchRemoteTasks()` en el servicio de red
    /// 3. Convierte cada `RemoteTask` a `TaskItem` (dominio)
    /// 4. Inserta un `SDTaskItem` por cada tarea en el contexto de SwiftData
    /// 5. Establece estado `.success` o `.error` según resultado
    func syncRemoteTasks() async {
        // Evitar sincronizaciones simultáneas
        guard syncState != .loading else {
            print("⚠️ Sincronización ya en progreso, ignorando petición duplicada.")
            return
        }
        
        syncState = .loading
        statusMessage = "Descargando tareas remotas..."
        
        do {
            // 1. Obtener tareas remotas
            let remoteTasks = try await networkService.fetchRemoteTasks()
            
            statusMessage = "Procesando \(remoteTasks.count) tareas..."
            
            // 2. Convertir a modelo local y persistir
            var insertedCount = 0
            
            if let context = modelContext {
                for remoteTask in remoteTasks {
                    let taskItem = remoteTask.toTaskItem()
                    context.insert(SDTaskItem(from: taskItem))
                    insertedCount += 1
                }
                
                // 3. Guardar el contexto
                try context.save()
            } else {
                // Sin contexto de SwiftData (modo preview o testing)
                insertedCount = remoteTasks.count
                print("⚠️ ModelContext no disponible. Tareas obtenidas pero no persistidas.")
            }
            
            // 4. Actualizar estado a éxito
            syncState = .success(insertedCount)
            statusMessage = "✅ \(insertedCount) tareas sincronizadas correctamente."
            
        } catch let networkError as NetworkError {
            syncState = .error(networkError.errorDescription ?? "Error desconocido")
            statusMessage = "❌ \(networkError.errorDescription ?? "Error desconocido")"
            errorAlertMessage = networkError.errorDescription ?? "Error desconocido"
            showErrorAlert = true
        } catch {
            syncState = .error(error.localizedDescription)
            statusMessage = "❌ Error inesperado: \(error.localizedDescription)"
            errorAlertMessage = error.localizedDescription
            showErrorAlert = true
        }
    }
    
    /// Envía una tarea local al servidor remoto via POST.
    /// - Parameter task: El `TaskItem` a enviar.
    func uploadTask(_ task: TaskItem) async {
        syncState = .loading
        statusMessage = "Enviando tarea al servidor..."
        
        do {
            let createdRemoteTask = try await networkService.createRemoteTask(task)
            syncState = .success(1)
            statusMessage = "✅ Tarea creada en servidor con ID: \(createdRemoteTask.id)"
        } catch let networkError as NetworkError {
            syncState = .error(networkError.errorDescription ?? "Error desconocido")
            errorAlertMessage = networkError.errorDescription ?? "Error desconocido"
            showErrorAlert = true
        } catch {
            syncState = .error(error.localizedDescription)
            errorAlertMessage = error.localizedDescription
            showErrorAlert = true
        }
    }
    
    /// Restablece el estado a idle.
    func resetState() {
        syncState = .idle
        statusMessage = ""
        showErrorAlert = false
        errorAlertMessage = ""
    }
}
```

4. Compila con **⌘B**.

**Salida esperada:** Build Succeeded.

> **Explicación de `@MainActor`:** Al marcar la clase con `@MainActor`, garantizamos que **todas** las propiedades y métodos se ejecutan en el hilo principal. Esto es crucial porque las propiedades observables (`syncState`, `statusMessage`, etc.) se leen desde las vistas SwiftUI, que siempre deben actualizarse en el hilo principal. Sin `@MainActor`, tendríamos que envolver cada actualización de propiedad en `await MainActor.run { }` o usar `DispatchQueue.main.async { }` (el enfoque legacy).

**Verificación:**

En Xcode, verifica que `SyncViewModel.swift` compila sin errores y que el autocompletado reconoce `SyncState.loading`, `SyncState.success(Int)`, etc.

---

### Paso 6 — Integrar la sincronización en la interfaz SwiftUI (80 min)

**Objetivo:** Modificar la vista de lista de tareas SwiftUI para incluir un botón de sincronización, indicador de carga y alertas de error.

**Instrucciones:**

#### 6.1 — Crear la vista `SyncButton`

1. En el grupo de vistas SwiftUI de tu proyecto (por ejemplo, `Views/` o el grupo raíz), crea un nuevo archivo SwiftUI View llamado `SyncButtonView.swift`:

```swift
//
//  SyncButtonView.swift
//  TaskManagerApp
//
//  Botón reutilizable para sincronización de tareas remotas.
//

import SwiftUI

/// Vista que muestra un botón de sincronización con estados visuales.
struct SyncButtonView: View {
    
    /// Estado actual de la sincronización.
    let syncState: SyncState
    
    /// Acción a ejecutar al pulsar el botón.
    let action: () -> Void
    
    var body: some View {
        Button(action: action) {
            HStack(spacing: 6) {
                syncIcon
                Text(syncLabel)
                    .font(.subheadline)
            }
        }
        .disabled(syncState == .loading)
    }
    
    // MARK: - Subvistas privadas
    
    @ViewBuilder
    private var syncIcon: some View {
        switch syncState {
        case .idle:
            Image(systemName: "arrow.triangle.2.circlepath")
        case .loading:
            ProgressView()
                .scaleEffect(0.7)
        case .success:
            Image(systemName: "checkmark.circle.fill")
                .foregroundStyle(.green)
        case .error:
            Image(systemName: "exclamationmark.triangle.fill")
                .foregroundStyle(.red)
        }
    }
    
    private var syncLabel: String {
        switch syncState {
        case .idle:
            return "Sincronizar"
        case .loading:
            return "Sincronizando..."
        case .success(let count):
            return "\(count) tareas"
        case .error:
            return "Reintentar"
        }
    }
}

#Preview {
    VStack(spacing: 20) {
        SyncButtonView(syncState: .idle) { }
        SyncButtonView(syncState: .loading) { }
        SyncButtonView(syncState: .success(15)) { }
        SyncButtonView(syncState: .error("Error")) { }
    }
    .padding()
}
```

2. Compila y abre el **Canvas** (⌥⌘↩) para verificar que el preview muestra los cuatro estados del botón.

#### 6.2 — Integrar la sincronización en `SwiftDataTaskListView`

3. Abre `SwiftDataTaskListView.swift`, la vista con `@Query` que creaste en el Lab 03 (Paso 15). Usa esta vista y **no** `TaskListView`: `TaskListView` (Lab 02) recibe el repositorio por parámetro y la abre el botón "SwiftUI" de UIKit, así que si la reemplazaras por una versión sin ese parámetro el proyecto dejaría de compilar. `SwiftDataTaskListView` ya se abre desde el botón "SwiftData" con `.modelContainer(AppModelContainer.shared)`, que es lo que necesitan `@Query` y el `modelContext`.

4. Reemplaza el contenido del archivo por la siguiente versión. Las secciones marcadas con `// 🆕 NUEVO` son las de sincronización; el alta de tareas ahora reutiliza `AddTaskView` del Lab 02 con un `SwiftDataTaskRepository`, y `@Query` refresca la lista sola:

```swift
//
//  SwiftDataTaskListView.swift
//  TaskManagerApp
//
//  Lista de tareas en SwiftData con sincronización remota.
//

import SwiftUI
import SwiftData

struct SwiftDataTaskListView: View {
    
    // MARK: - Propiedades de SwiftData
    
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \SDTaskItem.createdAt, order: .reverse) private var tasks: [SDTaskItem]
    
    // 🆕 NUEVO: ViewModel de sincronización
    @State private var syncViewModel = SyncViewModel()
    @State private var showingAddTask = false
    
    var body: some View {
        NavigationStack {
            ZStack {
                if tasks.isEmpty && syncViewModel.syncState != .loading {
                    emptyStateView
                } else {
                    taskListContent
                }
            }
            .navigationTitle("Mis Tareas")
            .toolbar {
                // 🆕 NUEVO: Botón de sincronización en la barra de navegación
                ToolbarItem(placement: .topBarLeading) {
                    SyncButtonView(syncState: syncViewModel.syncState) {
                        Task {
                            await syncViewModel.syncRemoteTasks()
                        }
                    }
                }
                
                ToolbarItemGroup(placement: .topBarTrailing) {
                    NavigationLink(destination: PreferencesView()) {
                        Image(systemName: "gearshape")
                    }
                    Button {
                        showingAddTask = true
                    } label: {
                        Image(systemName: "plus")
                    }
                }
            }
            // Alta de tareas: AddTaskView del Lab 02 guardando en SwiftData
            .sheet(isPresented: $showingAddTask) {
                AddTaskView(
                    repository: SwiftDataTaskRepository(modelContext: modelContext),
                    onSave: {}   // @Query actualiza la lista automáticamente
                )
            }
            // 🆕 NUEVO: Alerta de error
            .alert(
                "Error de Sincronización",
                isPresented: $syncViewModel.showErrorAlert
            ) {
                Button("Aceptar", role: .cancel) {
                    syncViewModel.resetState()
                }
                Button("Reintentar") {
                    Task {
                        await syncViewModel.syncRemoteTasks()
                    }
                }
            } message: {
                Text(syncViewModel.errorAlertMessage)
            }
            // 🆕 NUEVO: Inyectar el ModelContext al ViewModel
            .onAppear {
                syncViewModel = SyncViewModel(
                    networkService: TaskNetworkService(),
                    modelContext: modelContext
                )
            }
        }
    }
    
    // MARK: - Subvistas
    
    /// Vista cuando la lista está vacía.
    private var emptyStateView: some View {
        ContentUnavailableView {
            Label("Sin tareas", systemImage: "checklist")
        } description: {
            Text("Pulsa '+' para crear una tarea o 'Sincronizar' para importar tareas remotas.")
        }
    }
    
    /// Contenido de la lista con indicador de carga.
    private var taskListContent: some View {
        VStack(spacing: 0) {
            // 🆕 NUEVO: Banner de estado de sincronización
            if syncViewModel.syncState == .loading {
                HStack {
                    ProgressView()
                        .padding(.trailing, 8)
                    Text(syncViewModel.statusMessage)
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
                .frame(maxWidth: .infinity)
                .padding(.vertical, 8)
                .background(Color(.systemGroupedBackground))
            }
            
            // 🆕 NUEVO: Banner de éxito (se muestra brevemente)
            if case .success(let count) = syncViewModel.syncState {
                HStack {
                    Image(systemName: "checkmark.circle.fill")
                        .foregroundStyle(.green)
                    Text("\(count) tareas importadas correctamente")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
                .frame(maxWidth: .infinity)
                .padding(.vertical, 8)
                .background(Color.green.opacity(0.1))
                .onAppear {
                    // Auto-dismiss después de 3 segundos
                    DispatchQueue.main.asyncAfter(deadline: .now() + 3) {
                        syncViewModel.resetState()
                    }
                }
            }
            
            List {
                ForEach(tasks) { task in
                    TaskRowView(task: task.toTaskItem())
                }
                .onDelete(perform: deleteTasks)
            }
        }
    }
    
    // MARK: - Acciones
    
    private func deleteTasks(at offsets: IndexSet) {
        for index in offsets {
            modelContext.delete(tasks[index])
        }
        try? modelContext.save()
    }
}

#Preview {
    SwiftDataTaskListView()
        .modelContainer(for: SDTaskItem.self, inMemory: true)
}
```

> **Nota:** `TaskRowView` y `AddTaskView` son las vistas del Lab 02; `TaskRowView` recibe un `TaskItem`, por eso se usa `task.toTaskItem()`. Lo esencial de este paso es integrar: (1) el `@State private var syncViewModel`, (2) el `SyncButtonView` en el toolbar, (3) la alerta de error, (4) el `.onAppear` que inyecta el `modelContext`, y (5) los banners de estado.

5. Compila con **⌘B**.

**Salida esperada:** Build Succeeded.

6. Ejecuta la aplicación en el simulador **iPhone 15 Pro** con **⌘R**.

7. En la lista UIKit toca **SwiftData** y, en esa pantalla, pulsa el botón **"Sincronizar"** en la esquina superior izquierda de la barra de navegación.

**Salida esperada:**

- Aparece un `ProgressView` con el texto "Sincronizando…"
- Después de 1-3 segundos, aparece un banner verde: "20 tareas importadas correctamente"
- La lista se actualiza mostrando 20 tareas importadas de JSONPlaceholder
- El banner desaparece después de 3 segundos

8. Desactiva la conexión a Internet del Mac (Wi-Fi off o desconectar cable) y pulsa "Sincronizar" nuevamente.

**Salida esperada:**

- Aparece una alerta con título "Error de Sincronización" y mensaje sobre la conexión de red
- Botones "Aceptar" y "Reintentar" visibles

9. Reactiva la conexión a Internet.

**Verificación:**

Confirma visualmente que:
- El botón muestra el ícono de flechas circulares en estado idle
- Durante la carga, muestra un spinner
- Tras el éxito, muestra un checkmark verde
- La lista contiene tareas con títulos en inglés provenientes de JSONPlaceholder

---

### Paso 7 — Integrar la sincronización en UIKit (30 min)

**Objetivo:** Añadir funcionalidad de sincronización al `UIViewController` existente usando `Task { }` para llamadas async y `UIActivityIndicatorView` para indicar carga.

**Instrucciones:**

1. Abre `UIKitViews/TaskListViewController.swift` (Lab 02, actualizado en el Lab 03). La extensión de sincronización va en otro archivo, y en Swift un miembro `private` solo es visible dentro de su archivo. Quita `private` de estas dos declaraciones para que la extensión pueda usarlas:

```swift
// Antes (Lab 03):
// private lazy var repository: TaskRepository = AppConfiguration.shared.makeRepository()
// private func loadTasks() { ... }

// Después:
lazy var repository: TaskRepository = AppConfiguration.shared.makeRepository()
func loadTasks() { ... }   // el cuerpo no cambia
```

2. En el grupo `UIKitViews`, crea el archivo `TaskListViewController+Sync.swift` con la siguiente extensión. El botón de sincronización se **agrega** a la izquierda de la barra sin quitar los botones "SwiftUI" (Lab 02) y "SwiftData" (Lab 03) que ya están ahí:

```swift
//
//  TaskListViewController+Sync.swift
//  TaskManagerApp
//
//  Extensión que añade funcionalidad de sincronización remota al ViewController UIKit.
//

import UIKit

// MARK: - Extensión de sincronización para UIKit

extension TaskListViewController {

    /// Identificador para reconocer el botón de sincronización (o su spinner)
    /// entre los demás botones de la barra.
    private static let syncItemID = "syncButton"

    /// Configura el botón de sincronización. Llamar desde `viewDidLoad()`.
    func setupSyncUI() {
        let syncButton = UIBarButtonItem(
            image: UIImage(systemName: "arrow.triangle.2.circlepath"),
            style: .plain,
            target: self,
            action: #selector(syncButtonTapped)
        )
        replaceSyncItem(with: syncButton)
    }

    /// Pone `item` en el lugar del botón de sincronización y conserva
    /// los demás botones de la izquierda (SwiftUI, SwiftData).
    private func replaceSyncItem(with item: UIBarButtonItem) {
        item.accessibilityIdentifier = Self.syncItemID
        let others = (navigationItem.leftBarButtonItems ?? [])
            .filter { $0.accessibilityIdentifier != Self.syncItemID }
        navigationItem.leftBarButtonItems = [item] + others
    }

    /// Acción del botón de sincronización.
    @objc func syncButtonTapped() {
        // Mostrar el indicador de actividad en lugar del botón
        let activityIndicator = UIActivityIndicatorView(style: .medium)
        activityIndicator.startAnimating()
        replaceSyncItem(with: UIBarButtonItem(customView: activityIndicator))

        // Usar Task { } para ejecutar código async desde un contexto síncrono
        Task {
            let service = TaskNetworkService()

            do {
                let remoteTasks = try await service.fetchRemoteTasks()

                // Task hereda el MainActor del ViewController:
                // podemos tocar la UI y el repositorio directamente.
                let taskItems = remoteTasks.map { $0.toTaskItem() }
                taskItems.forEach { repository.add($0) }
                loadTasks()

                let alert = UIAlertController(
                    title: "Sincronización Exitosa",
                    message: "Se importaron \(taskItems.count) tareas remotas.",
                    preferredStyle: .alert
                )
                alert.addAction(UIAlertAction(title: "Aceptar", style: .default))
                present(alert, animated: true)

            } catch let error as NetworkError {
                let alert = UIAlertController(
                    title: "Error de Sincronización",
                    message: error.errorDescription,
                    preferredStyle: .alert
                )
                alert.addAction(UIAlertAction(title: "Aceptar", style: .cancel))
                alert.addAction(UIAlertAction(title: "Reintentar", style: .default) { [weak self] _ in
                    self?.syncButtonTapped()
                })
                present(alert, animated: true)

            } catch {
                let alert = UIAlertController(
                    title: "Error",
                    message: error.localizedDescription,
                    preferredStyle: .alert
                )
                alert.addAction(UIAlertAction(title: "Aceptar", style: .default))
                present(alert, animated: true)
            }

            // Restaurar el botón de sincronización
            setupSyncUI()
        }
    }
}
```

> **Nota:** Las tareas se guardan con el repositorio activo de `AppConfiguration` (SwiftData por defecto), así que también aparecen en la pantalla **SwiftData**. Cada sincronización vuelve a importar las 20 tareas con identificadores nuevos; evitar duplicados queda como ejercicio opcional.

> **Nota sobre `Task { }` en UIKit:** Cuando creas un `Task { }` dentro de un método de un `UIViewController`, la tarea hereda el contexto de `MainActor` porque los ViewControllers de UIKit son implícitamente `@MainActor`. Esto significa que no necesitas `DispatchQueue.main.async` para actualizar la UI dentro del bloque `Task`.
>
> **Alternativa legacy con `DispatchQueue`:** Si estuvieras usando el patrón antiguo con completion handlers en lugar de `async/await`, necesitarías:
> ```swift
> // Patrón legacy (NO usar en este lab, solo referencia)
> URLSession.shared.dataTask(with: request) { data, response, error in
>     DispatchQueue.main.async {
>         // Actualizar UI aquí
>         self.tableView.reloadData()
>     }
> }.resume()
> ```

3. En `TaskListViewController.viewDidLoad()`, añade la llamada a `setupSyncUI()` después de `configureNavigationBar()`:

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    // ... código existente ...
    setupSyncUI() // 🆕 Añadir esta línea
}
```

4. Compila con **⌘B**.

**Salida esperada:** Build Succeeded.

**Verificación:**

Si ejecutas la vista UIKit, al pulsar el botón de sincronización:
- El botón se reemplaza por un spinner y los botones "SwiftUI" y "SwiftData" siguen visibles
- Tras completar, aparece una alerta con el conteo de tareas y la lista muestra las tareas importadas
- El botón se restaura a su estado original

---

### Paso 8 — Demostrar conceptos de concurrencia (22 min)

**Objetivo:** Comprender y documentar los patrones de concurrencia utilizados en el proyecto.

**Instrucciones:**

1. Crea un archivo de documentación en el grupo `Services/Network` llamado `ConcurrencyNotes.swift`:

```swift
//
//  ConcurrencyNotes.swift
//  TaskManagerApp
//
//  Notas de referencia sobre los patrones de concurrencia usados en el proyecto.
//  Este archivo es solo documentación con ejemplos — no se ejecuta en producción.
//

import Foundation

// MARK: - Patrones de Concurrencia en TaskManagerApp

/*
 ═══════════════════════════════════════════════════════════════
 1. async/await (Swift Concurrency - PATRÓN PRINCIPAL)
 ═══════════════════════════════════════════════════════════════
 
 Usado en: TaskNetworkService.fetchRemoteTasks()
 
 - `async` marca una función como asíncrona
 - `await` suspende la ejecución hasta que el resultado esté disponible
 - NO bloquea el hilo (a diferencia de semáforos o DispatchGroup.wait())
 - El compilador garantiza la seguridad de concurrencia
 
 Ejemplo:
 ```
 func fetchRemoteTasks() async throws -> [RemoteTask] {
     let (data, response) = try await session.data(for: request)
     // La ejecución se suspende aquí y se reanuda cuando llegan los datos
     return try decoder.decode([RemoteTask].self, from: data)
 }
 ```
 
 ═══════════════════════════════════════════════════════════════
 2. Task { } (Creación de contexto asíncrono)
 ═══════════════════════════════════════════════════════════════
 
 Usado en: SwiftDataTaskListView (SwiftUI), TaskListViewController (UIKit)
 
 - Crea un contexto asíncrono desde código síncrono
 - Hereda el actor del contexto donde se crea
 - En SwiftUI: hereda MainActor si se crea en un View
 - En UIKit: hereda MainActor si se crea en un UIViewController
 
 Ejemplo:
 ```
 Button("Sincronizar") {
     Task {
         await viewModel.syncRemoteTasks()
     }
 }
 ```
 
 ═══════════════════════════════════════════════════════════════
 3. @MainActor (Aislamiento al hilo principal)
 ═══════════════════════════════════════════════════════════════
 
 Usado en: SyncViewModel
 
 - Garantiza que TODAS las propiedades y métodos de la clase
   se ejecutan en el hilo principal
 - Imprescindible para clases cuyos datos se leen desde la UI
 - Alternativa a `DispatchQueue.main.async` pero con verificación
   en tiempo de compilación
 
 Ejemplo:
 ```
 @MainActor
 @Observable
 final class SyncViewModel {
     var syncState: SyncState = .idle  // Siempre se modifica en main thread
 }
 ```
 
 ═══════════════════════════════════════════════════════════════
 4. DispatchQueue.main.async (LEGACY - solo referencia)
 ═══════════════════════════════════════════════════════════════
 
 Patrón anterior a Swift Concurrency. Aún válido pero NO recomendado
 para código nuevo.
 
 Ejemplo legacy:
 ```
 URLSession.shared.dataTask(with: url) { data, response, error in
     // ⚠️ Estamos en un hilo de background
     DispatchQueue.main.async {
         // ✅ Ahora estamos en el hilo principal
         self.label.text = "Datos recibidos"
     }
 }.resume()
 ```
 
 Equivalente moderno con async/await:
 ```
 Task {
     let (data, _) = try await URLSession.shared.data(from: url)
     // ✅ Si estamos en @MainActor, ya estamos en main thread
     self.label.text = "Datos recibidos"
 }
 ```
 ═══════════════════════════════════════════════════════════════
*/
```

2. Revisa el archivo para comprender los cuatro patrones de concurrencia documentados.

**Verificación:**

Compila con **⌘B** — Build Succeeded. El archivo es solo documentación y no afecta la ejecución.

---

### Paso 9 — Commit y merge a `develop` (20 min)

**Objetivo:** Confirmar todos los cambios en Git, hacer merge de la rama feature a `develop` y push a GitHub.

**Instrucciones:**

1. Primero, elimina el archivo temporal de verificación:

```bash
cd ~/Developer/TaskManagerApp
rm -f TaskManagerApp/Services/Network/NetworkServiceTest_Temp.swift
```

2. En Xcode, si el archivo aún aparece en el Project Navigator, haz clic derecho → **Delete** → **Move to Trash**.

3. Verifica los archivos modificados y nuevos:

```bash
git status
```

**Salida esperada (aproximada):**

```
On branch feature/networking
Changes not staged for commit:
  modified:   TaskManagerApp.xcodeproj/project.pbxproj

Untracked files:
  TaskManagerApp/Services/Network/ConcurrencyNotes.swift
  TaskManagerApp/Services/Network/NetworkError.swift
  TaskManagerApp/Services/Network/RemoteTask.swift
  TaskManagerApp/Services/Network/RemoteTaskRequest.swift
  TaskManagerApp/Services/Network/TaskNetworkService.swift
  TaskManagerApp/ViewModels/SyncViewModel.swift
  TaskManagerApp/Views/SyncButtonView.swift
  ...
```

4. Añade todos los archivos al staging area:

```bash
git add -A
```

5. Crea el commit con el mensaje siguiendo Conventional Commits:

```bash
git commit -m "feat: network layer with URLSession and Codable

- Add RemoteTask model (Decodable) mapping JSONPlaceholder /todos
- Add RemoteTaskRequest model (Encodable) for POST requests
- Add NetworkError enum with localized descriptions
- Add TaskNetworkService with async/await GET and POST methods
- Add SyncViewModel with @MainActor and @Observable
- Add SyncButtonView for SwiftUI sync UI
- Integrate sync in SwiftDataTaskListView (SwiftUI) with loading/success/error states
- Add UIKit sync extension with UIActivityIndicatorView
- Add concurrency patterns documentation"
```

**Salida esperada:**

```
[feature/networking xxxxxxx] feat: network layer with URLSession and Codable
 X files changed, XXX insertions(+)
 create mode 100644 TaskManagerApp/Services/Network/...
```

6. Cambia a la rama `develop` y haz merge:

```bash
git checkout develop
git merge feature/networking
```

**Salida esperada:**

```
Updating xxxxxxx..xxxxxxx
Fast-forward
 TaskManagerApp/Services/Network/ConcurrencyNotes.swift      | XX +++
 TaskManagerApp/Services/Network/NetworkError.swift           | XX +++
 ...
 X files changed, XXX insertions(+)
```

7. Push a GitHub:

```bash
git push origin develop
```

**Salida esperada:**

```
Enumerating objects: XX, done.
...
To github.com:<tu-usuario>/task-manager-ios.git
   xxxxxxx..xxxxxxx  develop -> develop
```

8. Opcionalmente, elimina la rama feature local:

```bash
git branch -d feature/networking
```

**Salida esperada:**

```
Deleted branch feature/networking (was xxxxxxx).
```

**Verificación:**

```bash
git log --oneline -3
```

Debe mostrar el commit de networking en la parte superior:

```
xxxxxxx feat: network layer with URLSession and Codable
xxxxxxx (commits anteriores del Lab 03)
...
```

---

## Validación y Pruebas

Ejecuta las siguientes verificaciones para confirmar que todo el lab funciona correctamente:

### Prueba 1 — Petición GET funcional

1. Ejecuta la app en el simulador iPhone 15 Pro (**⌘R**).
2. Pulsa el botón **"Sincronizar"** en la barra de navegación.
3. **Resultado esperado:** La lista se actualiza con 20 tareas de JSONPlaceholder. Los títulos son en inglés (ej: "delectus aut autem").

### Prueba 2 — Manejo de error de red

1. En el Mac, desactiva Wi-Fi o desconecta la red.
2. Pulsa **"Sincronizar"**.
3. **Resultado esperado:** Aparece una alerta con "Error de Sincronización" y un mensaje sobre la conexión de red.
4. Reactiva la red y pulsa **"Reintentar"** en la alerta.
5. **Resultado esperado:** Las tareas se cargan correctamente.

### Prueba 3 — Compilación limpia

```bash
cd ~/Developer/TaskManagerApp
xcodebuild -scheme TaskManagerApp -destination 'platform=iOS Simulator,name=iPhone 15 Pro,OS=17.5' build 2>&1 | tail -5
```

**Resultado esperado:** La última línea debe contener `** BUILD SUCCEEDED **`.

### Prueba 4 — Estructura de archivos

```bash
find TaskManagerApp/Services/Network -name "*.swift" | sort
```

**Resultado esperado:**

```
TaskManagerApp/Services/Network/ConcurrencyNotes.swift
TaskManagerApp/Services/Network/NetworkError.swift
TaskManagerApp/Services/Network/RemoteTask.swift
TaskManagerApp/Services/Network/RemoteTaskRequest.swift
TaskManagerApp/Services/Network/TaskNetworkService.swift
```

### Prueba 5 — Verificación de Git

```bash
git log --oneline --graph -5
```

El commit `feat: network layer with URLSession and Codable` debe estar presente en `develop`.

---

## Solución de Problemas

### Problema 1: Error de compilación "Cannot find type 'TaskItem' in scope" en `RemoteTask.swift`

**Síntomas:** Xcode muestra el error `Cannot find type 'TaskItem' in scope` en la función `toTaskItem()` del archivo `RemoteTask.swift`. El proyecto no compila.

**Causa:** El archivo `RemoteTask.swift` no pertenece al mismo target que el modelo `TaskItem`, o el modelo `TaskItem` tiene un nombre diferente en tu proyecto (por ejemplo, `Task`, `TodoItem`, etc.).

**Solución:**

1. Selecciona `RemoteTask.swift` en el Project Navigator.
2. En el panel derecho (**File Inspector**), verifica que en la sección **Target Membership** esté marcado el checkbox `TaskManagerApp`.
3. Haz lo mismo con el archivo que contiene tu modelo `TaskItem`.
4. Si tu modelo tiene un nombre diferente, actualiza todas las referencias en `RemoteTask.toTaskItem()` y `RemoteTaskRequest.init(from:)` para usar el nombre correcto.
5. Limpia el build con **⇧⌘K** y recompila con **⌘B**.

---

### Problema 2: La sincronización se queda en estado "loading" indefinidamente

**Síntomas:** Al pulsar "Sincronizar", el spinner aparece pero nunca desaparece. No se muestra ni éxito ni error. La app no se congela (sigue respondiendo a toques).

**Causa:** El `ModelContext` no se inyectó correctamente en el `SyncViewModel`, lo que puede causar que `context.save()` lance una excepción silenciosa, o bien el timeout de la petición es demasiado largo y la red está muy lenta.

**Solución:**

1. Verifica que el `.onAppear` en `SwiftDataTaskListView` inyecta correctamente el `modelContext`:

```swift
.onAppear {
    syncViewModel = SyncViewModel(
        networkService: TaskNetworkService(),
        modelContext: modelContext
    )
}
```

2. Si el problema persiste, añade logging temporal en `SyncViewModel.syncRemoteTasks()`:

```swift
print("🔍 Estado antes de fetch: \(syncState)")
let remoteTasks = try await networkService.fetchRemoteTasks()
print("🔍 Tareas recibidas: \(remoteTasks.count)")
```

3. Revisa la consola de Xcode (**⇧⌘C**) para ver dónde se detiene el flujo.
4. Si ves `⚠️ ModelContext no disponible`, el contexto es `nil`. Verifica que la acción `showSwiftDataTapped(_:)` del Lab 03 aplica `.modelContainer(AppModelContainer.shared)` a `SwiftDataTaskListView` antes de crear el `UIHostingController`.
5. Reduce el `timeoutInterval` en `TaskNetworkService` a `10` segundos para detectar timeouts más rápido.

---

## Limpieza

1. **Eliminar archivo temporal** (si no se hizo en el Paso 9):

```bash
rm -f TaskManagerApp/Services/Network/NetworkServiceTest_Temp.swift
```

2. **Verificar que no quedan archivos temporales:**

```bash
find ~/Developer/TaskManagerApp -name "*_Temp*" -o -name "*_temp*"
```

Debe devolver vacío (sin resultados).

3. **Estado final del repositorio:**

```bash
cd ~/Developer/TaskManagerApp
git status
```

**Salida esperada:**

```
On branch develop
nothing to commit, working tree clean
```

---

## Reto Opcional (KMP) — Servicio de Red Compartido con Ktor

> Este reto es **opcional** y no forma parte de la evaluación. Se trabaja sobre el proyecto KMP de los labs anteriores.

**Objetivo:** Reescribir `TaskNetworkService` en el módulo `shared` con Ktor y kotlinx.serialization, y llamarlo desde la app iOS.

**Instrucciones:**

1. Agrega Ktor (`ktor-client-core`, `ktor-client-content-negotiation`, `ktor-serialization-kotlinx-json`) y los motores de cada plataforma (`ktor-client-okhttp` para Android y `ktor-client-darwin` para iOS).
2. Declara el modelo remoto como `@Serializable data class` con los campos de `/todos` de JSONPlaceholder.
3. Implementa en `commonMain` un servicio con funciones `suspend` para el GET de `/todos` y el POST de una tarea.
4. Desde Swift, llama a las funciones `suspend` con `try await` y muestra las tareas en la app iOS.

**Resultado esperado:** el servicio de red con Ktor vive en el módulo `shared` y la app iOS lo llama con `try await`, con el mismo resultado que la versión con `URLSession`.

---

## Resumen

En este laboratorio implementaste la **capa de networking completa** para TaskManagerApp:

| Componente | Archivo | Responsabilidad |
|---|---|---|
| **Modelo remoto** | `RemoteTask.swift` | Decodifica JSON de JSONPlaceholder y convierte a `TaskItem` |
| **Modelo de petición** | `RemoteTaskRequest.swift` | Codifica `TaskItem` a JSON para peticiones POST |
| **Errores tipados** | `NetworkError.swift` | Enum con casos específicos de error de red |
| **Servicio de red** | `TaskNetworkService.swift` | GET/POST con `URLSession` + `async/await` |
| **ViewModel** | `SyncViewModel.swift` | Orquesta sincronización con `@MainActor` + `@Observable` |
| **Vista SwiftUI** | `SyncButtonView.swift` + `SwiftDataTaskListView` | UI reactiva con estados loading/success/error |
| **Extensión UIKit** | `TaskListViewController+Sync.swift` | Sincronización con `Task { }` y `UIActivityIndicatorView` |
| **Documentación** | `ConcurrencyNotes.swift` | Referencia de patrones de concurrencia |

**Conceptos clave aplicados:**

- **`URLSession.shared.data(for:)`** con `async/await` para peticiones HTTP
- **`Codable`** (`Decodable` + `Encodable`) para parsing JSON bidireccional
- **`@MainActor`** para garantizar actualizaciones de UI en el hilo principal
- **`Task { }`** para crear contextos asíncronos desde código síncrono
- **Protocolo `TaskNetworkServiceProtocol`** para facilitar testing futuro con mocks
- **Git feature branch workflow** con Conventional Commits

### Recursos Adicionales

| Recurso | URL |
|---|---|
| URLSession — Apple Documentation | https://developer.apple.com/documentation/foundation/urlsession |
| Swift Concurrency — The Swift Programming Language | https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/ |
| JSONPlaceholder API | https://jsonplaceholder.typicode.com |
| Codable — Apple Documentation | https://developer.apple.com/documentation/swift/codable |
| @MainActor — Swift Evolution | https://github.com/apple/swift-evolution/blob/main/proposals/0316-global-actors.md |

---

## Anexo — Corrección pendiente en la presentación (diapositiva 274, «Indicador de Carga en UIKit»)

El código de la diapositiva 274 es una versión resumida del Paso 7 y difiere del código de este laboratorio en tres puntos. Mientras no se actualice la diapositiva, **el código válido es el de este README**.

| # | En la diapositiva 274 | En este README (Paso 7) | Por qué importa |
|---|---|---|---|
| 1 | `navigationItem.leftBarButtonItem = UIBarButtonItem(customView: indicator)` | `replaceSyncItem(with:)` actualiza `navigationItem.leftBarButtonItems` | Asignar `leftBarButtonItem` borra los botones "SwiftUI" (Lab 02) y "SwiftData" (Lab 03). |
| 2 | `tasks = remote.map { $0.toTaskItem() }` y `tableView.reloadData()` | `taskItems.forEach { repository.add($0) }` y `loadTasks()` | La diapositiva solo muestra las tareas en pantalla. El README las guarda en el repositorio activo (SwiftData), así que no se pierden al volver a cargar la lista y también aparecen en la pantalla SwiftData. |
| 3 | `mostrarError(error)` | `UIAlertController` con "Aceptar" y "Reintentar", distinguiendo `NetworkError` | `mostrarError(_:)` no existe en el proyecto, así que el código de la diapositiva no compila tal cual. |

**Código de reemplazo sugerido para la diapositiva** (versión resumida y consistente con el README):

```swift
@objc func syncButtonTapped() {
    // 1. Mostrar el indicador sin quitar los demás botones
    let indicator = UIActivityIndicatorView(style: .medium)
    indicator.startAnimating()
    replaceSyncItem(with: UIBarButtonItem(customView: indicator))

    // 2. Llamar código async desde un método síncrono
    Task {
        do {
            let remote = try await TaskNetworkService().fetchRemoteTasks()
            remote.map { $0.toTaskItem() }.forEach { repository.add($0) }
            loadTasks()
        } catch {
            let alert = UIAlertController(title: "Error de Sincronización",
                                          message: error.localizedDescription,
                                          preferredStyle: .alert)
            alert.addAction(UIAlertAction(title: "Aceptar", style: .cancel))
            present(alert, animated: true)
        }
        // 3. Restaurar el botón de sincronización
        setupSyncUI()
    }
}
```

**Puntos clave de la diapositiva que no cambian:** `UIActivityIndicatorView` con `startAnimating()` y `stopAnimating()`, `hidesWhenStopped` en `true` por defecto, `Task { }` que hereda el `MainActor` del ViewController y `ProgressView()` como equivalente en SwiftUI.

> **Nota:** En la versión del README, `stopAnimating()` no hace falta, porque `setupSyncUI()` reemplaza el indicador por el botón y el indicador sale de la barra.