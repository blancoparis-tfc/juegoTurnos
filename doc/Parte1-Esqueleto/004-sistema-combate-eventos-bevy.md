# ⚔️ SISTEMA DE COMBATE Y EVENTOS EN BEVY (RUST): PROYECTO HEX-TACTICS

Este documento detalla la implementación del **Sistema de Combate** mediante la arquitectura orientada a **Eventos (`Events`)** de Bevy. Al desacoplar la acción de atacar de su resolución matemática, distintos sistemas pueden reaccionar de forma independiente para calcular el daño por rasgos, aplicar bonos de altura, detectar flanqueos hexagonales y otorgar experiencia (XP).

---

## 🛠️ 1. Código Fuente (`combat_system.rs`)

```rust
use bevy::prelude::*;
use std::collections::HashSet;
use crate::hex_grid::HexCoord;
use crate::game_systems::{PosicionHex, Estadisticas, Unidad};

// ==========================================
// 1. COMPONENTES DE COMBATE Y RASGOS
// ==========================================

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum RasgoCombate {
    InfanteriaLigera,
    Distancia,
    Caballeria,
}

#[derive(Component, Debug)]
pub struct PerfilCombate {
    pub dano_base: i32,
    pub defensa_base: i32,
    pub rasgos: Vec<RasgoCombate>, // Permite tener varios rasgos (Unidades Híbridas)
    pub bando_jugador: bool,
}

#[derive(Component, Debug, Default)]
pub struct Experiencia {
    pub xp_actual: u32,
    pub nivel: u32,
}

#[derive(Component, Debug, Clone, Copy, PartialEq, Eq)]
pub enum Elevacion {
    Llanura = 0,
    Colina = 1,
    Montana = 2,
}

// ==========================================
// 2. EVENTOS (Mensajes entre Sistemas)
// ==========================================

#[derive(Event, Debug)]
pub struct EventoAtaque {
    pub atacante: Entity,
    pub defensor: Entity,
}

#[derive(Event, Debug)]
pub struct EventoUnidadEliminada {
    pub victima: Entity,
    pub asesino: Entity,
}

// ==========================================
// 3. SISTEMAS DE RESOLUCIÓN DE COMBATE
// ==========================================

/// Sistema A: Escucha los eventos de ataque y aplica las fórmulas del GDD
pub fn procesar_combate(
    mut eventos_ataque: EventReader<EventoAtaque>,
    mut eventos_muerte: EventWriter<EventoUnidadEliminada>,
    // Consultamos datos de lectura para calcular bonos
    query_info: Query<(&PosicionHex, &PerfilCombate, &Elevacion), With<Unidad>>,
    // Consultamos datos mutables para aplicar daño y XP
    mut query_estado: Query<(&mut Estadisticas, &mut Experiencia), With<Unidad>>,
) {
    for evento in eventos_ataque.read() {
        // Obtenemos la información espacial y de rasgos de ambos combatientes
        let Ok((pos_atk, perfil_atk, elev_atk)) = query_info.get(evento.atacante) else { continue; };
        let Ok((pos_def, perfil_def, elev_def)) = query_info.get(evento.defensor) else { continue; };

        let mut multiplicador_dano = 1.0;

        // 1. Bono del Triángulo de Combate (+25% daño)
        if tiene_ventaja_rasgo(&perfil_atk.rasgos, &perfil_def.rasgos) {
            multiplicador_dano += 0.25;
            println!("⚔️ ¡Ventaja de rasgo aplicada! (+25%)");
        }

        // 2. Bono de Altura (Sección 2 del GDD: +15% si ataca desde arriba)
        if (*elev_atk as i32) > (*elev_def as i32) {
            multiplicador_dano += 0.15;
            println!("⛰️ ¡Bono de altura aplicado! (+15%)");
        }

        // 3. Flanqueo Colectivo (Sección 6 del GDD: Aliados con distinto rasgo rodeando al objetivo)
        let flanqueado = verificar_flanqueo_colectivo(
            pos_def.0, 
            perfil_atk.bando_jugador, 
            &perfil_atk.rasgos, 
            &query_info
        );

        let defensa_efectiva = if flanqueado {
            println!("🛡️💥 ¡Ataque Combinado! Defensa del enemigo anulada.");
            0 // Anula los bonos defensivos del rival
        } else {
            perfil_def.defensa_base
        };

        // Cálculo final determinista
        let dano_bruto = (perfil_atk.dano_base as f32 * multiplicador_dano).round() as i32;
        let dano_final = (dano_bruto - defensa_efectiva).max(1); // Mínimo 1 de daño

        // Aplicamos el daño al defensor
        let mut murio = false;
        if let Ok((mut stats_def, _)) = query_estado.get_mut(evento.defensor) {
            stats_def.vida_actual -= dano_final;
            println!("💥 Daño infligido: {}. Vida restante: {}", dano_final, stats_def.vida_actual);
            if stats_def.vida_actual <= 0 {
                murio = true;
            }
        }

        // Otorgamos experiencia (XP) al atacante
        if let Ok((_, mut xp_atk)) = query_estado.get_mut(evento.atacante) {
            if murio {
                xp_atk.xp_actual += 50; // Gran bono por golpe de gracia (Last-hit)
                println!("⭐ ¡Enemigo eliminado! +50 XP para el atacante.");
                eventos_muerte.send(EventoUnidadEliminada {
                    victima: evento.defensor,
                    asesino: evento.atacante,
                });
            } else {
                xp_atk.xp_actual += 10; // XP base por combatir
            }
        }
    }
}

/// Sistema B: Limpia del tablero las unidades derrotadas
pub fn gestionar_bajas(
    mut commands: Commands,
    mut eventos_muerte: EventReader<EventoUnidadEliminada>,
) {
    for evento in eventos_muerte.read() {
        // Despachamos la entidad fuera del mundo de Bevy
        // (Aquí evaluaríamos antes si es Héroe para enviarlo a la lista de "Heridos" en campaña)
        commands.entity(evento.victima).despawn();
        println!("💀 Entidad {:?} eliminada del tablero.", evento.victima);
    }
}

// ==========================================
// 4. FUNCIONES AUXILIARES DE REGLAS
// ==========================================

fn tiene_ventaja_rasgo(atacante: &[RasgoCombate], defensor: &[RasgoCombate]) -> bool {
    for &atk in atacante {
        for &def in defensor {
            match (atk, def) {
                (RasgoCombate::InfanteriaLigera, RasgoCombate::Distancia) => return true,
                (RasgoCombate::Distancia, RasgoCombate::Caballeria) => return true,
                (RasgoCombate::Caballeria, RasgoCombate::InfanteriaLigera) => return true,
                _ => {}
            }
        }
    }
    false
}

fn verificar_flanqueo_colectivo(
    pos_objetivo: HexCoord,
    bando_atacante: bool,
    rasgos_atacante: &[RasgoCombate],
    query_info: &Query<(&PosicionHex, &PerfilCombate, &Elevacion), With<Unidad>>,
) -> bool {
    let vecinos = pos_objetivo.obtener_todos_los_vecinos();
    let mut rasgos_aliados_adyacentes = HashSet::new();

    // Revisamos qué aliados están tocando al enemigo
    for (pos, perfil, _) in query_info.iter() {
        if perfil.bando_jugador == bando_atacante && vecinos.contains(&pos.0) {
            for &r in &perfil.rasgos {
                rasgos_aliados_adyacentes.insert(r);
            }
        }
    }

    // Si hay algún aliado adyacente con un rasgo distinto al del atacante principal, se activa la sinergia
    rasgos_aliados_adyacentes.iter().any(|r| !rasgos_atacante.contains(r))
}
```

