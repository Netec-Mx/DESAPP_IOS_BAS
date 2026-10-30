# Retos Opcionales (KMP) por capítulo

<!-- Capítulo 01: pegar antes de la sección Resumen -->

## Reto Opcional (KMP) — Saludo desde el Módulo Compartido

> Este reto es **opcional** y no forma parte de la evaluación. Requiere **Android Studio** con el plugin **Kotlin Multiplatform**, además de Xcode.

**Objetivo:** Crear un proyecto Kotlin Multiplatform y mostrar en la app iOS un texto generado por el módulo `shared`.

**Instrucciones:**

1. En Android Studio, crea un proyecto con el asistente **Kotlin Multiplatform** y marca los destinos Android e iOS.
2. Revisa la estructura generada: `shared/src/commonMain` (código común), `androidApp` e `iosApp`.
3. Abre `iosApp/iosApp.xcodeproj` en Xcode y ejecuta la app en el simulador iPhone 15 Pro.
4. En la vista principal de `iosApp`, importa el framework con `import Shared` y muestra el resultado de `Greeting().greet()`.
5. En `shared/src/commonMain`, crea el modelo compartido que usarán los siguientes retos:

```kotlin
data class Tarea(
    val id: String,
    val titulo: String,
    val descripcion: String? = null,
    val completada: Boolean = false
)
```

**Resultado esperado:** la app iOS muestra el saludo que devuelve la clase `Greeting` del módulo `shared`, lo que confirma que el código Kotlin se usa desde Swift, y el modelo `Tarea` compila en `commonMain`.

---

<!-- Capítulo 02: pegar antes de la sección Resumen -->

## Reto Opcional (KMP) — Lista de Tareas con Compose Multiplatform

> Este reto es **opcional** y no forma parte de la evaluación. Se trabaja sobre el proyecto KMP del Lab 01.

**Objetivo:** Mostrar la lista de tareas del módulo `shared` en una pantalla de Compose Multiplatform y abrirla desde la app iOS.

**Instrucciones:**

1. Verifica que el proyecto KMP del Lab 01 tenga habilitado Compose Multiplatform (el asistente lo agrega si marcaste la opción de UI compartida).
2. En `commonMain`, crea un `@Composable fun ListaTareas(tareas: List<Tarea>)` que muestre las tareas con `LazyColumn`. Usa el modelo `Tarea` del reto 1 y, por ahora, una lista fija de 3 tareas de ejemplo (en el reto 3 vendrán de SQLDelight).
3. En `iosMain`, expón la pantalla con `fun ListaTareasViewController() = ComposeUIViewController { ListaTareas(...) }`.
4. En la app iOS, envuelve ese controlador en un `UIViewControllerRepresentable` y muéstralo desde SwiftUI (o haz push desde UIKit).

**Resultado esperado:** la app iOS abre una pantalla Compose con las tareas del módulo `shared`, junto a las pantallas nativas de UIKit y SwiftUI.

---

<!-- Capítulo 03: pegar antes de la sección Resumen -->

## Reto Opcional (KMP) — Persistencia Compartida con SQLDelight

> Este reto es **opcional** y no forma parte de la evaluación. Se trabaja sobre el proyecto KMP del Lab 01.

**Objetivo:** Guardar las tareas en el módulo `shared` con SQLDelight y mostrarlas en la app iOS.

**Instrucciones:**

1. Agrega el plugin y las dependencias de SQLDelight al `build.gradle.kts` del módulo `shared` (driver de Android y driver nativo para iOS).
2. Crea el archivo `Tarea.sq` en `shared/src/commonMain/sqldelight` con la tabla de tareas y las consultas para insertar, listar y eliminar.
3. En `commonMain`, declara la **interfaz** `TareasRepositorio` (agregar, listar, eliminar) e impleméntala en `SQLDelightTareasRepositorio` con las consultas generadas por SQLDelight. La app iOS depende de la interfaz, no de la implementación (es el mismo DIP de la diapositiva «SOLID en el Módulo Compartido», y la interfaz que usa el reto 6).
4. Crea el driver de cada plataforma con `expect`/`actual` (o con una fábrica de driver por plataforma).
5. Desde la app iOS, llama al repositorio y muestra la lista de tareas.

