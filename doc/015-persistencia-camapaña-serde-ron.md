# 💾 PERSISTENCIA DE CAMPAÑA CON SERDE Y RON EN BEVY (RUST): PROYECTO HEX-TACTICS

Este documento detalla la implementación del **Sistema de Persistencia de Campaña** en **Hex-Tactics** utilizando las librerías **Serde** y **RON** (*Rusty Object Notation*). Permite extraer el estado de las unidades supervivientes del jugador (vida, experiencia, nivel, fatiga y rasgos) y la tesorería para guardarlos en un archivo de texto legible en disco duro (`campana_guardada.ron`) y restaurarlos entre batallas.

---

## 🛠️ 1. Estructura de Datos de Campaña (`src/save_system.rs`)

Crea el archivo `src/save_system.rs` para definir las estructuras serializables que representan el progreso persistente entre mapas y los eventos de guardado/carga.

```rust
use bevy::prelude::*;
use serde::{Deserialize, Serialize};
use std::fs;
use crate::combat_system::{PerfilCombate, Experiencia, RasgoCombate};
use crate::game_systems::{Estadisticas, EconomiaUnidad, TesoreriaJugador, Unidad};

const RUTA_GUARDADO: &str = "campana_guardada.ron";

// ==========================================
// 1. ESTRUCTURAS SERIALIZABLES (Disco Duro)
// ==========================================

#[derive(Serialize, Deserialize, Debug, Clone)]
pub struct VeteranoGuardado {
    pub nombre: String,
    pub es_heroe: bool,
    pub vida_actual: i32,
    pub vida_maxima: i32,
    pub velocidad: i32,
    pub dano_base: i32,
    pub defensa_base: i32,
    pub rango_ataque: u32,
    pub rasgos: Vec<RasgoCombate>,
    pub xp_actual: u32,
    pub nivel: u32,
    pub fatiga: u32, // 0 = Fresco, >0 = Penalización si despliega sin descansar
}

#[derive(Resource, Serialize, Deserialize, Debug, Clone, Default)]
pub struct DatosCampana {
    pub nivel_mapa_actual: u32,
    pub oro_tesoreria: i32,
    pub reserva_veteranos: Vec<VeteranoGuardado>,
}

// ==========================================
// 2. EVENTOS DE GUARDADO Y CARGA
// ==========================================

#[derive(Event, Debug)]
pub struct EventoGuardarCampana;

#[derive(Event, Debug)]
pub struct EventoCargarCampana;
```

*(Nota: Asegúrate de añadir `Serialize, Deserialize` al `#[derive(...)]` de `RasgoCombate` en `src/combat_system.rs` e importar `use serde::{Serialize, Deserialize};`).*

---

## 📂 2. Sistemas para Escribir y Leer en Disco (`src/save_system.rs`)

Añade los sistemas que escuchan las teclas rápidas (**`F5` para Guardar** y **`F9` para Cargar**) y traducen entre el mundo activo de Bevy y el archivo `.ron`.

