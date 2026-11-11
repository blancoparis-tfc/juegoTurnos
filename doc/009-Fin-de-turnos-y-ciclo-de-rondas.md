# ⏳ SISTEMA DE FIN DE TURNO Y CICLO DE RONDAS: PROYECTO HEX-TACTICS

El **Sistema de Fin de Turno** es el motor que hace avanzar la **Línea de Tiempo** en **Hex-Tactics**. 

Para mantener una arquitectura limpia en **Bevy**, conectamos la entrada del jugador (pulsar la tecla `Espacio`) con un evento (`EventoFinTurno`). Cuando el sistema procesa este evento, avanza el índice de la `ColaIniciativa` hacia la siguiente unidad. Si el índice llega al final de la lista, significa que **todas las unidades han actuado**, por lo que se dispara automáticamente el cierre de la ronda global, la recarga de Puntos de Acción (PA) y el cálculo económico.

---

## 1. 📨 Eventos de Control de Turno (`src/game_systems.rs`)

Añadimos dos eventos en `src/game_systems.rs` para separar el avance individual del cierre de ronda global:

```rust
use bevy::prelude::*;

// Evento A: Se dispara cuando una unidad termina su activación individual
#[derive(Event, Debug)]
pub struct EventoFinTurno;

// Evento B: Se dispara cuando toda la cola ha actuado una vez (Fin de Ronda Global)
#[derive(Event, Debug)]
pub struct EventoFinRondaGlobal;
```

---

## 2. ⚙️ Sistemas de Avance de Turno y Economía (`src/game_systems.rs`)

Implementamos el detector de teclado y el gestor de la cola. También ajustamos `sistema_economia_ronda` para que escuche el evento `EventoFinRondaGlobal` en lugar de ejecutarse sin control en cada fotograma.

```rust
/// Sistema Update: Detecta si el jugador pulsa Espacio para terminar el turno actual
pub fn detectar_input_fin_turno(
    teclado: Res<ButtonInput<KeyCode>>,
    mut eventos_fin_turno: EventWriter<EventoFinTurno>,
) {
    if teclado.just_pressed(KeyCode::Space) {
        println!("⏭️ Tecla Espacio pulsada: Solicitando fin de turno...");
        eventos_fin_turno.send(EventoFinTurno);
    }
}

/// Sistema Update: Avanza la Cola de Iniciativa hacia la siguiente unidad
pub fn gestionar_avance_turno(
    mut eventos_fin_turno: EventReader<EventoFinTurno>,
    mut eventos_fin_ronda: EventWriter<EventoFinRondaGlobal>,
    mut cola: ResMut<ColaIniciativa>,
) {
    for _ in eventos_fin_turno.read() {
        if cola.orden_unidades.is_empty() { return; }

        // Avanzamos el puntero al siguiente índice
        cola.indice_actual += 1;

        // Comprobamos si ya actuaron todas las unidades de la lista
        if cola.indice_actual >= cola.orden_unidades.len() {
            // Reiniciamos el puntero al principio de la cola
            cola.indice_actual = 0;
            println!("🔄 ¡Ronda Global completada! Reiniciando línea de tiempo...");
            
            // Disparamos el evento económico y de recarga de PA
            eventos_fin_ronda.send(EventoFinRondaGlobal);
        } else {
            let siguiente_entidad = cola.orden_unidades[cola.indice_actual];
            println!("👉 Turno de la siguiente unidad: {:?}", siguiente_entidad);
        }
    }
}

/// Sistema Update: Se ejecuta SOLO cuando se recibe EventoFinRondaGlobal
pub fn sistema_economia_ronda(
    mut eventos_fin_ronda: EventReader<EventoFinRondaGlobal>,
    query: Query<&EconomiaUnidad, With<Unidad>>,
    mut tesoreria: ResMut<TesoreriaJugador>,
    mut query_pa: Query<&mut PuntosAccion, With<Unidad>>,
) {
    for _ in eventos_fin_ronda.read() {
        let mut costo_total_mantenimiento = 0;

        for econ in query.iter() {
            if !econ.es_heroe {
                costo_total_mantenimiento += econ.costo_mantenimiento as i32;
            }
        }

        // Ejemplo: 3 casas controladas generan 30 de oro
        let ingresos_casas = 30; 
        let balance_neto = ingresos_casas - costo_total_mantenimiento;
        
        tesoreria.dinero += balance_neto;
        println!(
            "💰 Balance de Ronda -> Ingresos: +{}, Mantenimiento: -{}. Tesorería: {}",
            ingresos_casas, costo_total_mantenimiento, tesoreria.dinero
        );

        if tesoreria.dinero < 0 {
            tesoreria.dinero = 0;
            println!("⚠️ ¡BANCARROTA! Las unidades sufren penalización por Hambruna.");
        }

        // Rellenamos los Puntos de Acción de todas las unidades para la nueva ronda
        for mut pa in query_pa.iter_mut() {
            pa.actuales = pa.maximos;
        }
        println!("⚡ Puntos de Acción (PA) restaurados para todas las tropas.");
    }
}
```

---

## 3. 🎮 Probar el Salto de Turnos con Dos Unidades (`src/main.rs`)

Para ver cómo los anillos cian de movimiento saltan automáticamente de una unidad a otra al pulsar `Espacio`, añadimos una segunda tropa (un **Soldado Regular**) dentro de `configurar_escena` y registramos los nuevos sistemas en `main.rs`.

