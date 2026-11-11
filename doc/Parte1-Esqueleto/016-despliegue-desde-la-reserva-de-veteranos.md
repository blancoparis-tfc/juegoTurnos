# 🚩 DESPLIEGUE DESDE LA RESERVA DE VETERANOS EN BEVY (RUST): PROYECTO HEX-TACTICS

Este documento detalla la implementación del **Sistema de Carga y Despliegue** en **Hex-Tactics**. Conecta el archivo persistente en disco (`campana_guardada.ron`) con el arranque de cada batalla (`Startup`), instanciando automáticamente a los veteranos supervivientes con sus heridas, experiencia acumulada y penalizaciones por fatiga, o generando una escuadra inicial si es una partida nueva.

---

## 🛠️ 1. Sistema de Carga y Despliegue (`src/save_system.rs`)

Añade esta función en `src/save_system.rs`. Se encarga de leer el archivo `.ron` al arrancar, aplicar penalizaciones de velocidad si una unidad está fatigada, asignar las coordenadas de despliegue y dibujar su ficha en pantalla.

```rust
use crate::hex_grid::HexCoord;
use crate::game_systems::{PosicionHex, PuntosAccion};
use crate::combat_system::Elevacion;
use crate::render_system::RADIO_HEXAGONO;

/// Sistema Startup: Carga la partida guardada (si existe) y despliega los veteranos en el tablero
pub fn desplegar_desde_reserva(
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<ColorMaterial>>,
    mut campana: ResMut<DatosCampana>,
    mut tesoreria: ResMut<TesoreriaJugador>,
) {
    // 1. Intentamos leer el archivo guardado en disco
    if let Ok(contenido_texto) = fs::read_to_string(RUTA_GUARDADO) {
        if let Ok(datos_leidos) = ron::from_str::<DatosCampana>(&contenido_texto) {
            *campana = datos_leidos;
            tesoreria.dinero = campana.oro_tesoreria;
            println!("📂 Partida cargada para despliegue (Mapa {}).", campana.nivel_mapa_actual);
        }
    }

    // 2. Si no hay veteranos guardados (primera partida), creamos la escuadra inicial
    if campana.reserva_veteranos.is_empty() {
        println!("🆕 Iniciando nueva campaña con tropas base.");
        campana.oro_tesoreria = 50;
        tesoreria.dinero = 50;
        campana.reserva_veteranos = vec![
            VeteranoGuardado {
                nombre: "Comandante (Héroe)".to_string(),
                es_heroe: true,
                vida_actual: 100,
                vida_maxima: 100,
                velocidad: 15,
                dano_base: 25,
                defensa_base: 10,
                rango_ataque: 1,
                rasgos: vec![RasgoCombate::InfanteriaLigera],
                xp_actual: 0,
                nivel: 0,
                fatiga: 0,
            },
            VeteranoGuardado {
                nombre: "Arquero Recluta".to_string(),
                es_heroe: false,
                vida_actual: 60,
                vida_maxima: 60,
                velocidad: 10,
                dano_base: 15,
                defensa_base: 5,
                rango_ataque: 2,
                rasgos: vec![RasgoCombate::Distancia],
                xp_actual: 0,
                nivel: 0,
                fatiga: 0,
            },
        ];
    }

    // 3. Definimos las casillas de despliegue disponibles en el tablero 3x3
    let zonas_despliegue = vec![
        HexCoord::new(0, 1),  // Centro
        HexCoord::new(0, 2),  // Sur
        HexCoord::new(-1, 2), // Suroeste
    ];

    let ficha_unidad = meshes.add(Circle::new(RADIO_HEXAGONO * 0.45));
    let color_heroe = materials.add(Color::srgb(0.2, 0.6, 0.9));   // Azul brillante
    let color_veterano = materials.add(Color::srgb(0.2, 0.85, 0.4)); // Verde

    // 4. Instanciamos cada veterano de la reserva en su casilla asignada
    for (indice, veterano) in campana.reserva_veteranos.iter().enumerate() {
        let Some(&coord_despliegue) = zonas_despliegue.get(indice) else {
            println!("⚠️ No hay más espacio de despliegue para {}", veterano.nombre);
            break;
        };

        // Aplicamos regla de fatiga: si fatiga > 0, pierde 2 puntos de velocidad por cansancio
        let velocidad_efectiva = if veterano.fatiga > 0 {
            println!("😓 {} despliega fatigado (-2 Velocidad).", veterano.nombre);
            (veterano.velocidad - 2).max(1)
        } else {
            veterano.velocidad
        };

        let pixel = coord_despliegue.a_pixel(RADIO_HEXAGONO);
        let material = if veterano.es_heroe { color_heroe.clone() } else { color_veterano.clone() };
        let costo_mantenimiento = if veterano.es_heroe { 0 } else { 10 };

        commands.spawn((
            Unidad,
            PosicionHex(coord_despliegue),
            Estadisticas {
                vida_actual: veterano.vida_actual,
                vida_maxima: veterano.vida_maxima,
                velocidad: velocidad_efectiva,
            },
            PuntosAccion { actuales: 2, maximos: 2 },
            EconomiaUnidad {
                costo_mantenimiento,
                es_heroe: veterano.es_heroe,
            },
            PerfilCombate {
                dano_base: veterano.dano_base,
                defensa_base: veterano.defensa_base,
                rango_ataque: veterano.rango_ataque,
                rasgos: veterano.rasgos.clone(),
                bando_jugador: true,
            },
            Experiencia {
                xp_actual: veterano.xp_actual,
                nivel: veterano.nivel,
            },
            Elevacion::Llanura,
            Mesh2d(ficha_unidad.clone()),
            MeshMaterial2d(material),
            Transform::from_xyz(pixel.x, pixel.y, 10.0),
        ));

        println!(
            "🚩 Desplegado: {} en {:?} (PV: {}/{}, XP: {})",
            veterano.nombre, coord_despliegue, veterano.vida_actual, veterano.vida_maxima, veterano.xp_actual
        );
    }
}
```