```rust
// ==========================================
// 3. SISTEMAS DE PERSISTENCIA
// ==========================================

/// Detecta teclas rápidas F5 (Guardar) y F9 (Cargar)
pub fn detectar_input_guardado(
    teclado: Res<ButtonInput<KeyCode>>,
    mut ev_guardar: EventWriter<EventoGuardarCampana>,
    mut ev_cargar: EventWriter<EventoCargarCampana>,
) {
    if teclado.just_pressed(KeyCode::F5) {
        ev_guardar.send(EventoGuardarCampana);
    }
    if teclado.just_pressed(KeyCode::F9) {
        ev_cargar.send(EventoCargarCampana);
    }
}

/// Extrae las unidades supervivientes del jugador y las guarda en campana_guardada.ron
pub fn sistema_guardar_campana(
    mut eventos: EventReader<EventoGuardarCampana>,
    mut campana: ResMut<DatosCampana>,
    tesoreria: Res<TesoreriaJugador>,
    query_unidades: Query<(
        &Estadisticas,
        &PerfilCombate,
        &Experiencia,
        &EconomiaUnidad,
    ), With<Unidad>>,
) {
    for _ in eventos.read() {
        campana.oro_tesoreria = tesoreria.dinero;
        campana.reserva_veteranos.clear();

        let mut contador = 1;
        for (stats, perfil, xp, econ) in query_unidades.iter() {
            // Solo guardamos las tropas pertenecientes al bando del jugador
            if !perfil.bando_jugador {
                continue;
            }

            let nombre_tropa = if econ.es_heroe {
                "Comandante (Héroe)".to_string()
            } else {
                format!("Veterano #{}", contador)
            };
            contador += 1;

            campana.reserva_veteranos.push(VeteranoGuardado {
                nombre: nombre_tropa,
                es_heroe: econ.es_heroe,
                vida_actual: stats.vida_actual,
                vida_maxima: stats.vida_maxima,
                velocidad: stats.velocidad,
                dano_base: perfil.dano_base,
                defensa_base: perfil.defensa_base,
                rango_ataque: perfil.rango_ataque,
                rasgos: perfil.rasgos.clone(),
                xp_actual: xp.xp_actual,
                nivel: xp.nivel,
                fatiga: 1, // Al terminar una batalla, ganan 1 punto de fatiga
            });
        }

        // Convertimos la estructura a texto formato RON con indentación legible
        let config_ron = ron::ser::PrettyConfig::default();
        match ron::ser::to_string_pretty(&*campana, config_ron) {
            Ok(contenido_texto) => {
                if let Err(e) = fs::write(RUTA_GUARDADO, contenido_texto) {
                    println!("❌ Error al escribir archivo en disco: {}", e);
                } else {
                    println!(
                        "💾 ¡Campaña guardada con éxito en '{}'! ({} veteranos en reserva, {} oro)",
                        RUTA_GUARDADO,
                        campana.reserva_veteranos.len(),
                        campana.oro_tesoreria
                    );
                }
            }
            Err(e) => println!("❌ Error al serializar datos RON: {}", e),
        }
    }
}

/// Lee el archivo del disco duro y actualiza el recurso DatosCampana en memoria
pub fn sistema_cargar_campana(
    mut eventos: EventReader<EventoCargarCampana>,
    mut campana: ResMut<DatosCampana>,
    mut tesoreria: ResMut<TesoreriaJugador>,
) {
    for _ in eventos.read() {
        match fs::read_to_string(RUTA_GUARDADO) {
            Ok(contenido_texto) => {
                match ron::from_str::<DatosCampana>(&contenido_texto) {
                    Ok(datos_leidos) => {
                        *campana = datos_leidos;
                        tesoreria.dinero = campana.oro_tesoreria;
                        println!(
                            "📂 ¡Campaña cargada! Mapa: {}, Oro: {}, Veteranos listos: {}",
                            campana.nivel_mapa_actual,
                            campana.oro_tesoreria,
                            campana.reserva_veteranos.len()
                        );
                        for v in &campana.reserva_veteranos {
                            println!(
                                "   -> {} | Nivel {} ({} XP) | PV: {}/{} | Fatiga: {}",
                                v.nombre, v.nivel, v.xp_actual, v.vida_actual, v.vida_maxima, v.fatiga
                            );
                        }
                    }
                    Err(e) => println!("❌ El archivo de guardado está corrupto: {}", e),
                }
            }
            Err(_) => println!("⚠️ No se encontró ninguna partida guardada en '{}'.", RUTA_GUARDADO),
        }
    }
}
```

---

## 🔌 3. Registrar el Módulo en `src/main.rs`

Registra el módulo, el recurso global y los eventos dentro de `App::new()` en tu `src/main.rs`:

```rust
mod save_system;
use save_system::*;

// Dentro de App::new():
.init_resource::<DatosCampana>()
.add_event::<EventoGuardarCampana>()
.add_event::<EventoCargarCampana>()

// Dentro de .add_systems(Update, (...)):
.add_systems(Update, (
    // ... tus sistemas existentes ...
    detectar_input_guardado,
    sistema_guardar_campana,
    sistema_cargar_campana,
))
```

---

## 📄 4. Ejemplo Real del Archivo Generado (`campana_guardada.ron`)

Al pulsar **`F5`** durante la partida, se generará en la raíz de tu proyecto un archivo de texto estructurado como este:

```ron
(
    nivel_mapa_actual: 1,
    oro_tesoreria: 50,
    reserva_veteranos: [
        (
            nombre: "Comandante (Héroe)",
            es_heroe: true,
            vida_actual: 96,
            vida_maxima: 100,
            velocidad: 15,
            dano_base: 25,
            defensa_base: 10,
            rango_ataque: 1,
            rasgos: [
                InfanteriaLigera,
            ],
            xp_actual: 60,
            nivel: 1,
            fatiga: 1,
        ),
        (
            nombre: "Veterano #2",
            es_heroe: false,
            vida_actual: 60,
            vida_maxima: 60,
            velocidad: 10,
            dano_base: 15,
            defensa_base: 5,
            rango_ataque: 2,
            rasgos: [
                Distancia,
            ],
            xp_actual: 10,
            nivel: 0,
            fatiga: 1,
        ),
    ],
)
```