---

## 🧠 2. Explicación de la Arquitectura de Combate

* **Desacoplamiento mediante Eventos (`EventReader` / `EventWriter`):** Al disparar un `EventoAtaque`, el sistema de interfaz o movimiento no necesita saber cómo se calcula el daño. El sistema `procesar_combate` lee el evento, ejecuta las reglas y, si la vida llega a 0, emite un nuevo `EventoUnidadEliminada` para que otro sistema se encargue de borrar la ficha o activar el bono de chatarra/moral.
* **Unidades Híbridas (`Vec<RasgoCombate>`):** Al almacenar los rasgos en un vector en lugar de un único valor, las unidades avanzadas (como un *Jinete Arquero*) pueden evaluar automáticamente si alguno de sus dos rasgos tiene ventaja sobre el enemigo sin duplicar código.
* **Flanqueo Colectivo Hexagonal:** La función `verificar_flanqueo_colectivo` aprovecha el método `.obtener_todos_los_vecinos()`. Revisa las 6 casillas alrededor del enemigo y recopila los rasgos de las tropas aliadas adyacentes en un `HashSet`. Si detecta al menos un rasgo distinto al del atacante principal, anula por completo la defensa del objetivo.
* **Prevención de Conflictos de Memoria en Bevy:** En `procesar_combate` se realizan dos `Queries` separadas (`query_info` de solo lectura y `query_estado` mutable). Esto permite inspeccionar posiciones de todo el tablero mientras se modifica la vida y experiencia de los combatientes sin violar las reglas de préstamo de memoria de Rust.
