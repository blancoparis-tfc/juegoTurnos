# ⚙️ ARQUITECTURA ECS EN BEVY (RUST): PROYECTO HEX-TACTICS

Este documento recoge la implementación base en Rust para trasladar las mecánicas de **Hex-Tactics** al paradigma **Entity Component System (ECS)** de Bevy. 

Aquí las unidades no son clases tradicionales; son identificadores únicos (`Entity`) que agrupan contenedores de datos puros (**Componentes**), mientras que las reglas del juego se ejecutan mediante funciones independientes (**Sistemas**).

---

## 🛠️ Código Fuente (`game_systems.rs`)

```rust
use bevy::prelude::*;
use std::collections::HashMap;
use crate::hex_grid::HexCoord; // Importamos tu struct matemático anterior

// ==========================================
// 1. COMPONENTES (Estructuras de Datos)
// ==========================================

#[derive(Component, Debug)]
pub struct Unidad;

#[derive(Component, Debug)]
pub struct PosicionHex(pub HexCoord);

#[derive(Component, Debug)]
pub struct Estadisticas {
    pub vida_actual: i32,
    pub vida_maxima: i32,
    pub velocidad: i32, // Determina el orden en la línea de tiempo
}

#[derive(Component, Debug)]
pub struct PuntosAccion {
    pub actuales: u32,
    pub maximos: u32,
}

#[derive(Component, Debug)]
pub struct EconomiaUnidad {
    pub costo_mantenimiento: u32,
    pub es_heroe: bool, // Si es héroe, el mantenimiento es 0
}

#[derive(Component, Debug)]
pub struct EstadoGuardia {
    pub activo: bool,
}

// ==========================================
// 2. RECURSOS (Datos Globales del Mundo)
// ==========================================

#[derive(Resource, Debug, Default)]
pub struct TesoreriaJugador {
    pub dinero: i32,
}

#[derive(Resource, Debug, Default)]
pub struct ColaIniciativa {
    pub orden_unidades: Vec<Entity>, // Lista de IDs ordenados por velocidad
    pub indice_actual: usize,        // ¿A quién le toca ahora mismo?
}

// ==========================================
// 3. SISTEMAS (La Lógica del Juego)
// ==========================================

/// Sistema A: Calcula y ordena la cola de iniciativa al empezar el combate
pub fn calcular_linea_de_tiempo(
    query: Query<(Entity, &Estadisticas), With<Unidad>>,
    mut cola: ResMut<ColaIniciativa>,
) {
    let mut unidades: Vec<(Entity, i32)> = query
        .iter()
        .map(|(entity, stats)| (entity, stats.velocidad))
        .collect();

    // Ordenamos de mayor a menor velocidad
    unidades.sort_by(|a, b| b.1.cmp(&a.1));

    cola.orden_unidades = unidades.into_iter().map(|(entity, _)| entity).collect();
    cola.indice_actual = 0;
    
    println!("⏰ Línea de tiempo calculada. Unidades en cola: {:?}", cola.orden_unidades.len());
}

/// Sistema B: Gestiona el movimiento de la unidad activa consumiendo 1 PA
pub fn sistema_movimiento_tactico(
    mut query: Query<(&mut PosicionHex, &mut PuntosAccion), With<Unidad>>,
    cola: Res<ColaIniciativa>,
) {
    // Si la cola está vacía, no hacemos nada
    if cola.orden_unidades.is_empty() { return; }

    // Obtenemos la entidad que tiene el turno actual
    let entidad_activa = cola.orden_unidades[cola.indice_actual];

    // Buscamos los componentes de la unidad activa mediante su ID
    if let Ok((mut posicion, mut pa)) = query.get_mut(entidad_activa) {
        if pa.actuales >= 1 {
            // Ejemplo: El jugador presiona mover al Este (Dirección 1)
            let direccion_este = 1; 
            let nueva_posicion = posicion.0.obtener_vecino(direccion_este);
            
            // Actualizamos los datos
            posicion.0 = nueva_posicion;
            pa.actuales -= 1;
            
            println!("🏃 Unidad activa movida a: {:?}", posicion.0);
        } else {
            println!("🚫 ¡No te quedan Puntos de Acción para moverte!");
        }
    }
}

/// Sistema C: Procesa la economía (Ingresos de casas - Mantenimiento) al terminar la ronda global
pub fn sistema_economia_ronda(
    query: Query<&EconomiaUnidad, With<Unidad>>,
    mut tesoreria: ResMut<TesoreriaJugador>,
    mut query_pa: Query<&mut PuntosAccion, With<Unidad>>,
) {
    let mut costo_total_mantenimiento = 0;

    for econ in query.iter() {
        if !econ.es_heroe {
            costo_total_mantenimiento += econ.costo_mantenimiento as i32;
        }
    }

    // Simulación: El jugador controla 3 casas y cada una da 10 de oro
    let ingresos_casas = 30; 
    let balance_neto = ingresos_casas - costo_total_mantenimiento;
    
    tesoreria.dinero += balance_neto;
    println!("💰 Fin de ronda. Ingresos: +{}, Mantenimiento: -{}. Tesorería actual: {}", ingresos_casas, costo_total_mantenimiento, tesoreria.dinero);

    // Penalización por Bancarrota (Sección 4 del GDD)
    if tesoreria.dinero < 0 {
        tesoreria.dinero = 0;
        println!("⚠️ ¡BANCARROTA! Las unidades entran en Hambruna.");
    }

    // Aprovechamos el fin de ronda para resetear los PA de todos a sus máximos
    for mut pa in query_pa.iter_mut() {
        pa.actuales = pa.maximos;
    }
}
```

---

## 🧠 Explicación de la Arquitectura

* **Componentes Modulares (`#[derive(Component)]`):** No contienen lógica. `PosicionHex` envuelve la coordenada cúbica y `EconomiaUnidad` gestiona el coste de mantenimiento. Esta separación evita conflictos con el *borrow checker* de Rust y facilita reutilizar datos (por ejemplo, darle vida y posición a un edificio sin darle movimiento).
* **Recursos Globales (`#[derive(Resource)]`):** Almacenan información única del estado general de la partida que no pertenece a ninguna unidad individual, como el dinero del banco (`TesoreriaJugador`) o el orden de los turnos (`ColaIniciativa`).
* **Consultas Atómicas (`Query`):** Permiten filtrar el mundo para leer o modificar (`mut`) únicamente los componentes necesarios en cada sistema, habilitando que Bevy ejecute procesos en paralelo de forma segura.
* **Acceso por Identificador (`get_mut`):** Extrayendo el `Entity` activo de la cola de iniciativa, el sistema garantiza que durante un turno individual solo se mutan los datos de la tropa activa, dejando congelado el resto del tablero.
