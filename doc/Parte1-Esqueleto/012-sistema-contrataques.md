# ⚔️ SISTEMA DE CONTRAATAQUES EN BEVY (RUST): PROYECTO HEX-TACTICS

El **Sistema de Contraataques** añade profundidad táctica a los enfrentamientos cuerpo a cuerpo. Si una unidad ataca a un enemigo adyacente y no logra derrotarlo de un solo golpe, el defensor devuelve el ataque automáticamente sin gastar sus propios Puntos de Acción (PA).

Para evitar bucles infinitos de respuestas (donde A golpea a B, B contraataca a A, A contraataca a B, etc.), introducimos una bandera booleana **`es_contraataque`** dentro de `EventoAtaque`. Si el impacto recibido ya es un contraataque, la cadena de respuestas se detiene de inmediato.

---

## 1. 🛠️ Actualizar la Estructura del Evento (`src/combat_system.rs`)

Añade el campo `es_contraataque: bool` al struct `EventoAtaque` en `src/combat_system.rs`. Además, aplicaremos una penalización táctica (los contraataques infligen un **75% del daño normal**) para recompensar al jugador que toma la iniciativa.

```rust
#[derive(Event, Debug)]
pub struct EventoAtaque {
    pub atacante: Entity,
    pub defensor: Entity,
    pub es_contraataque: bool, // Evita bucles infinitos de respuestas
}
```

*(Nota: Recuerda actualizar el punto donde se emite este evento dentro de `sistema_movimiento_tactico` en `src/game_systems.rs`, añadiendo `es_contraataque: false`).*

---

## 2. 🧠 Lógica de Contraataque y Emisión Diferida (`src/combat_system.rs`)

En Bevy, un mismo sistema no puede pedir simultáneamente lectura (`EventReader<EventoAtaque>`) y escritura (`EventWriter<EventoAtaque>`) del mismo tipo de evento. 

La solución idiomática en Bevy es inyectar **`mut commands: Commands`** en el sistema para encolar el contraataque mediante `commands.queue(...)`, ejecutándolo de forma limpia en el siguiente fotograma sin conflictos de memoria.

Sustituye la función `procesar_combate` en `src/combat_system.rs` por esta versión completa:

