# ⚙️ ARQUITECTURA ECS EN BEVY (RUST): PROYECTO HEX-TACTICS

Este documento contiene la implementación base y la explicación detallada de la arquitectura **Entity Component System (ECS)** de Bevy para el juego **Hex-Tactics**. En este paradigma, los datos puros (**Componentes**) están completamente separados de la lógica (**Sistemas**).

---

## 🛠️ 1. Código Fuente (`game_systems.rs`)

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

## 🧠 2. Desglose de la Arquitectura

### 1. Los Componentes (Los Datos Puros)
En Rust tradicional, crearías una estructura enorme llamada `Soldado` con todo dentro. En Bevy, creamos piezas de Lego muy pequeñas llamadas **Componentes** (`#[derive(Component)]`) y se las pegamos a una entidad.

* **`pub struct Unidad;`**: Es un "componente marcador" (no tiene datos por dentro). Sirve simplemente como una etiqueta para que Bevy sepa diferenciar entre un soldado y un árbol o una nube en el mapa.
* **`pub struct PosicionHex(pub HexCoord);`**: Envuelve la coordenada matemática `(q, r, s)` que programamos antes. Si una entidad tiene este componente, significa que ocupa un lugar físico en el tablero.
* **`Estadisticas`, `PuntosAccion`, `EconomiaUnidad`**: Son contenedores de números simples. Nota cómo `EconomiaUnidad` tiene el campo booleano `es_heroe` para aplicar la regla de tu diseño donde los héroes tienen coste cero.

### 2. Los Recursos (La Memoria Global)
Mientras que los componentes pertenecen a una unidad individual, los **Recursos** (`#[derive(Resource)]`) son datos únicos que pertenecen a toda la partida en general.

* **`TesoreriaJugador`**: Es la cuenta bancaria global donde se guarda el dinero que producen tus casas.
* **`ColaIniciativa`**: Es el cerebro de tus turnos. Guarda una lista (`Vec<Entity>`) con los identificadores numéricos de todas las unidades ordenadas por velocidad, y un puntero (`indice_actual`) que señala a quién le toca moverse en este preciso instante.

### 3. Los Sistemas (El Motor de Reglas)
Los sistemas son funciones normales de Rust. Bevy se encarga de ejecutarlas automáticamente en cada fotograma o cuando cambias de estado, pasándoles los datos que piden en sus parámetros.

* **Sistema A (`calcular_linea_de_tiempo`):** Pide a Bevy el ID (`Entity`) y las `Estadisticas` de todo lo que tenga la etiqueta `Unidad`, las ordena de mayor a menor velocidad y guarda esa lista en `ColaIniciativa`.
* **Sistema B (`sistema_movimiento_tactico`):** Consulta la cola para averiguar cuál es el ID de la unidad activa y usa `query.get_mut(entidad_activa)` para modificar *únicamente* a esa tropa. Si tiene al menos 1 Punto de Acción, calcula su casilla vecina, actualiza su coordenada y le resta 1 PA, dejando intacto el resto del tablero.
* **Sistema C (`sistema_economia_ronda`):** Recorre todas las unidades con un bucle `for`. Si `!econ.es_heroe`, suma su coste de mantenimiento a la factura total, aplica el balance con los ingresos de las casas, dispara la alerta de *Hambruna* si la tesorería cae por debajo de cero y rellena los Puntos de Acción al máximo para la siguiente ronda global.
