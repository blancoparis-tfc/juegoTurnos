# 🖱️ MOVIMIENTO DE UNIDADES CON EL RATÓN: PROYECTO HEX-TACTICS

Para conectar el clic del ratón con el desplazamiento de una unidad en **Bevy**, utilizamos una arquitectura desacoplada basada en **Eventos (`Events`)**. 

La capa visual no mueve al personaje directamente. En su lugar, el sistema gráfico detecta dónde hizo clic el usuario y lanza un mensaje (`EventoOrdenMovimiento`). Luego, el sistema táctico intercepta ese mensaje, valida si el movimiento cumple las reglas del tablero (límites del mapa, distancia y Puntos de Acción) y aplica el cambio.

---

### 1. Guardar el Tablero como Recurso Global (`src/game_systems.rs`)
Para evitar que la unidad camine fuera de los límites del mapa al hacer clic en el espacio vacío, el sistema de movimiento necesita consultar qué hexágonos existen realmente. Convertimos `TableroTactico` en un **Recurso (`Resource`)** global de Bevy.

Añade o actualiza esto en `src/game_systems.rs`:

```rust
use crate::hex_grid::{HexCoord, TipoTerreno};
use std::collections::HashMap;

// Convertimos el tablero en un Recurso global accesible por cualquier sistema
#[derive(Resource, Debug, Default)]
pub struct TableroTactico {
    pub casillas: HashMap<HexCoord, TipoTerreno>,
}

// Nuevo Evento: Se dispara cuando el jugador hace clic en la pantalla
#[derive(Event, Debug)]
pub struct EventoOrdenMovimiento {
    pub destino: HexCoord,
}
```

---

### 2. Actualizar el Sistema de Movimiento (`src/game_systems.rs`)
Modificamos `sistema_movimiento_tactico` para que escuche el `EventoOrdenMovimiento`. El sistema comprobará tres reglas fundamentales antes de autorizar el paso:
1. Que el hexágono destino exista dentro del diccionario `TableroTactico`.
2. Que esté a exactamente **1 casilla de distancia** (movimiento paso a paso).
3. Que la unidad activa tenga al menos **1 Punto de Acción (PA)** disponible.

Reemplaza tu antiguo `sistema_movimiento_tactico` en `src/game_systems.rs` por este:

```rust
/// Sistema Update: Mueve la unidad activa hacia el hexágono donde se hizo clic
pub fn sistema_movimiento_tactico(
    mut eventos_movimiento: EventReader<EventoOrdenMovimiento>,
    mut query: Query<(&mut PosicionHex, &mut PuntosAccion), With<Unidad>>,
    cola: Res<ColaIniciativa>,
    tablero: Res<TableroTactico>,
) {
    // Leemos todos los clics ocurridos en este fotograma
    for evento in eventos_movimiento.read() {
        if cola.orden_unidades.is_empty() { return; }

        // 1. Validamos que el destino exista dentro de los límites del mapa
        if !tablero.casillas.contains_key(&evento.destino) {
            println!("🚫 Clic fuera del tablero: {:?}", evento.destino);
            continue;
        }

        let entidad_activa = cola.orden_unidades[cola.indice_actual];

        if let Ok((mut posicion, mut pa)) = query.get_mut(entidad_activa) {
            // 2. Validamos que el destino sea un vecino directo (distancia == 1)
            let distancia = posicion.0.distancia_a(&evento.destino);
            if distancia != 1 {
                println!("⚠️ Solo puedes moverte a casillas adyacentes (Distancia actual: {})", distancia);
                continue;
            }

            // 3. Validamos Puntos de Acción disponibles
            if pa.actuales >= 1 {
                posicion.0 = evento.destino;
                pa.actuales -= 1;
                println!("🏃 Unidad movida a {:?}. PA restantes: {}", posicion.0, pa.actuales);
            } else {
                println!("🚫 ¡Sin Puntos de Acción! Termina el turno de esta unidad.");
            }
        }
    }
}
```

---

### 3. Emitir la Orden al Hacer Clic (`src/render_system.rs`)
Actualizamos `dibujar_tablero_inicial` para que rellene el recurso `TableroTactico` al generar los 9 hexágonos, y modificamos `detectar_clic_tablero` para que envíe el evento de movimiento al motor.

Actualiza `src/render_system.rs` con estos cambios:

```rust
use bevy::window::PrimaryWindow;
use crate::hex_grid::{HexCoord, TipoTerreno};
use crate::game_systems::{PosicionHex, Unidad, TableroTactico, EventoOrdenMovimiento};

/// Sistema Startup: Genera los 9 hexágonos y los registra en el recurso TableroTactico
pub fn dibujar_tablero_inicial(
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<ColorMaterial>>,
    mut tablero: ResMut<TableroTactico>, // Pedimos acceso de escritura al tablero lógico
) {
    let forma_hex = meshes.add(RegularPolygon::new(RADIO_HEXAGONO * 0.94, 6));
    let color_llanura = materials.add(Color::srgb(0.35, 0.55, 0.30));
    let color_colina = materials.add(Color::srgb(0.55, 0.45, 0.30));

    for r in 0..3 {
        for q in (0 - r)..(3 - r) {
            let coord = HexCoord::new(q, r);
            let pos_pixel = coord.a_pixel(RADIO_HEXAGONO);

            let (material, terreno) = if q == 0 && r == 1 {
                (color_colina.clone(), TipoTerreno::Montana)
            } else {
                (color_llanura.clone(), TipoTerreno::Llanura)
            };

            // Guardamos la casilla en la memoria lógica del juego
            tablero.casillas.insert(coord, terreno);

            // Creamos la entidad visual en pantalla
            commands.spawn((
                CasillaVisual,
                Mesh2d(forma_hex.clone()),
                MeshMaterial2d(material),
                Transform::from_xyz(pos_pixel.x, pos_pixel.y, 0.0),
            ));
        }
    }
}

/// Sistema Update: Traduce el clic del ratón y envía la orden al sistema táctico
pub fn detectar_clic_tablero(
    botones_raton: Res<ButtonInput<MouseButton>>,
    q_ventana: Query<&Window, With<PrimaryWindow>>,
    q_camara: Query<(&Camera, &GlobalTransform)>,
    mut eventos_movimiento: EventWriter<EventoOrdenMovimiento>, // Emisor de órdenes
) {
    if botones_raton.just_pressed(MouseButton::Left) {
        let Ok(ventana) = q_ventana.single() else { return; };
        let Ok((camara, camara_transform)) = q_camara.single() else { return; };

        if let Some(pos_cursor) = ventana.cursor_position() {
            if let Ok(pos_mundo) = camara.viewport_to_world_2d(camara_transform, pos_cursor) {
                let hex_clicado = HexCoord::desde_pixel(pos_mundo, RADIO_HEXAGONO);
                
                // Disparamos la orden hacia el motor de reglas
                eventos_movimiento.send(EventoOrdenMovimiento {
                    destino: hex_clicado,
                });
            }
        }
    }
}
```