---

## 🔌 2. Actualizar el Arranque en `src/main.rs`

Reemplaza la creación manual de las tropas del jugador en `configurar_escena` por el sistema `desplegar_desde_reserva`. El encadenamiento `.chain()` garantiza que primero se cree el escenario, luego se carguen los veteranos y finalmente `calcular_linea_de_tiempo` ordene a todas las unidades por velocidad.

```rust
        // En App::new() -> Startup:
        .add_systems(Startup, (
            configurar_escena,
            dibujar_tablero_inicial,
            desplegar_desde_reserva,   // <-- Carga y despliega tus veteranos
            calcular_linea_de_tiempo,  // <-- Ordena la iniciativa con los datos ya cargados
        ).chain())
```

Simplifica `configurar_escena` para que solo genere la cámara y los enemigos del nivel:

```rust
fn configurar_escena(
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<ColorMaterial>>,
) {
    commands.spawn(Camera2d);

    // Solo instanciamos los enemigos del escenario (las tropas del jugador vienen de la Reserva)
    let ficha_unidad = meshes.add(Circle::new(RADIO_HEXAGONO * 0.45));
    let color_enemigo = materials.add(Color::srgb(0.9, 0.2, 0.2));
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
            rango_ataque: 1,
            rasgos: vec![RasgoCombate::Distancia],
            bando_jugador: false,
        },
        Elevacion::Llanura,
        Mesh2d(ficha_unidad),
        MeshMaterial2d(color_enemigo),
        Transform::from_xyz(pixel_enemigo.x, pixel_enemigo.y, 10.0),
    ));
}
```

---

## 🎮 3. Ejemplos Prácticos del Ciclo Completo de Campaña

1. **Primera Ejecución (Sin archivo `.ron`):** Al ejecutar `cargo run`, `desplegar_desde_reserva` detecta que no existe partida previa, crea al Comandante (100 PV) y al Arquero Recluta (60 PV) con 50 de oro y los sitúa en `(0, 1)` y `(0, 2)`.
2. **Combate y Guardado (`F5`):** Atacas al Orco, tu Comandante recibe 4 puntos de daño (`96 PV`), gana `+60 XP` y pulsas **`F5`**. Se guarda el estado en `campana_guardada.ron` con `fatiga: 1`.
3. **Siguiente Batalla (Reinicio del juego):** Cierras el juego y vuelves a ejecutar `cargo run`. Esta vez, `desplegar_desde_reserva` lee el archivo `.ron` automáticamente:
   * Tu Comandante aparece en el mapa con **96/100 PV** (conservando la herida del combate anterior) y sus **60 XP**.
   * Como tiene `fatiga: 1`, su velocidad pasa de `15` a `13` (`-2` por cansancio), pero sigue siendo más rápido que el Orco (`12`), por lo que `calcular_linea_de_tiempo` lo coloca igualmente primero en la cola.
