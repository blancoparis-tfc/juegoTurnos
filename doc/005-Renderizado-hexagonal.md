# 🎨 GUÍA COMPLETA DE RENDERIZADO HEXAGONAL: PROYECTO HEX-TACTICS

Para dibujar nuestro tablero en la pantalla de **Bevy**, debemos construir un puente entre dos mundos: el **espacio lógico** (coordenadas cúbicas `q, r, s`) y el **espacio visual** (píxeles `X, Y` en el monitor).

---

## Paso 1: Entender la Geometría del Hexágono (*Pointy-Topped*)

Como elegimos hexágonos con la **punta hacia arriba**, cada pieza tiene proporciones trigonométricas exactas basadas en su **Radio (`R`)** (la distancia desde el centro del hexágono hasta cualquiera de sus 6 vértices).

* **Altura total:** Es exactamente el doble del radio (`2 * R`).
* **Anchura total:** Por trigonometría básica (triángulos equiláteros internos de 60°), la distancia de lado plano a lado plano es `√3 * R` (aproximadamente `1.732 * R`).

### Explicación de las Fórmulas de Traducción (Hexágono ➔ Píxel)

1. **Cálculo del Eje Horizontal (`X`):**
   `X = R * √3 * (q + r / 2.0)`
   * **Explicación paso a paso:** Cada paso en la columna `q` mueve el centro exactamente una anchura completa (`R * √3`). Sin embargo, debido a la forma de seis lados, cuando bajas a la siguiente fila (`r`), las casillas no caen en línea recta, sino que encajan en los huecos de arriba. Por eso sumamos `r / 2.0`: cada vez que `r` aumenta en 1, el hexágono se desplaza **media anchura** hacia la derecha.

2. **Cálculo del Eje Vertical (`Y`):**
   `Y = -R * 1.5 * r`
   * **Explicación paso a paso:** Si los hexágonos se apilaran uno encima de otro sin encajar sus puntas, la distancia vertical sería `2 * R`. Como sus vértices superiores e inferiores se entrelazan, las filas se acercan un 25%, dejando una separación exacta de tres cuartos de su altura (`1.5 * R`). 
   * **¿Por qué el signo negativo (`-`)?** En la lógica de tableros, la fila `r = 1` está *debajo* de la fila `r = 0`. Pero en **Bevy**, las coordenadas `Y` positivas van hacia *arriba* de la pantalla. Invertir el signo garantiza que el mapa crezca hacia abajo de forma natural.

---

## Paso 2: Ejemplos Matemáticos Numéricos

Supongamos que definimos un tamaño de **`R = 50.0` píxeles** para cada casilla (`√3 ≈ 1.732`, por lo tanto una anchura completa es `50 * 1.732 = 86.6 píxeles`).

### Ejemplo A: El Origen `(q: 0, r: 0, s: 0)`
* **Matemática en X:** `50 * 1.732 * (0 + 0 / 2) = 86.6 * 0 = 0.0 px`
* **Matemática en Y:** `-50 * 1.5 * 0 = 0.0 px`
* **Explicación:** Esta casilla actúa como el ancla del universo. Se dibuja en el centro exacto de la cámara de Bevy `(0.0, 0.0)`.

### Ejemplo B: Casilla contigua a la derecha `(q: 1, r: 0, s: -1)`
* **Matemática en X:** `50 * 1.732 * (1 + 0 / 2) = 86.6 * 1 = 86.6 px`
* **Matemática en Y:** `-50 * 1.5 * 0 = 0.0 px`
* **Explicación:** Al mantenerse en la fila `0`, el valor de `Y` no cambia. En `X`, se desplaza exactamente una anchura completa (`86.6 px`), quedando pegada al borde derecho de la casilla anterior.

### Ejemplo C: Primera casilla de la segunda fila `(q: -1, r: 1, s: 0)`
* **Matemática en X:** `50 * 1.732 * (-1 + 1 / 2) = 86.6 * (-0.5) = -43.3 px`
* **Matemática en Y:** `-50 * 1.5 * 1 = -75.0 px`
* **Explicación:** Como `r` vale `1`, baja `75 píxeles` en vertical. Para formar nuestro bloque cuadrado de 3x3 sin que el mapa se incline en diagonal, restamos 1 a `q`. La fórmula compensa el desplazamiento y coloca la casilla exactamente media anchura a la izquierda (`-43.3 px`).

### Ejemplo D: El centro del mapa de 3x3 `(q: 0, r: 1, s: -1)`
* **Matemática en X:** `50 * 1.732 * (0 + 1 / 2) = 86.6 * 0.5 = 43.3 px`
* **Matemática en Y:** `-50 * 1.5 * 1 = -75.0 px`
* **Explicación:** Al tener `q = 0` y `r = 1`, queda situada en `X = 43.3 px`, que es justo la mitad entre `0.0 px` (Ejemplo A) y `86.6 px` (Ejemplo B). Encaja matemáticamente en el hueco inferior de las dos casillas de arriba.

---

## Paso 3: Implementación Matemática en Rust (`src/hex_grid.rs`)

Añadimos este método dentro de `impl HexCoord` para que cualquier coordenada sepa traducirse a sí misma a píxeles:

```rust
use bevy::math::Vec2;

impl HexCoord {
    /// Traduce coordenadas cúbicas (q, r) a píxeles 2D en pantalla (X, Y)
    pub fn a_pixel(&self, radio: f32) -> Vec2 {
        // Calculamos la raíz cuadrada de 3 en punto flotante de 32 bits
        let sqrt_3 = 3.0_f32.sqrt();
        
        // Aplicamos las ecuaciones geométricas convirtiendo los enteros (i32) a decimales (f32)
        let x = radio * sqrt_3 * (self.q as f32 + self.r as f32 / 2.0);
        let y = -radio * 1.5 * (self.r as f32);
        
        // Devolvemos un vector bidimensional listo para usar en el motor gráfico
        Vec2::new(x, y)
    }
}
```
* **Explicación del código:** Rust es muy estricto con los tipos numéricos y no permite multiplicar enteros (`i32`) con decimales (`f32`) por accidente. Por eso usamos `as f32` para convertir las coordenadas de cuadrícula en medidas de pantalla continuas y devolvemos un `Vec2`, que es el tipo nativo de Bevy para posiciones 2D.

---

## Paso 4: El Sistema de Dibujo y Sincronización (`src/render_system.rs`)

Creamos este archivo para generar las mallas visuales del tablero y mantener las fichas de las unidades sincronizadas con sus datos lógicos:

```rust
use bevy::prelude::*;
use crate::hex_grid::HexCoord;
use crate::game_systems::{PosicionHex, Unidad};

// Definimos el radio como constante global para que todo el juego use la misma escala
pub const RADIO_HEXAGONO: f32 = 50.0;

#[derive(Component)]
pub struct CasillaVisual;

/// Sistema Startup: Genera los 9 hexágonos del mapa de prueba (3x3)
pub fn dibujar_tablero_inicial(
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<ColorMaterial>>,
) {
    // 1. Creamos la geometría base en la memoria de la tarjeta gráfica (GPU)
    let forma_hex = meshes.add(RegularPolygon::new(RADIO_HEXAGONO * 0.94, 6));
    
    // 2. Registramos los materiales de color
    let color_llanura = materials.add(Color::srgb(0.35, 0.55, 0.30));
    let color_colina = materials.add(Color::srgb(0.55, 0.45, 0.30));

    // 3. Generamos la cuadrícula de 3 filas por 3 columnas
    for r in 0..3 {
        for q in (0 - r)..(3 - r) {
            let coord = HexCoord::new(q, r);
            let pos_pixel = coord.a_pixel(RADIO_HEXAGONO);

            // Asignamos color marrón a la casilla central (0, 1) para simular elevación
            let material = if q == 0 && r == 1 {
                color_colina.clone()
            } else {
                color_llanura.clone()
            };

            // 4. Instanciamos la casilla en el mundo de Bevy
            commands.spawn((
                CasillaVisual,
                Mesh2d(forma_hex.clone()),
                MeshMaterial2d(material),
                Transform::from_xyz(pos_pixel.x, pos_pixel.y, 0.0),
            ));
        }
    }
}

/// Sistema Update: Mueve el sprite solo cuando cambia su PosicionHex
pub fn sincronizar_visuales_unidades(
    mut query: Query<(&PosicionHex, &mut Transform), (With<Unidad>, Changed<PosicionHex>)>,
) {
    for (pos_hex, mut transform) in query.iter_mut() {
        let pixel = pos_hex.0.a_pixel(RADIO_HEXAGONO);
        transform.translation = Vec3::new(pixel.x, pixel.y, 10.0);
    }
}
```

### Explicación Detallada de cada bloque del Sistema

1. **`RegularPolygon::new(RADIO_HEXAGONO * 0.94, 6)`:** Crea un polígono regular de 6 lados con la punta hacia arriba. Multiplicamos por `0.94` (94% de su tamaño) para dejar un margen del 6% entre casillas; así el fondo oscuro de la ventana actúa automáticamente como el borde separador de la cuadrícula.
2. **`meshes.add(...)` y `.clone()`:** En Bevy, subir una forma geométrica a la tarjeta gráfica es costoso. Al guardarla en `Assets<Mesh>`, obtenemos un identificador ligero (`Handle`). Usar `.clone()` dentro del bucle no duplica la geometría en memoria, solo comparte la referencia, logrando que dibujar miles de hexágonos sea instantáneo.
3. **Jerarquía del Eje `Z` (`0.0` vs `10.0`):** En gráficos 2D, Bevy utiliza la tercera dimensión (`Z`) para decidir qué se dibuja por encima. Colocamos el suelo en `Z = 0.0` y las unidades en `Z = 10.0`. Esto deja libre el espacio intermedio (`Z = 5.0`) para dibujar en el futuro indicadores de selección o rutas de movimiento sin que queden tapados.
4. **El filtro `Changed<PosicionHex>`:** Este filtro es una de las mayores ventajas de ECS. Le indica a Bevy que **ignore** este sistema en todos los fotogramas en los que las unidades estén quietas. Solo cuando el sistema de movimiento altere la coordenada lógica de una tropa, Bevy ejecutará la conversión a píxeles y actualizará su `Transform` en pantalla.
