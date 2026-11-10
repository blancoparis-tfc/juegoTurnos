# 🚀 GUÍA DE CONFIGURACIÓN DEL PROYECTO EN RUST: HEX-TACTICS

Este documento detalla los pasos necesarios para inicializar el proyecto con **Cargo**, configurar el motor **Bevy** con optimizaciones de compilación y estructurar los módulos del juego.

---

## 1. 💻 Inicializar el proyecto desde la terminal
Abre tu terminal en la carpeta de tus proyectos y ejecuta los siguientes comandos:

```bash
cargo new hex_tactics
cd hex_tactics
```

Esto generará la estructura base con tu archivo de configuración `Cargo.toml` y la carpeta de código fuente `src/`.

---

## 2. ⚙️ Configurar `Cargo.toml` (Dependencias y Velocidad)
Abre el archivo `Cargo.toml` en la raíz del proyecto y reemplaza su contenido con el siguiente bloque. Incluye las optimizaciones oficiales para compilar tu código rápido sin sacrificar el rendimiento del motor:

```toml
[package]
name = "hex_tactics"
version = "0.1.0"
edition = "2024"

[dependencies]
bevy = "0.20.1"
pathfinding = "4.11"
serde = { version = "1.0", features = ["derive"] }
ron = "0.8"

# Optimización vital: compila tu código rápido pero ejecuta Bevy a máxima velocidad
[profile.dev]
opt-level = 1

[profile.dev.package."*"]
opt-level = 3
```

---

## 3. 📂 Organizar los archivos en la carpeta `src/`
Crea y organiza los módulos del proyecto dentro de la carpeta `src/`:

```text
hex_tactics/
├── Cargo.toml
└── src/
    ├── main.rs            <-- Punto de entrada y registro de sistemas
    ├── hex_grid.rs        <-- Coordenadas cúbicas (q, r, s) y vecindad
    ├── game_systems.rs    <-- Componentes ECS, turnos y economía
    └── combat_system.rs   <-- Rasgos, altura, flanqueo y eventos
```

---

## 4. 🔌 Conectar todo en `src/main.rs`
El archivo `main.rs` une tus módulos e inicializa la aplicación de Bevy registrando tus recursos, eventos y sistemas:

```rust
use bevy::prelude::*;

// 1. Declaramos los módulos del proyecto
mod hex_grid;
mod game_systems;
mod combat_system;

use game_systems::*;
use combat_system::*;

fn main() {
    App::new()
        // Carga los plugins básicos (ventana, renderizado, inputs)
        .add_plugins(DefaultPlugins.set(WindowPlugin {
            primary_window: Some(Window {
                title: "Proyecto Hex-Tactics".into(),
                resolution: (1280.0, 720.0).into(),
                ..default()
            }),
            ..default()
        }))
        // 2. Registramos los Recursos globales
        .init_resource::<TesoreriaJugador>()
        .init_resource::<ColaIniciativa>()
        // 3. Registramos los Eventos de combate
        .add_event::<EventoAtaque>()
        .add_event::<EventoUnidadEliminada>()
        // 4. Sistemas que corren una sola vez al arrancar
        .add_systems(Startup, (configurar_escena, calcular_linea_de_tiempo).chain())
        // 5. Sistemas que corren en cada fotograma
        .add_systems(Update, (
            sistema_movimiento_tactico,
            procesar_combate,
            gestionar_bajas,
        ))
        .run();
}

// Sistema inicial para colocar la cámara 2D y probar tus primeras unidades
fn configurar_escena(mut commands: Commands) {
    commands.spawn(Camera2d);

    // Creamos un Héroe de prueba en el centro (0, 0, 0)
    commands.spawn((
        Unidad,
        PosicionHex(hex_grid::HexCoord::new(0, 0)),
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
    ));
}
```

---

## 5. ▶️ Ejecutar y probar la compilación
En la terminal, lanza la primera compilación usando el enlace dinámico (*dynamic linking*) para acelerar las compilaciones posteriores mientras desarrollas:

```bash
cargo run --features bevy/dynamic_linking
```

*(Nota: La primera ejecución tardará unos minutos mientras descarga y compila las librerías de Bevy, pero las siguientes tardarán apenas unos segundos).*