**Resultado esperado:** las tareas se guardan con SQLDelight en el módulo `shared`, la app iOS las muestra y se conservan al cerrar y volver a abrir la app.

---

<!-- Capítulo 04: pegar antes de la sección Resumen -->

## Reto Opcional (KMP) — Servicio de Red Compartido con Ktor

> Este reto es **opcional** y no forma parte de la evaluación. Se trabaja sobre el proyecto KMP de los labs anteriores.

**Objetivo:** Reescribir `TaskNetworkService` en el módulo `shared` con Ktor y kotlinx.serialization, y llamarlo desde la app iOS.

**Instrucciones:**

1. Agrega Ktor (`ktor-client-core`, `ktor-client-content-negotiation`, `ktor-serialization-kotlinx-json`) y los motores de cada plataforma (`ktor-client-okhttp` para Android y `ktor-client-darwin` para iOS).
2. Declara el modelo remoto como `@Serializable data class TareaRemota` con los campos de `/todos` de JSONPlaceholder (`userId`, `id`, `title`, `completed`) y una función que la convierta a `Tarea`.
3. Implementa en `commonMain` un servicio con funciones `suspend` para el GET de `/todos` y el POST de una tarea.
4. Desde Swift, llama a las funciones `suspend` con `try await` y muestra las tareas en la app iOS.

**Resultado esperado:** el servicio de red con Ktor vive en el módulo `shared` y la app iOS lo llama con `try await`, con el mismo resultado que la versión con `URLSession`.

---

<!-- Capítulo 05: pegar antes de la sección Resumen -->

## Reto Opcional (KMP) — Buscador Compartido con StateFlow

> Este reto es **opcional** y no forma parte de la evaluación. Se trabaja sobre el proyecto KMP de los labs anteriores.

**Objetivo:** Mover la lógica del buscador al módulo `shared` con `StateFlow` y `debounce`, y consumirla desde SwiftUI.

**Instrucciones:**

1. En `commonMain`, crea `BuscadorViewModel` con un `MutableStateFlow` para el texto de búsqueda.
2. Expón los resultados como `StateFlow<List<Tarea>>` aplicando `debounce(300)` y `map` sobre el texto, y conviértelos con `stateIn`.
3. Agrega el plugin **SKIE** (de Touchlab) al módulo `shared` para que Swift vea los `Flow` de Kotlin como `AsyncSequence`.
4. En SwiftUI, recorre los resultados con `for await` dentro de `.task { }` y actualiza la lista.

**Resultado esperado:** la búsqueda se filtra mientras el usuario escribe, la lógica vive en el módulo `shared` y SwiftUI recibe los resultados con `for await`.

---

<!-- Capítulo 06: pegar antes de la sección Resumen -->

## Reto Opcional (KMP) — Pruebas en commonTest

> Este reto es **opcional** y no forma parte de la evaluación. Se trabaja sobre el proyecto KMP de los labs anteriores.

**Objetivo:** Probar el repositorio compartido en `commonTest` con kotlin.test, de modo que las mismas pruebas corran en Android y en iOS.

**Instrucciones:**

1. Agrega `kotlin("test")` y `kotlinx-coroutines-test` a las dependencias de `commonTest`.
2. Crea un `FakeTareasRepositorio` que implemente la interfaz `TareasRepositorio` del reto 3 con una lista en memoria (en código compartido no se pueden usar bibliotecas de mocks de Android).
3. Escribe pruebas con `@Test` y `assertEquals` para agregar, listar y eliminar tareas; usa `runTest` para probar funciones `suspend` y `Flow`.
4. Ejecuta las pruebas con `./gradlew :shared:allTests`.

**Resultado esperado:** las pruebas de `commonTest` pasan en Android y en iOS al ejecutar `./gradlew :shared:allTests`.