---

### 4. Registrar el Recurso y el Evento en `src/main.rs`
Por último, registra el nuevo recurso y el evento en la configuración de `App::new()` dentro de tu `src/main.rs`:

```rust
        // En App::new()
        .init_resource::<TesoreriaJugador>()
        .init_resource::<ColaIniciativa>()
        .init_resource::<TableroTactico>() // <-- Nuevo recurso del mapa
        .add_event::<EventoAtaque>()
        .add_event::<EventoUnidadEliminada>()
        .add_event::<EventoOrdenMovimiento>() // <-- Nuevo evento de movimiento
        // Y en Update añade detectar_clic_tablero:
        .add_systems(Update, (
            detectar_clic_tablero,
            sistema_movimiento_tactico,
            procesar_combate,
            gestionar_bajas,
            sincronizar_visuales_unidades,
        ))
```

---

### 5. Ejemplos Prácticos de Ejecución (Simulación de Turno)

Para entender cómo interactúan los sistemas fotograma a fotograma, veamos **4 escenarios reales** con tu Héroe situado inicialmente en la casilla central **`(0, 1, -1)`** con **`2 PA`**:

#### Caso 1: Movimiento Válido (Éxito)
* **Acción del jugador:** Hace clic en la casilla superior derecha `(1, 0, -1)`.
* **Proceso interno:**
  1. `detectar_clic_tablero` envía `EventoOrdenMovimiento { destino: (1, 0, -1) }`.
  2. `tablero.casillas.contains_key(&(1, 0, -1))` devuelve `true` (está dentro del mapa 3x3).
  3. `distancia_a` calcula la separación entre `(0, 1, -1)` y `(1, 0, -1)` ➔ devuelve **`1`**.
  4. El Héroe tiene `2 PA` (cumple `pa >= 1`).
* **Resultado:** La unidad cambia su `PosicionHex` a `(1, 0, -1)` y sus PA bajan a **`1`**. El sistema `sincronizar_visuales_unidades` detecta el cambio y traslada el círculo azul en pantalla.

#### Caso 2: Clic fuera del mapa (Rechazo por límites)
* **Acción del jugador:** Hace clic en el fondo negro de la pantalla, lejos de los 9 hexágonos dibujados.
* **Proceso interno:**
  1. La fórmula inversa traduce ese píxel a la coordenada matemática `(4, -2, -2)`.
  2. `tablero.casillas.contains_key(&(4, -2, -2))` devuelve **`false`**.
* **Resultado:** El código ejecuta `continue;`. La orden se descarta al instante, no se gastan Puntos de Acción y la unidad permanece quieta.

#### Caso 3: Clic demasiado lejos (Rechazo por distancia)
* **Acción del jugador:** Estando en `(0, 1, -1)`, hace clic directamente en la esquina superior izquierda `(0, 0, 0)` (que en este momento está a 2 saltos de distancia desde su posición actual).
* **Proceso interno:**
  1. La casilla existe en el tablero (`true`).
  2. `distancia_a` calcula la separación y devuelve **`2`**.
  3. La validación `if distancia != 1` se activa.
* **Resultado:** Se imprime el aviso en consola y la unidad no se mueve, obligando al jugador a avanzar paso a paso (o a implementar en el futuro un algoritmo de *Pathfinding* para rutas largas).

#### Caso 4: Agotar Puntos de Acción (Rechazo por fatiga de turno)
* **Acción del jugador:** Ya movió al Héroe dos veces en este turno (`PA actuales = 0`) e intenta hacer un tercer clic en una casilla vecina.
* **Proceso interno:**
  1. El destino existe y la distancia es `1`.
  2. Al evaluar `if pa.actuales >= 1`, la condición falla (`0 >= 1` es falso).
* **Resultado:** El motor bloquea el desplazamiento hasta que termine la ronda global y el sistema económico rellene los `PA` de todas las tropas.

---

### Claves Técnicas de esta Arquitectura
* **Seguridad contra datos corruptos:** Al centralizar la validación dentro de `sistema_movimiento_tactico`, es imposible que un clic rápido o un error de interfaz saque una ficha del tablero o gaste puntos negativos.
* **Reactividad automática:** El sistema de movimiento nunca toca el componente `Transform` (píxeles). Solo modifica `PosicionHex`. Esto mantiene la lógica del juego 100% independiente de los gráficos.
