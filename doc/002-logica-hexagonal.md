# 🗺️ GUÍA DE LÓGICA HEXAGONAL EN RUST: PROYECTO HEX-TACTICS

---

## 1. 🔍 ¿Qué es la Lógica Hexagonal?
Es el conjunto de reglas matemáticas y geométricas utilizadas para gestionar tableros de 6 lados en videojuegos. A diferencia de las cuadrículas cuadradas tradicionales, los hexágonos ofrecen una **distancia uniforme**: la separación entre el centro de una casilla y cualquiera de sus 6 vecinas es exactamente la misma. Esto elimina las deformaciones de movimiento y alcance que ocurren con las diagonales en los cuadrados.

* **Orientación (Pointy-topped):** Hexágonos con la punta hacia arriba. Es ideal para pantallas panorámicas en 2D.
* **Coordenadas Cúbicas (q, r, s):** Se utilizan 3 ejes matemáticos que mantienen una propiedad inviolable en todo momento: **q + r + s = 0**. Esto simplifica drásticamente el cálculo de distancias y trayectorias.

---

## 2. 📍 Ejemplo Práctico: Mapa de 3x3 (9 Hexágonos)
Debido a la geometría hexagonal, las casillas no se alinean verticalmente en línea recta. Cada fila sucesiva debe desplazarse hacia la izquierda para encajar perfectamente como azulejos. 

Así se distribuyen las coordenadas exactas `(q, r, s)` en el espacio para formar un bloque visual de 3 filas de 3:

```text
       Fila 0 (r = 0):           [ 0, 0, 0 ]      [ 1, 0, -1 ]     [ 2, 0, -2 ]
       
       Fila 1 (r = 1):    [ -1, 1, 0 ]      [ 0, 1, -1 ]      [ 1, 1, -2 ]
       
       Fila 2 (r = 2):  [ -2, 2, 0 ]      [ -1, 2, -1 ]     [ 0, 2, -2 ]
```

### Ejemplo de Movimiento Vectorial
Si una unidad está en el centro `(0, 1, -1)` y se desplaza hacia el **Este** (cuyo vector fijo es `(1, -1, 0)`), la operación matemática es una suma directa de componentes:
* `Nueva Q` = 0 + 1 = 1
* `Nueva R` = 1 + (-1) = 0
* `Nueva S` = -1 + 0 = -1

**Resultado:** La unidad se mueve de manera exacta a la casilla `(1, 0, -1)` (Fila 0, columna del medio).

---

## 3. 🛠️ Código Completo en Rust (`hex_grid.rs`)

Este código implementa de forma segura el struct de coordenadas, la sobrecarga del operador de suma, el cálculo de distancia y la obtención de vecinos.

```rust
use std::collections::HashMap;
use std::ops::Add;

// 1. Estructura de la coordenada hexagonal cúbica
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct HexCoord {
    pub q: i32,
    pub r: i32,
    pub s: i32,
}

impl HexCoord {
    // Constructor seguro: calcula 's' automáticamente para garantizar q + r + s = 0
    pub fn new(q: i32, r: i32) -> Self {
        Self { q, r, s: -q - r }
    }

    // 2. Cálculo determinista de distancia entre dos casillas
    pub fn distancia_a(&self, destino: &HexCoord) -> u32 {
        let dq = (self.q - destino.q).abs();
        let dr = (self.r - destino.r).abs();
        let ds = (self.s - destino.s).abs();
        
        dq.max(dr).max(ds) as u32
    }

    // 3. Obtener la lista completa de los 6 vecinos adyacentes
    pub fn obtener_todos_los_vecinos(&self) -> Vec<HexCoord> {
        VECINOS_DIRECCIONALES
            .iter()
            .map(|&offset| *self + offset)
            .collect()
    }
}

// Sobrecarga del operador suma (+) para permitir hacer: hex_actual + desplazamiento
impl Add for HexCoord {
    type Output = Self;

    fn add(self, other: Self) -> Self {
        Self {
            q: self.q + other.q,
            r: self.r + other.r,
            s: self.s + other.s,
        }
    }
}

// 4. Constantes fijas de desplazamiento para las 6 direcciones (Pointy-topped)
const VECINOS_DIRECCIONALES: [HexCoord; 6] = [
    HexCoord { q: 1,  r: 0,  s: -1 }, // Dirección 0: Noreste
    HexCoord { q: 1,  r: -1, s: 0  }, // Dirección 1: Este
    HexCoord { q: 0,  r: -1, s: 1  }, // Dirección 2: Sureste
    HexCoord { q: -1, r: 0,  s: 1  }, // Dirección 3: Suroeste
    HexCoord { q: -1, r: 1,  s: 0  }, // Dirección 4: Oeste
    HexCoord { q: 0,  r: 1,  s: -1 }, // Dirección 5: Noroeste
];

// 5. Estructuras del mundo para la integración con el motor
pub enum TipoTerreno { 
    Llanura, 
    Bosque, 
    Montana, 
    Casa 
}

pub struct TableroTactico {
    pub casillas: HashMap<HexCoord, TipoTerreno>,
}
```

---

## 4. 🧠 Explicación de los Componentes Clave

* **`s: -q - r` (Garantía Arquitectónica):** Al forzar el cálculo del tercer eje en el constructor, Rust impide a nivel de compilación que un error humano rompa la simetría tridimensional indispensable del hexágono.
* **`distancia_a` (Eficiencia de Cómputo):** Resta los componentes de las dos coordenadas y devuelve el número absoluto máximo. Al no utilizar raíces cuadradas, el motor de interfaz puede evaluar rangos de ataque de decenas de unidades instantáneamente sin caídas de rendimiento.
* **`impl Add for HexCoord` (Azúcar Sintáctico):** Permite usar el operador `+` nativo para aplicar los vectores fijos de movimiento de `VECINOS_DIRECCIONALES` de forma limpia y legible.
* **`HashMap<HexCoord, TipoTerreno>` (Almacenamiento Óptimo):** En lugar de reservar memoria para una matriz bidimensional gigante con casillas vacías (ineficiente para mapas irregulares), el tablero se indexa mediante las coordenadas como llave. El motor Bevy puede validar si una casilla existe u obtener sus modificadores de terreno en tiempo constante $\mathcal{O}(1)$.
