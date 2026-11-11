# 🌍 BONOS DE TERRENO, CICLO DÍA/NOCHE Y CLIMA EN BEVY (RUST): PROYECTO HEX-TACTICS

Este documento detalla la implementación del **Sistema Ambiental** en **Hex-Tactics**. Conecta el estado atmosférico del mapa (**Día/Noche** y **Clima**) y el tipo de terreno de cada casilla directamente con las fórmulas de combate, el alcance efectivo de los ataques a distancia y la retroalimentación visual.

---

## 🛠️ 1. Recursos Ambientales y Tipos de Terreno (`src/environment_system.rs`)

Crea el archivo `src/environment_system.rs` para definir el estado atmosférico del mundo (`MomentoDia` y `ClimaActual`), el contador de rondas globales y los modificadores tácticos de cada terreno.

```rust
use bevy::prelude::*;
use crate::hex_grid::TipoTerreno;
use crate::game_systems::EventoFinRondaGlobal;

// ==========================================
// 1. ESTADOS AMBIENTALES (Día/Noche y Clima)
// ==========================================

#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum MomentoDia {
    #[default]
    Dia,
    Noche,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum ClimaActual {
    #[default]
    Despejado,
    Lluvia,
    Niebla,
}

#[derive(Resource, Debug)]
pub struct EntornoBatalla {
    pub ronda_actual: u32,
    pub momento: MomentoDia,
    pub clima: ClimaActual,
}

impl Default for EntornoBatalla {
    fn default() -> Self {
        Self {
            ronda_actual: 1,
            momento: MomentoDia::Dia,
            clima: ClimaActual::Despejado,
        }
    }
}

// ==========================================
// 2. REGLAS DE MODIFICADORES AMBIENTALES
// ==========================================

impl EntornoBatalla {
    /// Calcula el alcance efectivo de las unidades a distancia según el clima y la hora
    pub fn calcular_rango_efectivo(&self, rango_base: u32) -> u32 {
        // Las unidades cuerpo a cuerpo (rango 1) nunca ven reducido su alcance
        if rango_base <= 1 {
            return 1;
        }

        // De Noche o con Niebla densa, la visibilidad cae y los arqueros solo disparan a 1 casilla
        if self.momento == MomentoDia::Noche || self.clima == ClimaActual::Niebla {
            return 1;
        }

        rango_base
    }

    /// Devuelve el multiplicador de daño para unidades con rasgo Distancia
    pub fn multiplicador_dano_distancia(&self) -> f32 {
        match self.clima {
            ClimaActual::Lluvia => 0.80, // -20% de daño por cuerdas de arco mojadas y viento
            _ => 1.0,
        }
    }
}

/// Devuelve el bono plano de defensa que otorga el tipo de terreno donde pisa el defensor
pub fn obtener_bono_defensa_terreno(terreno: TipoTerreno) -> i32 {
    match terreno {
        TipoTerreno::Llanura => 0,
        TipoTerreno::Bosque => 5,   // +5 Defensa por cobertura arbórea
        TipoTerreno::Montana => 8,  // +8 Defensa por posición fortificada en roca
        TipoTerreno::Agua => -3,    // -3 Defensa por vulnerabilidad en vado/agua
    }
}

// ==========================================
// 3. SISTEMA DE AVANCE AMBIENTAL POR RONDA
// ==========================================

/// Sistema Update: Avanza el ciclo Día/Noche cada 3 rondas globales
pub fn actualizar_entorno_ronda(
    mut eventos_fin_ronda: EventReader<EventoFinRondaGlobal>,
    mut entorno: ResMut<EntornoBatalla>,
) {
    for _ in eventos_fin_ronda.read() {
        entorno.ronda_actual += 1;

        // Cada 3 rondas alternamos entre Día y Noche
        if (entorno.ronda_actual - 1) % 3 == 0 {
            entorno.momento = match entorno.momento {
                MomentoDia::Dia => {
                    println!("🌙 ¡Cae la NOCHE! El alcance de los ataques a distancia se reduce a 1.");
                    MomentoDia::Noche
                }
                MomentoDia::Noche => {
                    println!("☀️ ¡Amanece! Se restaura la visibilidad normal en el tablero.");
                    MomentoDia::Dia
                }
            };
        }

        println!(
            "🌍 Ronda {} | Momento: {:?} | Clima: {:?}",
            entorno.ronda_actual, entorno.momento, entorno.clima
        );
    }
}
```

---

## ⚔️ 2. Aplicar Terreno y Clima en el Combate (`src/combat_system.rs`)

Actualizamos `procesar_combate` para que consulte el recurso `TableroTactico` (para saber qué terreno pisa el defensor) y `EntornoBatalla` (para aplicar la penalización de lluvia a los disparos a distancia).

