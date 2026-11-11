# 🛡️ COLISIONES, BLOQUEO DE UNIDADES Y COMBATE CONTEXTUAL: PROYECTO HEX-TACTICS

Este documento detalla cómo gestionar la ocupación de casillas y el combate contextual al hacer clic en **Bevy**. Al inspeccionar el destino antes de actuar, el sistema decide entre tres comportamientos: **desplazarse** (casilla libre), **bloquear el paso** (casilla ocupada por un aliado) o **lanzar un ataque** (casilla ocupada por un enemigo).

---

## 1. ⚙️ Actualizar el Sistema de Movimiento y Combate Contextual (`src/game_systems.rs`)

Importamos el componente `PerfilCombate` y el evento `EventoAtaque` desde el módulo de combate. Al recibir un clic, comprobamos si alguna entidad del mundo ocupa ya esa coordenada hexagonal antes de gastar Puntos de Acción (PA).

Sustituye tu función `sistema_movimiento_tactico` en `src/game_systems.rs` por esta versión ampliada:

```rust
use crate::combat_system::{PerfilCombate, EventoAtaque};

/// Sistema Update: Gestiona el movimiento, bloqueos por aliados y ataques al hacer clic
pub fn sistema_movimiento_tactico(
    mut eventos_movimiento: EventReader<EventoOrdenMovimiento>,
    mut eventos_ataque: EventWriter<EventoAtaque>, // Emisor para disparar combates
    // Consultamos todas las unidades para saber qué casillas están ocupadas
    mut query_unidades: Query<(Entity, &mut PosicionHex, &mut PuntosAccion, &PerfilCombate), With<Unidad>>,
    cola: Res<ColaIniciativa>,
    tablero: Res<TableroTactico>,
) {
    for evento in eventos_movimiento.read() {
        if cola.orden_unidades.is_empty() { return; }

        // 1. Validamos límites del mapa
        if !tablero.casillas.contains_key(&evento.destino) {
            println!("🚫 Clic fuera del tablero: {:?}", evento.destino);
            continue;
        }

        let entidad_activa = cola.orden_unidades[cola.indice_actual];

        // 2. Buscamos si existe alguna unidad ocupando la casilla destino
        let mut ocupante_destino: Option<(Entity, bool)> = None;
        let mut pos_activa_actual = None;
        let mut bando_activo = true;

        for (entidad, pos, _, perfil) in query_unidades.iter() {
            if entidad == entidad_activa {
                pos_activa_actual = Some(pos.0);
                bando_activo = perfil.bando_jugador;
            } else if pos.0 == evento.destino {
                ocupante_destino = Some((entidad, perfil.bando_jugador));
            }
        }

        let Some(origen) = pos_activa_actual else { continue; };

        // 3. Validamos distancia adyacente (1 casilla)
        let distancia = origen.distancia_a(&evento.destino);
        if distancia != 1 {
            println!("⚠️ Solo puedes interactuar con casillas adyacentes (Distancia: {})", distancia);
            continue;
        }

        // 4. Obtenemos acceso mutable a la unidad activa para gastar sus PA
        if let Ok((_, mut posicion, mut pa, _)) = query_unidades.get_mut(entidad_activa) {
            if pa.actuales < 1 {
                println!("🚫 ¡Sin Puntos de Acción! Pulsa Espacio para terminar turno.");
                continue;
            }

            // 5. Decisión táctica según quién ocupe la casilla destino
            match ocupante_destino {
                // CASO A: Casilla vacía -> Movimiento normal
                None => {
                    posicion.0 = evento.destino;
                    pa.actuales -= 1;
                    println!("🏃 Unidad movida a {:?}. PA restantes: {}", posicion.0, pa.actuales);
                }

                // CASO B: Ocupada por un ALIADO -> Bloqueo de colisión
                Some((_, bando_ocupante)) if bando_ocupante == bando_activo => {
                    println!("🛡️ Casilla bloqueada por una unidad aliada.");
                }

                // CASO C: Ocupada por un ENEMIGO -> Ataque contextual
                Some((entidad_enemiga, _)) => {
                    pa.actuales -= 1;
                    println!("⚔️ ¡Objetivo enemigo detectado! Lanzando ataque...");
                    eventos_ataque.send(EventoAtaque {
                        atacante: entidad_activa,
                        defensor: entidad_enemiga,
                    });
                }
            }
        }
    }
}
```

---

## 2. 🎨 Actualizar los Indicadores Visuales UX (`src/render_system.rs`)

Para que el jugador distinga visualmente entre una casilla libre para caminar y una casilla ocupada por un enemigo atacable, mejoramos `actualizar_indicadores_rango` para pintar **anillos cian** en casillas vacías y **anillos rojos** sobre los enemigos adyacentes (ocultando el indicador en casillas bloqueadas por aliados).

Actualiza `actualizar_indicadores_rango` en `src/render_system.rs`:

```rust
use crate::combat_system::PerfilCombate;

/// Sistema Update: Dibuja anillos cian (mover), rojos (atacar) y oculta aliados bloqueados
pub fn actualizar_indicadores_rango(
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<ColorMaterial>>,
    q_indicadores_viejos: Query<Entity, With<IndicadorRango>>,
    // Ahora consultamos también el PerfilCombate para saber el bando de cada tropa
    q_unidades: Query<(Entity, &PosicionHex, &PuntosAccion, &PerfilCombate), With<Unidad>>,
    q_cambios: Query<Entity, (With<Unidad>, Or<(Changed<PosicionHex>, Changed<PuntosAccion>)>)>,
    cola: Res<ColaIniciativa>,
    tablero: Res<TableroTactico>,
) {
    if q_cambios.is_empty() && !cola.is_changed() {
        return;
    }

    for entidad_vieja in q_indicadores_viejos.iter() {
        commands.entity(entidad_vieja).despawn();
    }

    if cola.orden_unidades.is_empty() { return; }

    let entidad_activa = cola.orden_unidades[cola.indice_actual];

    if let Ok((_, pos_actual, pa, perfil_activo)) = q_unidades.get(entidad_activa) {
        if pa.actuales == 0 { return; }

        let anillo_hex = meshes.add(Annulus::new(RADIO_HEXAGONO * 0.65, RADIO_HEXAGONO * 0.82));
        let color_mover = materials.add(Color::srgba(0.1, 0.85, 0.95, 0.55));  // Cian
        let color_atacar = materials.add(Color::srgba(0.95, 0.2, 0.2, 0.70));  // Rojo

        for vecino in pos_actual.0.obtener_todos_los_vecinos() {
            if !tablero.casillas.contains_key(&vecino) { continue; }

            // Comprobamos si hay alguien en esta casilla vecina
            let ocupante = q_unidades
                .iter()
                .find(|(e, pos, _, _)| *e != entidad_activa && pos.0 == vecino);

            let material_seleccionado = match ocupante {
                None => Some(color_mover.clone()), // Libre: se puede caminar
                Some((_, _, _, perfil_vecino)) if perfil_vecino.bando_jugador != perfil_activo.bando_jugador => {
                    Some(color_atacar.clone()) // Enemigo: se puede atacar
                }
                _ => None, // Aliado: bloqueado, no dibujamos anillo
            };

            if let Some(material) = material_seleccionado {
                let pixel = vecino.a_pixel(RADIO_HEXAGONO);
                commands.spawn((
                    IndicadorRango,
                    Mesh2d(anillo_hex.clone()),
                    MeshMaterial2d(material),
                    Transform::from_xyz(pixel.x, pixel.y, 5.0),
                ));
            }
        }
    }
}
```

---

## 3. 👹 Añadir un Enemigo al Tablero de Prueba (`src/main.rs`)

Para probar el sistema de colisiones y combate completo, añade un **Orco Enemigo** (ficha roja) en la casilla `(1, 0, -1)` dentro de `configurar_escena` en tu archivo `src/main.rs`:

```rust
    // 3. Enemigo de Prueba (Velocidad 12 -> Actúa entre el Héroe y el Soldado) en (1, 0)
    let color_enemigo = materials.add(Color::srgb(0.9, 0.2, 0.2)); // Rojo
    let coord_enemigo = hex_grid::HexCoord::new(1, 0);
    let pixel_enemigo = coord_enemigo.a_pixel(RADIO_HEXAGONO);

    commands.spawn((
        Unidad,
        PosicionHex(coord_enemigo),
        Estadisticas { vida_actual: 50, vida_maxima: 50, velocidad: 12 },
        PuntosAccion { actuales: 2, maximos: 2 },
        EconomiaUnidad { costo_mantenimiento: 0, es_heroe: false },
        PerfilCombate {
            dano_base: 18,
            defensa_base: 5,
            rasgos: vec![RasgoCombate::Distancia], // Débil contra tu Héroe (Infantería Ligera)
            bando_jugador: false, // Pertenece al bando enemigo
        },
        Elevacion::Llanura,
        Mesh2d(ficha_unidad),
        MeshMaterial2d(color_enemigo),
        Transform::from_xyz(pixel_enemigo.x, pixel_enemigo.y, 10.0),
    ));
```

---

## 4. 🎮 Ejemplos Prácticos en Ejecución

Al arrancar el juego con estas tres unidades en el tablero de 3x3:

1. **Rodeado por aliados y enemigos:** Tu Héroe en `(0, 1, -1)` verá **4 anillos cian** (casillas libres), **1 anillo rojo** sobre el enemigo en `(1, 0, -1)` y **ningún anillo** sobre el Soldado aliado en `(0, 2, -2)`.
2. **Bloqueo físico:** Si intentas hacer clic sobre el Soldado aliado, la unidad no se moverá ni gastará Puntos de Acción, mostrando el aviso de bloqueo en consola.
3. **Ataque directo y eliminación:** Si haces clic sobre el enemigo rojo, tu Héroe gastará `1 PA`, aplicará el **+25% de ventaja de rasgo** (Infantería Ligera contra Distancia) y el **+15% por altura** (ataca desde Colina hacia Llanura). Con dos clics consecutivos derrotará al enemigo, ganará `+50 XP` y el sistema `gestionar_bajas` borrará la ficha roja del mapa.