```rust
fn main() {
    App::new()
        .add_plugins(DefaultPlugins.set(WindowPlugin {
            primary_window: Some(Window {
                title: "Proyecto Hex-Tactics".into(),
                resolution: (1280.0, 720.0).into(),
                ..default()
            }),
            ..default()
        }))
        .init_resource::<TesoreriaJugador>()
        .init_resource::<ColaIniciativa>()
        .init_resource::<TableroTactico>()
        // Registramos todos los eventos
        .add_event::<EventoAtaque>()
        .add_event::<EventoUnidadEliminada>()
        .add_event::<EventoOrdenMovimiento>()
        .add_event::<EventoFinTurno>()        // <-- Nuevo
        .add_event::<EventoFinRondaGlobal>()  // <-- Nuevo
        .add_systems(Startup, (
            configurar_escena, 
            dibujar_tablero_inicial, 
            calcular_linea_de_tiempo
        ).chain())
        .add_systems(Update, (
            detectar_clic_tablero,
            detectar_input_fin_turno, // <-- Nuevo
            gestionar_avance_turno,   // <-- Nuevo
            sistema_movimiento_tactico,
            sistema_economia_ronda,   // <-- Ahora reactivo por evento
            procesar_combate,
            gestionar_bajas,
            sincronizar_visuales_unidades,
            actualizar_indicadores_rango,
        ))
        .run();
}

fn configurar_escena(
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<ColorMaterial>>,
) {
    commands.spawn(Camera2d);

    let ficha_unidad = meshes.add(Circle::new(RADIO_HEXAGONO * 0.45));
    let color_heroe = materials.add(Color::srgb(0.2, 0.6, 0.9));   // Azul
    let color_soldado = materials.add(Color::srgb(0.2, 0.85, 0.4)); // Verde

    // 1. Héroe (Velocidad 15 -> Actúa primero) en el centro (0, 1)
    let coord_heroe = hex_grid::HexCoord::new(0, 1);
    let pixel_heroe = coord_heroe.a_pixel(RADIO_HEXAGONO);
    commands.spawn((
        Unidad,
        PosicionHex(coord_heroe),
        Estadisticas { vida_actual: 100, vida_maxima: 100, velocidad: 15 },
        PuntosAccion { actuales: 2, maximos: 2 },
        EconomiaUnidad { costo_mantenimiento: 0, es_heroe: true },
        PerfilCombate {
            dano_base: 25,
            defensa_base: 10,
            rasgos: vec![RasgoCombate::InfanteriaLigera],
            bando_jugador: true,
        },
        Elevacion::Colina,
        Mesh2d(ficha_unidad.clone()),
        MeshMaterial2d(color_heroe),
        Transform::from_xyz(pixel_heroe.x, pixel_heroe.y, 10.0),
    ));

    // 2. Soldado Regular (Velocidad 10 -> Actúa segundo) en (0, 2)
    let coord_soldado = hex_grid::HexCoord::new(0, 2);
    let pixel_soldado = coord_soldado.a_pixel(RADIO_HEXAGONO);
    commands.spawn((
        Unidad,
        PosicionHex(coord_soldado),
        Estadisticas { vida_actual: 60, vida_maxima: 60, velocidad: 10 },
        PuntosAccion { actuales: 2, maximos: 2 },
        EconomiaUnidad { costo_mantenimiento: 10, es_heroe: false },
        PerfilCombate {
            dano_base: 15,
            defensa_base: 5,
            rasgos: vec![RasgoCombate::Distancia],
            bando_jugador: true,
        },
        Elevacion::Llanura,
        Mesh2d(ficha_unidad),
        MeshMaterial2d(color_soldado),
        Transform::from_xyz(pixel_soldado.x, pixel_soldado.y, 10.0),
    ));
}
```

---

## 4. 🔄 Ejemplos Prácticos del Ciclo de Turnos

Al ejecutar el proyecto con estas dos tropas en pantalla, ocurrirá lo siguiente:

1. **Inicio de Partida (`indice_actual = 0`):** `calcular_linea_de_tiempo` ordena por velocidad. El **Héroe (Vel 15)** va primero y el **Soldado Verde (Vel 10)** va segundo. Los anillos cian aparecen alrededor del Héroe Azul.
2. **Primer Fin de Turno (`Espacio`):** El jugador pulsa `Espacio`. `gestionar_avance_turno` sube `indice_actual` a `1`. Como `cola` ha sido modificada (`cola.is_changed()`), el sistema `actualizar_indicadores_rango` borra los anillos del Héroe y los dibuja instantáneamente alrededor del **Soldado Verde**. Ahora los clics del ratón mueven al Soldado.
3. **Segundo Fin de Turno (`Espacio`):** El jugador vuelve a pulsar `Espacio`. `indice_actual` sube a `2`, igualando la longitud de la lista (`len() == 2`). El sistema reinicia `indice_actual = 0` y dispara `EventoFinRondaGlobal`.
4. **Cierre Económico Automático:** `sistema_economia_ronda` suma `+30` de ingresos de casas, resta `-10` por el mantenimiento del Soldado Regular (`0` por el Héroe), deja `+20` en la tesorería y devuelve los `2 PA` a ambas unidades para empezar la Ronda 2.
