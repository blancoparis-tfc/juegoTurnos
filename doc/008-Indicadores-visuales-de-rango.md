# 🎯 INDICADORES VISUALES DE RANGO (UX) EN BEVY: PROYECTO HEX-TACTICS

Implementar **indicadores visuales de rango (UX)** evita que el jugador tenga que adivinar o hacer clics al azar para saber a qué casillas puede moverse. Utilizaremos la capa intermedia de profundidad (**`Z = 5.0`**) para dibujar anillos semitransparentes sobre los hexágonos válidos.

El sistema está diseñado para ser **completamente reactivo**: solo borrará y volverá a dibujar los anillos cuando la unidad activa cambie de posición, gaste Puntos de Acción o avance el turno en la línea de tiempo.

---

## 🛠️ 1. Diseño del Sistema Visual de Rango (`src/render_system.rs`)

Añadiremos un componente marcador llamado `IndicadorRango` para identificar estos anillos en el mundo de Bevy y poder limpiarlos fácilmente cuando la unidad termine su movimiento.

Abre `src/render_system.rs` y añade el siguiente código:

```rust
use crate::game_systems::{PosicionHex, PuntosAccion, Unidad, ColaIniciativa, TableroTactico};

// Componente marcador para identificar y borrar los anillos de movimiento
#[derive(Component)]
pub struct IndicadorRango;

/// Sistema Update: Dibuja anillos sobre las casillas vecinas válidas de la unidad activa
pub fn actualizar_indicadores_rango(
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<ColorMaterial>>,
    // 1. Consultamos los indicadores antiguos para borrarlos
    q_indicadores_viejos: Query<Entity, With<IndicadorRango>>,
    // 2. Consultamos las unidades y detectamos si alguna cambió su posición o sus PA
    q_unidades: Query<(&PosicionHex, &PuntosAccion), With<Unidad>>,
    q_cambios: Query<Entity, (With<Unidad>, Or<(Changed<PosicionHex>, Changed<PuntosAccion>)>)>,
    cola: Res<ColaIniciativa>,
    tablero: Res<TableroTactico>,
) {
    // Optimización: Si ninguna unidad se movió ni gastó PA, y la cola no cambió, no hacemos nada
    if q_cambios.is_empty() && !cola.is_changed() {
        return;
    }

    // 3. Limpiamos todos los indicadores del paso anterior
    for entidad_vieja in q_indicadores_viejos.iter() {
        commands.entity(entidad_vieja).despawn();
    }

    if cola.orden_unidades.is_empty() { return; }

    // 4. Obtenemos la unidad que tiene el turno actualmente
    let entidad_activa = cola.orden_unidades[cola.indice_actual];

    if let Ok((pos_actual, pa)) = q_unidades.get(entidad_activa) {
        // Si no le quedan Puntos de Acción, no dibujamos ningún indicador
        if pa.actuales == 0 {
            return;
        }

        // Creamos la geometría de un anillo hexagonal (radio exterior, radio interior)
        let anillo_hex = meshes.add(Annulus::new(RADIO_HEXAGONO * 0.65, RADIO_HEXAGONO * 0.82));
        // Color cian brillante con transparencia (Alpha = 0.55)
        let color_valido = materials.add(Color::srgba(0.1, 0.85, 0.95, 0.55));

        // 5. Evaluamos los 6 vecinos directos usando tu lógica hexagonal
        for vecino in pos_actual.0.obtener_todos_los_vecinos() {
            // Solo dibujamos el indicador si la casilla existe dentro del mapa
            if tablero.casillas.contains_key(&vecino) {
                let pixel = vecino.a_pixel(RADIO_HEXAGONO);

                commands.spawn((
                    IndicadorRango,
                    Mesh2d(anillo_hex.clone()),
                    MeshMaterial2d(color_valido.clone()),
                    // Z = 5.0: Por encima del suelo (0.0) y por debajo de la ficha (10.0)
                    Transform::from_xyz(pixel.x, pixel.y, 5.0),
                ));
            }
        }
    }
}
```

---

## 🔌 2. Registrar el Sistema en `src/main.rs`

Añade `actualizar_indicadores_rango` dentro de tu bucle principal `.add_systems(Update, (...))` en `src/main.rs`:

```rust
        .add_systems(Update, (
            detectar_clic_tablero,
            sistema_movimiento_tactico,
            procesar_combate,
            gestionar_bajas,
            sincronizar_visuales_unidades,
            actualizar_indicadores_rango, // <-- Nuevo sistema visual UX
        ))
```

---

## 🎮 3. Ejemplos Prácticos en Pantalla (Comportamiento UX)

Para visualizar cómo responde este código durante una partida real en tu mapa de 3x3, analiza estos **3 escenarios**:

### Escenario A: Unidad en el centro del mapa `(0, 1, -1)` con `2 PA`
* **Estado:** Tu Héroe está rodeado por casillas válidas por los 6 costados.
* **Resultado visual:** El bucle `obtener_todos_los_vecinos()` devuelve 6 coordenadas. Todas existen en `tablero.casillas`. En pantalla aparecen **6 anillos cian brillantes** rodeando al Héroe, indicando que puede avanzar en cualquier dirección.

### Escenario B: Unidad desplazada a una esquina `(0, 0, 0)` con `1 PA`
* **Estado:** El jugador hace clic hacia la esquina superior izquierda. El Héroe se mueve a `(0, 0, 0)` y gasta `1 PA` (le queda `1 PA`).
* **Resultado visual:**
  1. El filtro `Changed<PosicionHex>` detecta el movimiento y borra los 6 anillos anteriores.
  2. Evalúa los 6 vecinos de `(0, 0, 0)`. Tres de ellos (`(-1, 0, 1)`, `(0, -1, 1)` y `(1, -1, 0)`) caen fuera del mapa de 3x3, por lo que `tablero.casillas.contains_key()` devuelve `false` y los descarta.
  3. Solo se dibujan **3 anillos cian** hacia el interior del tablero (`(1, 0, -1)`, `(0, 1, -1)` y `(-1, 1, 0)`), evitando que el jugador intente hacer clics inválidos fuera del borde.

### Escenario C: Unidad sin Puntos de Acción (`0 PA`)
* **Estado:** El jugador realiza un segundo movimiento y los `PuntosAccion` bajan a `0`.
* **Resultado visual:** El filtro `Changed<PuntosAccion>` dispara el sistema. Se borran los anillos anteriores y, al llegar a `if pa.actuales == 0 { return; }`, la función termina sin dibujar nada nuevo. La desaparición instantánea de las guías visuales comunica al jugador sin necesidad de texto que esa unidad ha agotado su movimiento.

---

## 🧠 4. Claves Técnicas de esta Implementación

* **Primitiva `Annulus` de Bevy:** En lugar de cargar una textura PNG externa para el borde de selección, utilizamos la forma nativa `Annulus` (corona circular). Al darle un radio interior del 65% y exterior del 82%, deja ver el color del terreno que hay debajo (llanura o colina) mientras resalta la casilla.
* **Transparencia (`Color::srgba`):** El cuarto parámetro (`0.55`) controla el canal Alfa (opacidad). Bevy mezcla automáticamente el color cian con el verde o marrón del hexágono inferior.
* **Cero desperdicio de CPU:** Gracias a la combinación `Or<(Changed<PosicionHex>, Changed<PuntosAccion>)>` y `!cola.is_changed()`, este sistema no destruye ni crea entidades en los fotogramas donde el jugador simplemente está pensando su jugada.