```rust
/// Sistema A: Escucha los eventos de ataque, aplica bonos y dispara contraataques si el defensor sobrevive
pub fn procesar_combate(
    mut commands: Commands, // Permite encolar eventos diferidos de forma segura
    mut eventos_ataque: EventReader<EventoAtaque>,
    mut eventos_muerte: EventWriter<EventoUnidadEliminada>,
    query_info: Query<(&PosicionHex, &PerfilCombate, &Elevacion), With<Unidad>>,
    mut query_estado: Query<(&mut Estadisticas, &mut Experiencia), With<Unidad>>,
) {
    for evento in eventos_ataque.read() {
        let Ok((pos_atk, perfil_atk, elev_atk)) = query_info.get(evento.atacante) else { continue; };
        let Ok((pos_def, perfil_def, elev_def)) = query_info.get(evento.defensor) else { continue; };

        if evento.es_contraataque {
            println!("🔄 ¡CONTRAATAQUE! {:?} responde al golpe de {:?}", evento.atacante, evento.defensor);
        }

        let mut multiplicador_dano = 1.0;

        // 1. Bono del Triángulo de Combate (+25% daño)
        if tiene_ventaja_rasgo(&perfil_atk.rasgos, &perfil_def.rasgos) {
            multiplicador_dano += 0.25;
            println!("⚔️ ¡Ventaja de rasgo aplicada! (+25%)");
        }

        // 2. Bono de Altura (+15% si ataca desde arriba)
        if (*elev_atk as i32) > (*elev_def as i32) {
            multiplicador_dano += 0.15;
            println!("⛰️ ¡Bono de altura aplicado! (+15%)");
        }

        // 3. Penalización por Contraataque (-25% de potencia por defenderse de improviso)
        if evento.es_contraataque {
            multiplicador_dano *= 0.75;
        }

        // 4. Flanqueo Colectivo (Solo aplica en ataques directos, no en contraataques)
        let flanqueado = if !evento.es_contraataque {
            verificar_flanqueo_colectivo(
                pos_def.0, 
                perfil_atk.bando_jugador, 
                &perfil_atk.rasgos, 
                &query_info
            )
        } else {
            false
        };

        let defensa_efectiva = if flanqueado {
            println!("🛡️💥 ¡Ataque Combinado! Defensa del enemigo anulada.");
            0
        } else {
            perfil_def.defensa_base
        };

        // Cálculo final determinista
        let dano_bruto = (perfil_atk.dano_base as f32 * multiplicador_dano).round() as i32;
        let dano_final = (dano_bruto - defensa_efectiva).max(1);

        // Aplicamos el daño al defensor
        let mut murio = false;
        if let Ok((mut stats_def, _)) = query_estado.get_mut(evento.defensor) {
            stats_def.vida_actual -= dano_final;
            println!("💥 Daño infligido: {}. Vida restante de {:?}: {}", dano_final, evento.defensor, stats_def.vida_actual);
            if stats_def.vida_actual <= 0 {
                murio = true;
            }
        }

        // Otorgamos experiencia (XP) al atacante
        if let Ok((_, mut xp_atk)) = query_estado.get_mut(evento.atacante) {
            if murio {
                xp_atk.xp_actual += 50;
                println!("⭐ ¡Enemigo eliminado! +50 XP para {:?}", evento.atacante);
                eventos_muerte.send(EventoUnidadEliminada {
                    victima: evento.defensor,
                    asesino: evento.atacante,
                });
            } else {
                xp_atk.xp_actual += 10;
            }
        }

        // 5. EVALUACIÓN DE CONTRAATAQUE
        // Condiciones: El defensor sobrevivió, el golpe actual NO era ya un contraataque,
        // y ambos están a distancia cuerpo a cuerpo (1 casilla).
        let distancia = pos_atk.0.distancia_a(&pos_def.0);
        if !murio && !evento.es_contraataque && distancia == 1 {
            let nuevo_atacante = evento.defensor;
            let nuevo_defensor = evento.atacante;

            // Encolamos el evento de contraataque para el siguiente fotograma
            commands.queue(move |world: &mut World| {
                world.send_event(EventoAtaque {
                    atacante: nuevo_atacante,
                    defensor: nuevo_defensor,
                    es_contraataque: true,
                });
            });
        }
    }
}
```

---

## 3. 🎮 Ejemplos Prácticos en Ejecución

Así se resuelve un intercambio de golpes en tu tablero de 3x3 con el **Héroe (100 PV, Daño 25)** atacando al **Orco (50 PV, Daño 18)**:

1. **Primer Golpe (Fotograma 1):**
   * Haces clic sobre el Orco. Se envía `EventoAtaque` con `es_contraataque: false`.
   * Tu Héroe golpea con ventaja de rasgo (+25%) y altura (+15%), infligiendo **30 puntos de daño**.
   * El Orco queda con **20 PV** (`murio = false`). Como está a distancia `1`, `commands.queue` programa el contraataque.

2. **Respuesta Automática (Fotograma 2):**
   * Bevy ejecuta el evento encolado con `es_contraataque: true`.
   * El Orco devuelve el golpe con un multiplicador del `0.75` (penalización por contraataque) y sin bono de flanqueo, infligiendo unos **4 puntos de daño** tras restar la defensa de tu Héroe.
   * Tu Héroe baja de `100 PV` a `96 PV` y el Orco gana `+10 XP`.
   * Al evaluar el paso 5, como `evento.es_contraataque` es `true`, el sistema **no programa un segundo contraataque**, cerrando el intercambio de golpes.

3. **Golpe de Gracia (Fotograma 3):**
   * Haces un segundo clic sobre el Orco herido (le quedan `20 PV`).
   * Tu Héroe inflige otros **30 puntos de daño**, dejando al Orco en `-10 PV` (`murio = true`).
   * Como `murio` es verdadero, el Orco **no puede contraatacar**, muere al instante y tu Héroe sale ileso del segundo intercambio.