```rust
use crate::game_systems::{PosicionHex, Estadisticas, Unidad, TableroTactico};
use crate::environment_system::{EntornoBatalla, obtener_bono_defensa_terreno};

pub fn procesar_combate(
    mut commands: Commands,
    mut eventos_ataque: EventReader<EventoAtaque>,
    mut eventos_muerte: EventWriter<EventoUnidadEliminada>,
    query_info: Query<(&PosicionHex, &PerfilCombate, &Elevacion), With<Unidad>>,
    mut query_estado: Query<(&mut Estadisticas, &mut Experiencia), With<Unidad>>,
    tablero: Res<TableroTactico>,   // <-- Para consultar el terreno del defensor
    entorno: Res<EntornoBatalla>,    // <-- Para consultar lluvia/noche
) {
    for evento in eventos_ataque.read() {
        let Ok((pos_atk, perfil_atk, elev_atk)) = query_info.get(evento.atacante) else { continue; };
        let Ok((pos_def, perfil_def, elev_def)) = query_info.get(evento.defensor) else { continue; };

        let mut multiplicador_dano = 1.0;

        // 1. Bono del Triángulo de Combate (+25%)
        if tiene_ventaja_rasgo(&perfil_atk.rasgos, &perfil_def.rasgos) {
            multiplicador_dano += 0.25;
        }

        // 2. Bono de Altura (+15%)
        if (*elev_atk as i32) > (*elev_def as i32) {
            multiplicador_dano += 0.15;
        }

        // 3. Penalización climática para ataques a distancia (Lluvia = -20%)
        if perfil_atk.rasgos.contains(&RasgoCombate::Distancia) {
            multiplicador_dano *= entorno.multiplicador_dano_distancia();
        }

        // 4. Penalización por Contraataque (-25%)
        if evento.es_contraataque {
            multiplicador_dano *= 0.75;
        }

        // 5. Cálculo de Defensa: Defensa Base + Bono de Terreno (Bosque/Montaña)
        let terreno_defensor = tablero
            .casillas
            .get(&pos_def.0)
            .copied()
            .unwrap_or(crate::hex_grid::TipoTerreno::Llanura);
            
        let bono_terreno = obtener_bono_defensa_terreno(terreno_defensor);

        let flanqueado = !evento.es_contraataque && verificar_flanqueo_colectivo(
            pos_def.0, 
            perfil_atk.bando_jugador, 
            &perfil_atk.rasgos, 
            &query_info
        );

        let defensa_efectiva = if flanqueado {
            println!("🛡️💥 ¡Ataque Combinado! Anula defensa base y cobertura de {:?}.", terreno_defensor);
            0 // El flanqueo anula tanto la armadura como la cobertura del bosque
        } else {
            (perfil_def.defensa_base + bono_terreno).max(0)
        };

        // ... el resto de la aplicación de daño, XP y contraataque se mantiene igual ...
```

---

## 👁️ 3. Sincronizar el Alcance Visual y Lógico con la Noche

Para que tanto la validación del clic del ratón (`sistema_movimiento_tactico` en `src/game_systems.rs`) como los anillos rojos (`actualizar_indicadores_rango` en `src/render_system.rs`) respeten la reducción de visibilidad nocturna:

* **En `src/game_systems.rs`:** Añade `entorno: Res<EntornoBatalla>` a los parámetros de `sistema_movimiento_tactico` y sustituye la validación de alcance por:
  ```rust
  let rango_efectivo = entorno.calcular_rango_efectivo(rango_activo);
  if distancia > rango_efectivo {
      println!("🌙 Objetivo fuera de visión/alcance actual (Rango efectivo: {})", rango_efectivo);
      continue;
  }
  ```

* **En `src/render_system.rs`:** Añade `entorno: Res<EntornoBatalla>` a `actualizar_indicadores_rango` (e incluye `&& !entorno.is_changed()` en la comprobación inicial de cambios). Pasa `entorno.calcular_rango_efectivo(perfil_activo.rango_ataque)` al llamar a `obtener_casillas_en_rango(...)`.

---

## 🔌 4. Registrar el Módulo en `src/main.rs`

Registra el recurso global y el sistema ambiental dentro de `App::new()` en `src/main.rs`:

```rust
mod environment_system;
use environment_system::*;

// Dentro de App::new():
.init_resource::<EntornoBatalla>()

// En Update:
.add_systems(Update, (
    // ... tus sistemas existentes ...
    actualizar_entorno_ronda,
))
```

---

## 🎮 5. Ejemplos Prácticos en Ejecución

* **Cobertura en Bosque y Sinergia de Flanqueo:** Si el Orco enemigo se refugia en una casilla de `Bosque`, su defensa sube de `5` a `10` (`+5` por cobertura de árboles). Sin embargo, si rodeas al Orco con dos unidades aliadas de distinto rasgo, el **Ataque Combinado** anula por completo tanto la armadura como la cobertura del bosque (`defensa_efectiva = 0`).
* **Emboscada Nocturna:** Al llegar a la **Ronda 4**, `actualizar_entorno_ronda` cambia el estado a `MomentoDia::Noche`. Automáticamente, el Soldado Arquero ve reducido su `rango_efectivo` de `2` a `1`. Los anillos rojos a 2 casillas desaparecen al instante de la pantalla, obligando al arquero a acercarse o esperar al amanecer en la Ronda 7.
