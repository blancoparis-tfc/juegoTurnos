# Arquitectura ECS en Bevy

La arquitectura de **Bevy** (el motor de videojuegos en Rust) se basa en el patrón **ECS (Entity Component System)**. A diferencia de los motores tradicionales orientados a objetos (como Unity o Godot, que usan jerarquías de clases), Bevy separa completamente los datos de la lógica.

## Los 3 pilares del ECS en Bevy

* **Entities (Entidades):** Son simplemente identificadores únicos (como un ID numérico). Una entidad por sí sola no contiene datos ni comportamiento. Por ejemplo, un enemigo, una bala o la cámara son entidades.
* **Components (Componentes):** Son estructuras de datos (`structs` en Rust) que se adjuntan a las entidades. Definen qué "es" o qué "tiene" una entidad. Por ejemplo, puedes tener componentes como `Posicion`, `Velocidad` o `Salud`.
* **Systems (Sistemas):** Son funciones normales de Rust que contienen toda la lógica del juego. Los sistemas consultan (*queries*) entidades que tienen ciertos componentes y modifican sus datos. Por ejemplo, un `sistema_movimiento` busca todas las entidades que tengan `Posicion` y `Velocidad` para actualizar sus coordenadas.

## Piezas clave propias de Bevy

Para que el juego funcione de forma ordenada y rápida, Bevy añade otros conceptos fundamentales a su ECS:

* **World (Mundo):** Es el contenedor principal que almacena todas las entidades, componentes y recursos.
* **Resources (Recursos):** Son datos globales y únicos a los que cualquier sistema puede acceder. Se usan para cosas que no pertenecen a una entidad concreta, como el `Tiempo` del juego, la puntuación global o la configuración de entrada del teclado.
* **Queries (Consultas):** Es la herramienta que usan los sistemas para pedirle al `World` los componentes exactos que necesitan leer o modificar en cada fotograma.
* **Schedules y Plugins:** Bevy organiza cuándo se ejecuta cada sistema mediante horarios (*Schedules*), como `Startup` (al iniciar) o `Update` (en cada fotograma). Además, todo en Bevy es modular y se agrupa en **Plugins**.

## Ventajas de este enfoque

* **Alto rendimiento:** Al guardar los componentes del mismo tipo juntos en la memoria (en tablas o *archetypes*), el procesador lee los datos de forma lineal y aprovecha al máximo la caché de la CPU.
* **Paralelismo automático:** Como cada sistema declara exactamente qué datos lee y qué datos escribe, Bevy ejecuta en paralelo y de forma segura todos los sistemas que no interfieren entre sí.
* **Código limpio y reutilizable:** Puedes añadir o quitar comportamientos a un objeto en tiempo de ejecución simplemente añadiendo o quitando componentes, sin problemas de herencia compleja.
