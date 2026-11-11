# 🎖️ SISTEMA DE SUBIDA DE NIVEL (LEVEL UP) EN BEVY (RUST): PROYECTO HEX-TACTICS

Este documento detalla la implementación del **Sistema de Subida de Nivel (Level Up)** en **Hex-Tactics**. Utiliza el filtro reactivo `Changed<Experiencia>` de **Bevy** para detectar el instante exacto en que una tropa supera su umbral de experiencia tras combatir, aplicando mejoras permanentes de vida máxima, daño y defensa, junto con una curación táctica en el campo de batalla.

---

## 📊 1. Estructura y Umbrales de Experiencia (`src/combat_system.rs`)

Definimos el componente `Experiencia` con una fórmula escalonada predecible: **`100 * (nivel + 1)`** (Nivel 0 ➔ 1 requiere 100 XP; Nivel 1 ➔ 2 requiere 200 XP, etc.), además del evento `EventoSubidaNivel` para notificar a la interfaz o efectos visuales.

Añade o actualiza esto en `src/combat_system.rs`:

```rust
#[derive(Component, Debug, Default)]
pub struct Experiencia {
    pub xp_actual: u32,
    pub nivel: u32,
}

impl Experiencia {
    /// Calcula la experiencia necesaria para alcanzar el siguiente nivel
    pub fn xp_para_siguiente_nivel(&self) -> u32 {
        100 * (self.nivel + 1)
    }
}

/// Evento que se dispara cuando una unidad sube de nivel (útil para efectos visuales o UI)
#[derive(Event, Debug)]
pub struct EventoSubidaNivel {
    pub entidad: Entity,
    pub nuevo_nivel: u32,
}
```

---

## ⚙️ 2. El Sistema Reactivo de Ascenso (`src/combat_system.rs`)

Gracias al filtro **`Changed<Experiencia>`**, este sistema no consume recursos de CPU en los fotogramas donde no hay combates. Solo se ejecuta cuando una unidad recibe experiencia.

Añade esta función en `src/combat_system.rs`:

```rust
use crate::game_systems::{Estadisticas, EconomiaUnidad, Unidad};

/// Sistema Update: Comprueba si alguna unidad superó su umbral de XP y aplica mejoras permanentes
pub fn procesar_subida_nivel(
    mut query: Query<(
        Entity,
        &mut Experiencia,
        &mut Estadisticas,
        &mut PerfilCombate,
        &EconomiaUnidad,
    ), (With<Unidad>, Changed<Experiencia>)>,
    mut eventos_nivel: EventWriter<EventoSubidaNivel>,
) {
    for (entidad, mut xp, mut stats, mut perfil, econ) in query.iter_mut() {
        let mut umbral = xp.xp_para_siguiente_nivel();

        // Usamos un bucle 'while' por si una unidad gana tanta XP de golpe que sube 2 niveles
        while xp.xp_actual >= umbral {
            xp.xp_actual -= umbral;
            xp.nivel += 1;

            // Calculamos las mejoras permanentes según si es Héroe o Tropa Regular
            let (bono_vida, bono_dano, bono_defensa) = if econ.es_heroe {
                (15, 4, 2) // Los Héroes crecen más rápido
            } else {
                (10, 3, 1) // Los Veteranos regulares tienen un crecimiento sólido
            };

            // Aplicamos las mejoras a las estadísticas base
            stats.vida_maxima += bono_vida;
            // Curamos la vida ganada más un extra de adrenalina en combate (+10 PV)
            stats.vida_actual = (stats.vida_actual + bono_vida + 10).min(stats.vida_maxima);
            
            perfil.dano_base += bono_dano;
            perfil.defensa_base += bono_defensa;

            println!(
                "🎉 ¡SUBIDA DE NIVEL! Unidad {:?} alcanza el Nivel {}! (+{} Vida Máx, +{} Daño, +{} Defensa)",
                entidad, xp.nivel, bono_vida, bono_dano, bono_defensa
            );

            // Emitimos el evento por si la interfaz gráfica quiere mostrar un texto flotante
            eventos_nivel.send(EventoSubidaNivel {
                entidad,
                nuevo_nivel: xp.nivel,
            });

            // Recalculamos el umbral para el nuevo nivel
            umbral = xp.xp_para_siguiente_nivel();
        }
    }
}
```

---

## 🔌 3. Registrar el Evento y el Sistema en `src/main.rs`

Registra el evento `EventoSubidaNivel` y añade `procesar_subida_nivel` dentro del bucle principal `Update` (justo después de `procesar_combate`) en `src/main.rs`:

```rust
        // En App::new():
        .add_event::<EventoSubidaNivel>()

        // En Update:
        .add_systems(Update, (
            detectar_clic_tablero,
            detectar_input_fin_turno,
            gestionar_avance_turno,
            sistema_movimiento_tactico,
            sistema_economia_ronda,
            procesar_combate,
            procesar_subida_nivel, // <-- Evalúa ascensos inmediatamente después del combate
            limpiar_cola_iniciativa,
            gestionar_bajas,
            sincronizar_visuales_unidades,
            actualizar_indicadores_rango,
            actualizar_entorno_ronda,
            detectar_input_guardado,
            sistema_guardar_campana,
            sistema_cargar_campana,
        ))
```

---

## 🎮 4. Ejemplos Prácticos de Progresión en Campaña

1. **Acumulación en Batalla:**
   * Tu **Arquero Recluta** empieza en **Nivel 0** con `60 PV`, `15 Daño`, `5 Defensa` y `0/100 XP`.
   * Durante la primera batalla dispara dos veces (`+10 XP` por impacto) y remata a dos enemigos (`+50 XP` por cada baja), alcanzando **`120 XP`**.
2. **Ascenso Instantáneo:**
   * En el mismo fotograma del segundo golpe mortal, `procesar_subida_nivel` detecta que `120 >= 100`.
   * Resta `100 XP` (dejando **`20 XP`** sobrantes para el siguiente nivel) y sube al **Nivel 1**.
   * Sus estadísticas permanentes suben a **`70 PV Máx`**, **`18 Daño`** y **`6 Defensa`**, y recupera hasta `20 PV` en el acto.
3. **Persistencia Permanente (`F5`):**
   * Al terminar la batalla y pulsar **`F5`**, `sistema_guardar_campana` lee los componentes actualizados de la entidad y escribe en `campana_guardada.ron` el nuevo `nivel: 1`, `dano_base: 18` y `vida_maxima: 70`. En el siguiente mapa desplegará ya convertido en un veterano más letal.
