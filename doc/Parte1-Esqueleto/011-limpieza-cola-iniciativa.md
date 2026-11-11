# 🗑️ LIMPIEZA DE COLA DE INICIATIVA EN BEVY (RUST): PROYECTO HEX-TACTICS

Cuando una unidad es eliminada en combate y desaparece del tablero, su identificador (`Entity`) sigue guardado dentro del vector `ColaIniciativa`. Si no la retiramos de esa lista de forma segura, el juego puede sufrir un error crítico de índice fuera de rango (*index out of bounds*) o quedarse bloqueado intentando darle el turno a una unidad muerta.

Para resolverlo, implementamos un sistema reactivo que escucha el evento `EventoUnidadEliminada`, retira la entidad de la lista y reajusta el puntero `indice_actual` según la posición en la que estaba la víctima.

---

## 🛠️ 1. El Sistema de Limpieza de Cola (`src/game_systems.rs`)

Añade este sistema en tu archivo `src/game_systems.rs` e importa `EventoUnidadEliminada` desde tu módulo de combate:

```rust
use crate::combat_system::EventoUnidadEliminada;

/// Sistema Update: Retira de la Cola de Iniciativa a las unidades eliminadas
/// y reajusta el índice del turno actual para evitar saltos o bloqueos.
pub fn limpiar_cola_iniciativa(
    mut eventos_muerte: EventReader<EventoUnidadEliminada>,
    mut eventos_fin_ronda: EventWriter<EventoFinRondaGlobal>,
    mut cola: ResMut<ColaIniciativa>,
) {
    for evento in eventos_muerte.read() {
        // 1. Buscamos en qué posición de la fila estaba la unidad derrotada
        if let Some(posicion_victima) = cola.orden_unidades.iter().position(|&e| e == evento.victima) {
            
            // 2. La expulsamos de la lista de iniciativa
            cola.orden_unidades.remove(posicion_victima);
            println!("🗑️ Unidad {:?} retirada de la cola de iniciativa.", evento.victima);

            // Si ya no queda nadie vivo en el mapa, detenemos el ajuste
            if cola.orden_unidades.is_empty() {
                cola.indice_actual = 0;
                return;
            }

            // 3. Ajustamos el puntero del turno actual según dónde estaba la víctima
            if posicion_victima < cola.indice_actual {
                // La víctima iba ANTES que la unidad activa:
                // Al borrarse, toda la lista se corre un lugar a la izquierda,
                // así que restamos 1 al índice para seguir apuntando a la unidad correcta.
                cola.indice_actual -= 1;
            } else if cola.indice_actual >= cola.orden_unidades.len() {
                // La víctima era la ÚLTIMA de la lista y era su turno (o el índice quedó fuera):
                // Reiniciamos la cola al principio y disparamos el cierre de ronda global.
                cola.indice_actual = 0;
                println!("🔄 Última unidad de la cola eliminada. Cerrando ronda global...");
                eventos_fin_ronda.send(EventoFinRondaGlobal);
            }
        }
    }
}
```

---

## 🔌 2. Registrar el Sistema en `src/main.rs`

Añade `limpiar_cola_iniciativa` dentro de tu bucle principal `.add_systems(Update, (...))` en `src/main.rs`:

```rust
        .add_systems(Update, (
            detectar_clic_tablero,
            detectar_input_fin_turno,
            gestionar_avance_turno,
            sistema_movimiento_tactico,
            sistema_economia_ronda,
            procesar_combate,
            limpiar_cola_iniciativa, // <-- Nuevo sistema de limpieza de turnos
            gestionar_bajas,
            sincronizar_visuales_unidades,
            actualizar_indicadores_rango,
        ))
```

---

## 🧠 3. Ejemplos Prácticos de Ajuste de Índices

Imagina que en la cola tenemos tres unidades ordenadas por velocidad:
**`[ Héroe (Índice 0), Orco (Índice 1), Soldado (Índice 2) ]`**

* **Caso 1: El Héroe (Índice 0) mata al Orco (Índice 1):**
  * `indice_actual` es `0`. `posicion_victima` es `1`.
  * Al borrar al Orco, la lista pasa a ser `[ Héroe (0), Soldado (1) ]`.
  * Como `1 < 0` es falso, `indice_actual` se mantiene en **`0`**. El Héroe sigue con su turno y, al pulsar `Espacio`, el turno saltará al Soldado (que ahora ocupa el índice 1) sin saltárselo.

* **Caso 2: El Soldado (Índice 2) mata al Héroe (Índice 0):**
  * `indice_actual` es `2` (es el turno del Soldado). `posicion_victima` es `0`.
  * Al borrar al Héroe, la lista se encoge a `[ Orco (0), Soldado (1) ]`. El Soldado se ha desplazado al índice `1`.
  * Como `posicion_victima < indice_actual` (`0 < 2` es verdadero), el sistema ejecuta `indice_actual -= 1`, dejando `indice_actual = 1`. El Soldado conserva su turno activo sin provocar un fallo de memoria.

* **Caso 3: Una trampa o contraataque mata al Soldado (Índice 2) durante su propio turno:**
  * `indice_actual` es `2`. `posicion_victima` es `2`.
  * Al borrar al Soldado, la lista queda con longitud `2` (`[ Héroe (0), Orco (1) ]`).
  * Como `indice_actual >= len()` (`2 >= 2`), el sistema reinicia `indice_actual = 0` (devolviendo el turno al Héroe) y emite `EventoFinRondaGlobal` para recargar los Puntos de Acción de los supervivientes.
