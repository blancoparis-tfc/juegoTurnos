# 🏹 SISTEMA DE COMBATE A DISTANCIA: PROYECTO HEX-TACTICS

El **Sistema de Combate a Distancia** permite que unidades como los arqueros o ballesteros ataquen objetivos situados a **2 o más casillas** sin exponerse a contraataques cuerpo a cuerpo.

Para implementarlo de forma limpia en **Bevy**, añadimos un atributo `rango_ataque` al perfil de combate de cada unidad, creamos una función geométrica para obtener **todas las casillas dentro de un radio hexagonal** y actualizamos tanto el sistema de movimiento/ataque como los indicadores visuales (anillos rojos).

---

### 1. Obtener Todas las Casillas en un Radio Hexagonal (`src/hex_grid.rs`)

Hasta ahora solo obteníamos los 6 vecinos directos (radio 1). Para que un arquero pueda disparar a 2 o 3 casillas, necesitamos obtener todas las coordenadas hexagonales dentro de un radio $N$. 

Gracias a la propiedad cúbica $q + r + s = 0$, calcular un área circular hexagonal es un bucle matemático exacto sin raíces cuadradas. Añade este método a `impl HexCoord` en `src/hex_grid.rs`:

```rust
impl HexCoord {
    // ... tus métodos anteriores (new, distancia_a, a_pixel, desde_pixel, etc.) ...

    /// Devuelve todas las coordenadas hexagonales dentro de un radio 'alcance'
    /// (Excluye la casilla central donde está parada la unidad)
    pub fn obtener_casillas_en_rango(&self, alcance: u32) -> Vec<HexCoord> {
        let mut resultados = Vec::new();
        let n = alcance as i32;

        // Recorremos el eje Q desde -N hasta +N
        for dq in -n..=n {
            // Limitamos el eje R para mantener la simetría hexagonal exacta
            let r_min = (-n).max(-dq - n);
            let r_max = n.min(-dq + n);

            for dr in r_min..=r_max {
                // Saltamos el centro (0, 0) donde está la propia unidad
                if dq == 0 && dr == 0 {
                    continue;
                }
                let ds = -dq - dr;
                resultados.push(HexCoord {
                    q: self.q + dq,
                    r: self.r + dr,
                    s: self.s + ds,
                });
            }
        }

        resultados
    }
}
```

---

### 2. Añadir Alcance y Regla de Zona Muerta (`src/combat_system.rs`)

Actualizamos `PerfilCombate` para incluir `rango_ataque` (por defecto `1` para cuerpo a cuerpo y `2` para arqueros). Además, añadimos una regla táctica clásica de los juegos por turnos: **los arqueros no pueden disparar si tienen un enemigo pegado cuerpo a cuerpo (distancia 1)**, o bien sufren penalización; en nuestro caso, un arquero con `rango_ataque: 2` podrá disparar entre 1 y 2 casillas, pero **solo recibirá contraataque si dispara a distancia 1** (lo cual ya ocurre automáticamente gracias a nuestra condición `if distancia == 1` del paso anterior).

Actualiza `PerfilCombate` en `src/combat_system.rs`:

```rust
#[derive(Component, Debug)]
pub struct PerfilCombate {
    pub dano_base: i32,
    pub defensa_base: i32,
    pub rango_ataque: u32,         // <-- Nuevo: 1 = Cuerpo a cuerpo, 2+ = Distancia
    pub rasgos: Vec<RasgoCombate>,
    pub bando_jugador: bool,
}
```

*(Nota: Recuerda añadir `rango_ataque: 1` a tu Héroe y Orco, y `rango_ataque: 2` a tu Soldado Arquero dentro de `configurar_escena` en `src/main.rs`).*

---

### 3. Permitir Ataques a Distancia al Hacer Clic (`src/game_systems.rs`)

Modificamos `sistema_movimiento_tactico` para distinguir entre **distancia de movimiento** (siempre 1 casilla paso a paso) y **distancia de ataque** (hasta `perfil.rango_ataque`).

Sustituye la validación de distancia en `sistema_movimiento_tactico` dentro de `src/game_systems.rs`:

```rust
        // 2. Buscamos si existe alguna unidad ocupando la casilla destino
        let mut ocupante_destino: Option<(Entity, bool)> = None;
        let mut pos_activa_actual = None;
        let mut bando_activo = true;
        let mut rango_activo = 1; // <-- Guardamos el alcance de la unidad activa

        for (entidad, pos, _, perfil) in query_unidades.iter() {
            if entidad == entidad_activa {
                pos_activa_actual = Some(pos.0);
                bando_activo = perfil.bando_jugador;
                rango_activo = perfil.rango_ataque;
            } else if pos.0 == evento.destino {
                ocupante_destino = Some((entidad, perfil.bando_jugador));
            }
        }

        let Some(origen) = pos_activa_actual else { continue; };
        let distancia = origen.distancia_a(&evento.destino);

        // 3. Evaluamos la acción según si hay un enemigo o si es un movimiento
        match ocupante_destino {
            // CASO A: Hay un ENEMIGO en el destino -> Comprobamos rango_ataque
            Some((entidad_enemiga, bando_ocupante)) if bando_ocupante != bando_activo => {
                if distancia > rango_activo {
                    println!("🏹 Objetivo fuera de alcance (Distancia: {}, Rango máximo: {})", distancia, rango_activo);
                    continue;
                }

                if let Ok((_, _, mut pa, _)) = query_unidades.get_mut(entidad_activa) {
                    if pa.actuales < 1 {
                        println!("🚫 ¡Sin Puntos de Acción para atacar!");
                        continue;
                    }
                    pa.actuales -= 1;
                    println!("🏹 ¡Disparo a distancia ({} casillas) lanzado!", distancia);
                    eventos_ataque.send(EventoAtaque {
                        atacante: entidad_activa,
                        defensor: entidad_enemiga,
                        es_contraataque: false,
                    });
                }
            }

            // CASO B: Casilla vacía -> El movimiento paso a paso sigue requiriendo distancia == 1
            None => {
                if distancia != 1 {
                    println!("⚠️ Solo puedes caminar a casillas adyacentes (Distancia: {})", distancia);
                    continue;
                }

                if let Ok((_, mut posicion, mut pa, _)) = query_unidades.get_mut(entidad_activa) {
                    if pa.actuales < 1 {
                        println!("🚫 ¡Sin Puntos de Acción para moverte!");
                        continue;
                    }
                    posicion.0 = evento.destino;
                    pa.actuales -= 1;
                    println!("🏃 Unidad movida a {:?}. PA restantes: {}", posicion.0, pa.actuales);
                }
            }

            // CASO C: Ocupada por un ALIADO
            Some(_) => {
                println!("🛡️ Casilla bloqueada por una unidad aliada.");
            }
        }
```

---

### 4. Dibujar Anillos Rojos en Todo el Alcance (`src/render_system.rs`)

Actualizamos `actualizar_indicadores_rango` para que:
* Dibuje **anillos cian** solo en las casillas vecinas vacías (radio `1`).
* Escaneé todas las casillas dentro de `perfil_activo.rango_ataque` y dibuje **anillos rojos** sobre cualquier enemigo dentro de ese radio.

Actualiza el bucle de dibujo en `actualizar_indicadores_rango` (`src/render_system.rs`):

```rust
        // 1. Dibujamos anillos CIAN de movimiento (siempre radio 1, casillas vacías)
        for vecino in pos_actual.0.obtener_todos_los_vecinos() {
            if !tablero.casillas.contains_key(&vecino) { continue; }

            let esta_ocupada = q_unidades.iter().any(|(_, pos, _, _)| pos.0 == vecino);
            if !esta_ocupada {
                let pixel = vecino.a_pixel(RADIO_HEXAGONO);
                commands.spawn((
                    IndicadorRango,
                    Mesh2d(anillo_hex.clone()),
                    MeshMaterial2d(color_mover.clone()),
                    Transform::from_xyz(pixel.x, pixel.y, 5.0),
                ));
            }
        }

        // 2. Dibujamos anillos ROJOS de ataque (hasta rango_ataque del perfil)
        for casilla_alcance in pos_actual.0.obtener_casillas_en_rango(perfil_activo.rango_ataque) {
            if !tablero.casillas.contains_key(&casilla_alcance) { continue; }

            // Buscamos si hay un ENEMIGO en esa casilla
            let hay_enemigo = q_unidades.iter().any(|(e, pos, _, perfil)| {
                *e != entidad_activa && pos.0 == casilla_alcance && perfil.bando_jugador != perfil_activo.bando_jugador
            });

            if hay_enemigo {
                let pixel = casilla_alcance.a_pixel(RADIO_HEXAGONO);
                commands.spawn((
                    IndicadorRango,
                    Mesh2d(anillo_hex.clone()),
                    MeshMaterial2d(color_atacar.clone()),
                    Transform::from_xyz(pixel.x, pixel.y, 5.0),
                ));
            }
        }
```

---

### Ejemplos Prácticos en Ejecución

Con tu **Soldado Arquero (`rango_ataque: 2`)** situado en `(0, 2, -2)` y el **Orco Enemigo (`rango_ataque: 1`)** situado en `(1, 0, -1)`:

1. **Cálculo de Distancia:** La separación entre `(0, 2, -2)` y `(1, 0, -1)` es **`2` casillas**.
2. **Retroalimentación Visual (UX):** Al llegar el turno del Soldado Arquero, aparecerán anillos cian en sus casillas vecinas libres para caminar, y aparecerá un **anillo rojo brillante** sobre el Orco situado a 2 casillas de distancia.
3. **Disparo Seguro sin Contraataque:** Haces clic directamente sobre el Orco. `sistema_movimiento_tactico` comprueba `distancia <= rango_activo` (`2 <= 2` es verdadero), descuenta `1 PA` y emite `EventoAtaque`.
4. **Inmunidad al Contraataque:** `procesar_combate` aplica el daño de la flecha. Al llegar al paso de contraataque, evalúa `if !murio && !evento.es_contraataque && distancia == 1`. Como `distancia` es `2` (y el Orco solo tiene alcance cuerpo a cuerpo), **el Orco no puede devolver el golpe**, protegiendo la vida de tu arquero.
