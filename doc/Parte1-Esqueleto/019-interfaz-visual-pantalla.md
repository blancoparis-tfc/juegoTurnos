# 🖥️ INTERFAZ VISUAL EN PANTALLA (HUD CON BEVY UI) EN RUST: PROYECTO HEX-TACTICS

Este documento detalla la implementación del **HUD Táctico (Heads-Up Display)** en **Hex-Tactics** utilizando el motor nativo **`bevy::ui`**. Crea dos paneles superpuestos a la cámara 2D que muestran en tiempo real el estado global de la batalla (oro, ronda, momento del día y clima) y la ficha técnica de la unidad que tiene el turno activo (vida, puntos de acción, experiencia, estadísticas y rasgos).

---

## 🛠️ 1. Crear el Módulo de Interfaz (`src/ui_system.rs`)

Crea el archivo `src/ui_system.rs`. Definimos dos componentes marcadores (`TextoInfoEntorno` y `TextoInfoUnidad`), el sistema `Startup` que construye la jerarquía de nodos estilo Flexbox y el sistema `Update` que actualiza los textos de forma reactiva.

```rust
use bevy::prelude::*;
use crate::game_systems::{
    Estadisticas, PuntosAccion, TesoreriaJugador, ColaIniciativa, Unidad,
};
use crate::combat_system::{PerfilCombate, Experiencia};
use crate::environment_system::EntornoBatalla;

// ==========================================
// 1. COMPONENTES MARCADORES DE UI
// ==========================================

#[derive(Component)]
pub struct TextoInfoEntorno;

#[derive(Component)]
pub struct TextoInfoUnidad;

// ==========================================
// 2. SISTEMA STARTUP: CONSTRUIR EL HUD
// ==========================================

/// Crea los paneles visuales y textos iniciales en pantalla
pub fn configurar_hud_tactico(mut commands: Commands) {
    // Contenedor raíz que ocupa toda la ventana
    commands
        .spawn(Node {
            width: Val::Percent(100.0),
            height: Val::Percent(100.0),
            flex_direction: FlexDirection::Column,
            justify_content: JustifyContent::SpaceBetween, // Empuja un panel arriba y otro abajo
            padding: UiRect::all(Val::Px(16.0)),
            ..default()
        })
        .with_children(|parent| {
            // ------------------------------------------
            // PANEL SUPERIOR: Tesorería, Ronda y Clima
            // ------------------------------------------
            parent
                .spawn((
                    Node {
                        padding: UiRect::axes(Val::Px(16.0), Val::Px(10.0)),
                        border_radius: BorderRadius::all(Val::Px(8.0)),
                        align_self: AlignSelf::Center,
                        ..default()
                    },
                    BackgroundColor(Color::srgba(0.08, 0.10, 0.14, 0.85)),
                ))
                .with_children(|panel_sup| {
                    panel_sup.spawn((
                        TextoInfoEntorno,
                        Text::new("Cargando entorno..."),
                        TextFont {
                            font_size: 18.0,
                            ..default()
                        },
                        TextColor(Color::srgb(0.95, 0.85, 0.45)), // Dorado suave
                    ));
                });

            // ------------------------------------------
            // PANEL INFERIOR IZQUIERDO: Unidad Activa
            // ------------------------------------------
            parent
                .spawn((
                    Node {
                        padding: UiRect::all(Val::Px(14.0)),
                        border_radius: BorderRadius::all(Val::Px(8.0)),
                        align_self: AlignSelf::FlexStart,
                        max_width: Val::Px(420.0),
                        ..default()
                    },
                    BackgroundColor(Color::srgba(0.08, 0.10, 0.14, 0.88)),
                ))
                .with_children(|panel_inf| {
                    panel_inf.spawn((
                        TextoInfoUnidad,
                        Text::new("Esperando turno..."),
                        TextFont {
                            font_size: 16.0,
                            ..default()
                        },
                        TextColor(Color::WHITE),
                    ));
                });
        });
}

// ==========================================
// 3. SISTEMA UPDATE: REFRESCAR DATOS EN VIVO
// ==========================================

/// Actualiza los textos del HUD cuando cambian los recursos o el estado de las unidades
pub fn actualizar_hud_tactico(
    mut q_texto_entorno: Query<&mut Text, (With<TextoInfoEntorno>, Without<TextoInfoUnidad>)>,
    mut q_texto_unidad: Query<&mut Text, (With<TextoInfoUnidad>, Without<TextoInfoEntorno>)>,
    tesoreria: Res<TesoreriaJugador>,
    entorno: Res<EntornoBatalla>,
    cola: Res<ColaIniciativa>,
    q_unidades: Query<(
        &Estadisticas,
        &PuntosAccion,
        &PerfilCombate,
        &Experiencia,
    ), With<Unidad>>,
) {
    // 1. Actualizamos la barra superior (Oro, Ronda, Día/Noche, Clima)
    if tesoreria.is_changed() || entorno.is_changed() {
        if let Ok(mut texto_entorno) = q_texto_entorno.single_mut() {
            texto_entorno.0 = format!(
                "Oro: {}  |  Ronda: {}  |  Momento: {:?}  |  Clima: {:?}  |  [Espacio] Fin Turno  [F5] Guardar",
                tesoreria.dinero,
                entorno.ronda_actual,
                entorno.momento,
                entorno.clima
            );
        }
    }

    // 2. Actualizamos el panel inferior con la unidad que tiene el turno
    if cola.orden_unidades.is_empty() {
        return;
    }

    let entidad_activa = cola.orden_unidades[cola.indice_actual];

    if let Ok((stats, pa, perfil, xp)) = q_unidades.get(entidad_activa) {
        if let Ok(mut texto_unidad) = q_texto_unidad.single_mut() {
            let bando = if perfil.bando_jugador { "ALIADO" } else { "ENEMIGO" };
            let umbral_xp = xp.xp_para_siguiente_nivel();

            texto_unidad.0 = format!(
                "TURNO ACTIVO ({})\n\
                Nivel: {} ({}/{} XP)\n\
                Vida (PV): {}/{}  |  PA: {}/{}\n\
                Ataque: {}  |  Defensa: {}  |  Alcance: {}\n\
                Rasgos: {:?}",
                bando,
                xp.nivel,
                xp.xp_actual,
                umbral_xp,
                stats.vida_actual,
                stats.vida_maxima,
                pa.actuales,
                pa.maximos,
                perfil.dano_base,
                perfil.defensa_base,
                perfil.rango_ataque,
                perfil.rasgos
            );
        }
    }
}
```

