# 🌟 DESBLOQUEO DE RASGOS POR NIVEL EN BEVY (RUST): PROYECTO HEX-TACTICS

Este documento detalla la implementación del **Sistema de Especialización por Nivel** en **Hex-Tactics**. Aprovecha el vector `rasgos` dentro de `PerfilCombate` para otorgar nuevas especializaciones tácticas a las unidades al alcanzar hitos de experiencia (Nivel 2 para tropas regulares y Héroes, y Nivel 4 exclusivo para Héroes), ampliando sus ventajas en el triángulo de combate y sus posibilidades de flanqueo.

---

## 🛠️ 1. Lógica de Especialización en `src/combat_system.rs`

Añade esta función auxiliar en `src/combat_system.rs`. Evalúa el rasgo primario de la unidad al alcanzar el **Nivel 2** para asignarle una especialización complementaria, y otorga un tercer rasgo maestro a los **Héroes** al llegar al **Nivel 4**.

```rust
/// Asigna un nuevo rasgo táctico al alcanzar hitos de nivel (Nivel 2 y Nivel 4)
fn evaluar_desbloqueo_rasgos(perfil: &mut PerfilCombate, nuevo_nivel: u32, es_heroe: bool) {
    // Hito 1: Al llegar al Nivel 2, todas las tropas ganan su segunda especialización
    if nuevo_nivel == 2 {
        let nuevo_rasgo = if perfil.rasgos.contains(&RasgoCombate::InfanteriaLigera) {
            // La Infantería Ligera aprende tácticas de piquero (Anticaballería)
            Some(RasgoCombate::Anticaballeria)
        } else if perfil.rasgos.contains(&RasgoCombate::Distancia) {
            // Los Arqueros ganan movilidad montada (Caballería ligera / Arquero a caballo)
            Some(RasgoCombate::Caballeria)
        } else if perfil.rasgos.contains(&RasgoCombate::Caballeria) {
            // La Caballería aprende combate cerrado (Infantería Ligera)
            Some(RasgoCombate::InfanteriaLigera)
        } else {
            None
        };

        if let Some(rasgo) = nuevo_rasgo {
            if !perfil.rasgos.contains(&rasgo) {
                perfil.rasgos.push(rasgo.clone());
                println!("🌟 ¡ESPECIALIZACIÓN DESBLOQUEADA! Nuevo rasgo aprendido: {:?}", rasgo);
            }
        }
    }

    // Hito 2: Al llegar al Nivel 4, solo los Héroes desbloquean un tercer rasgo maestro
    if nuevo_nivel == 4 && es_heroe {
        let rasgo_maestro = RasgoCombate::Caballeria;
        if !perfil.rasgos.contains(&rasgo_maestro) {
            perfil.rasgos.push(rasgo_maestro.clone());
            println!("👑 ¡MAESTRÍA DE HÉROE! Nuevo rasgo aprendido: {:?}", rasgo_maestro);
        }
    }
}
```

---

## ⚙️ 2. Conectar el Desbloqueo en `procesar_subida_nivel` (`src/combat_system.rs`)

Llama a `evaluar_desbloqueo_rasgos` dentro del bucle `while` de `procesar_subida_nivel`, inmediatamente después de aplicar las mejoras de estadísticas base.

```rust
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

        while xp.xp_actual >= umbral {
            xp.xp_actual -= umbral;
            xp.nivel += 1;

            let (bono_vida, bono_dano, bono_defensa) = if econ.es_heroe {
                (15, 4, 2)
            } else {
                (10, 3, 1)
            };

            stats.vida_maxima += bono_vida;
            stats.vida_actual = (stats.vida_actual + bono_vida + 10).min(stats.vida_maxima);
            perfil.dano_base += bono_dano;
            perfil.defensa_base += bono_defensa;

            println!(
                "🎉 ¡SUBIDA DE NIVEL! Unidad {:?} alcanza el Nivel {}! (+{} Vida Máx, +{} Daño, +{} Defensa)",
                entidad, xp.nivel, bono_vida, bono_dano, bono_defensa
            );

            // Evaluamos si este nivel desbloquea un nuevo rasgo de combate
            evaluar_desbloqueo_rasgos(&mut perfil, xp.nivel, econ.es_heroe);

            eventos_nivel.send(EventoSubidaNivel {
                entidad,
                nuevo_nivel: xp.nivel,
            });

            umbral = xp.xp_para_siguiente_nivel();
        }
    }
}
```

---

## 🎮 3. Ejemplos Prácticos en Combate y Persistencia

* **Sinergia de Triángulo Doble:** Cuando tu **Comandante (Héroe)** alcanza el **Nivel 2**, su vector `rasgos` pasa a ser `[InfanteriaLigera, Anticaballeria]`. Como la función `tiene_ventaja_rasgo` recorre todos los rasgos del atacante, tu Héroe infligirá **+25% de daño tanto a Arqueros (*Distancia*) como a Jinetes (*Caballería*)**.
* **Sinergia de Ataque Combinado (Flanqueo):** Para activar el flanqueo que anula la defensa enemiga, las dos unidades aliadas que rodean al objetivo deben aportar **rasgos distintos**. Un veterano de Nivel 2 con dos rasgos tiene el doble de posibilidades de complementar el rasgo del aliado situado en el lado opuesto y activar el **Ataque Combinado**.
* **Guardado Automático en `.ron`:** Al pulsar **`F5`**, `sistema_guardar_campana` serializa el vector `perfil.rasgos` actualizado directamente en `campana_guardada.ron`:
  ```ron
  rasgos: [
      InfanteriaLigera,
      Anticaballeria,
  ],
  ```