---

## 🔌 2. Registrar el Módulo Visual en `src/main.rs`

Añade el módulo `ui_system` en `src/main.rs` y registra sus dos sistemas dentro de `Startup` y `Update`:

```rust
mod ui_system;
use ui_system::*;

// Dentro de App::new():
        .add_systems(Startup, (
            configurar_escena,
            dibujar_tablero_inicial,
            desplegar_desde_reserva,
            calcular_linea_de_tiempo,
            configurar_hud_tactico, // <-- Construye los paneles HUD al arrancar
        ).chain())
        .add_systems(Update, (
            detectar_clic_tablero,
            detectar_input_fin_turno,
            gestionar_avance_turno,
            sistema_movimiento_tactico,
            sistema_economia_ronda,
            procesar_combate,
            procesar_subida_nivel,
            limpiar_cola_iniciativa,
            gestionar_bajas,
            sincronizar_visuales_unidades,
            actualizar_indicadores_rango,
            actualizar_entorno_ronda,
            detectar_input_guardado,
            sistema_guardar_campana,
            sistema_cargar_campana,
            actualizar_hud_tactico, // <-- Refresca los textos en pantalla en tiempo real
        ))
```

---

## 🎮 3. Comportamiento en Pantalla

* **Barra Superior (Centro):** Muestra en color dorado `Oro: 50 | Ronda: 1 | Momento: Dia | Clima: Despejado | [Espacio] Fin Turno [F5] Guardar`. Al avanzar 3 rondas globales, el texto cambia automáticamente a `Momento: Noche`.
* **Panel Inferior Izquierdo:** Muestra la tarjeta de la unidad activa. Al hacer clic para desplazarte o atacar, el indicador `PA: 2/2` baja instantáneamente a `PA: 1/2`. Al pulsar `Espacio`, la tarjeta se actualiza con las estadísticas de la siguiente unidad en la `ColaIniciativa`.
