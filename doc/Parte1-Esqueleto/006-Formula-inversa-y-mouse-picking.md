# 🖱️ FÓRMULA INVERSA (PÍXEL ➔ HEXÁGONO) Y MOUSE PICKING: PROYECTO HEX-TACTICS

La **fórmula inversa (Píxel ➔ Hexágono)** es una de las piezas más elegantes de la programación de juegos hexagonales. Sin ella, el juego dibuja el mapa, pero no sabe sobre qué casilla estás haciendo clic con el ratón (*Mouse Picking*).

A diferencia de una cuadrícula cuadrada (donde basta con dividir la posición del ratón entre el ancho de la casilla), en los hexágonos un clic cerca de una esquina diagonal puede pertenecer a tres casillas distintas. Por eso usamos un proceso en dos fases: **conversión a coordenadas decimales** y **redondeo cúbico**.

Aquí tienes la guía paso a paso con sus matemáticas, ejemplos numéricos y el código listo para integrar en tu proyecto de **Rust / Bevy**.

---

## Paso 1: Las Fórmulas Matemáticas (Píxel ➔ Coordenadas Fraccionarias)

Partiendo de las coordenadas en píxeles `(X, Y)` donde el jugador hizo clic y el **Radio (`R`)** del hexágono, despejamos las ecuaciones que usamos para dibujar:

1. **Despejar la fila (`r_frac`):**
   $$r_{\text{frac}} = \frac{-Y}{1.5 \times R}$$
   * **¿Por qué?** Invertimos el signo de `Y` (porque en Bevy `Y` crece hacia arriba, pero las filas `r` crecen hacia abajo) y dividimos entre la altura efectiva de cada fila (`1.5 * R`).

2. **Despejar la columna (`q_frac`):**
   $$q_{\text{frac}} = \frac{X}{R \times \sqrt{3}} - \frac{r_{\text{frac}}}{2}$$
   * **¿Por qué?** Dividimos `X` entre la anchura total del hexágono (`R * √3`) y restamos la mitad de la fila (`r_frac / 2`) para deshacer el desplazamiento en zigzag.

3. **Calcular el tercer eje (`s_frac`):**
   $$s_{\text{frac}} = -q_{\text{frac}} - r_{\text{frac}}$$

El resultado de estas tres operaciones son números con decimales (por ejemplo: `q = 0.85, r = 0.12, s = -0.97`).

---

## Paso 2: El Algoritmo de Redondeo Cúbico (*Cube Round*)

No podemos simplemente redondear cada número por separado al entero más cercano, porque **podríamos romper la ley sagrada de los hexágonos**: $q + r + s = 0$.

Para solucionarlo, aplicamos el algoritmo estándar de **Redondeo Cúbico**:
1. Redondeamos los tres números por separado al entero más cercano: `rq`, `rr`, `rs`.
2. Calculamos cuánto error de redondeo tuvo cada eje (la diferencia absoluta entre el decimal original y el número redondeado):
   * `error_q = |rq - q_frac|`
   * `error_r = |rr - r_frac|`
   * `error_s = |rs - s_frac|`
3. **Descartamos el eje que tuvo el mayor error** y lo recalculamos usando los otros dos ejes para forzar que la suma vuelva a ser exactamente cero:
   * Si `error_q` fue el mayor ➔ `rq = -rr - rs`
   * Si `error_r` fue el mayor ➔ `rr = -rq - rs`
   * Si `error_s` fue el mayor ➔ `rs = -rq - rr`

---

## Paso 3: Ejemplo Matemático Paso a Paso

Imagina que tus hexágonos tienen un radio **`R = 50.0` píxeles** (`√3 ≈ 1.732`). El jugador hace clic con el ratón en el punto **`(X = 75.0, Y = -20.0)`** (cerca del borde derecho de la casilla `(1, 0, -1)`).

### 1. Conversión a decimales
* $r_{\text{frac}} = \frac{-(-20.0)}{1.5 \times 50.0} = \frac{20.0}{75.0} = \mathbf{0.267}$
* $q_{\text{frac}} = \frac{75.0}{50.0 \times 1.732} - \frac{0.267}{2} = \frac{75.0}{86.6} - 0.133 = 0.866 - 0.133 = \mathbf{0.733}$
* $s_{\text{frac}} = -0.733 - 0.267 = \mathbf{-1.000}$

### 2. Redondeo simple inicial
* `rq` = redondear(`0.733`) = **`1`**
* `rr` = redondear(`0.267`) = **`0`**
* `rs` = redondear(`-1.000`) = **`-1`**

### 3. Medición de errores y corrección
* `error_q` = $|1 - 0.733| = \mathbf{0.267}$ (Mayor error)
* `error_r` = $|0 - 0.267| = 0.267$
* `error_s` = $|-1 - (-1.000)| = 0.000$

Como `error_q` es uno de los mayores, lo recalculamos forzando la regla:
* `rq = -rr - rs` ➔ `rq = -0 - (-1) = 1`

**Resultado Final:** El clic en `(75.0, -20.0)` corresponde exactamente al hexágono **`(q: 1, r: 0, s: -1)`**.

---

## Paso 4: Implementación en Rust (`src/hex_grid.rs`)

Añade este método dentro de `impl HexCoord` en tu archivo `src/hex_grid.rs`:

```rust
impl HexCoord {
    // ... tus métodos anteriores (new, distancia_a, a_pixel, etc.) ...

    /// Convierte una posición en píxeles (X, Y) a la coordenada hexagonal exacta
    pub fn desde_pixel(pixel: Vec2, radio: f32) -> Self {
        let sqrt_3 = 3.0_f32.sqrt();

        // 1. Calculamos las coordenadas fraccionarias (con decimales)
        let r_frac = -pixel.y / (1.5 * radio);
        let q_frac = (pixel.x / (radio * sqrt_3)) - (r_frac / 2.0);
        let s_frac = -q_frac - r_frac;

        // 2. Aplicamos el redondeo cúbico para mantener q + r + s = 0
        Self::redondeo_cubico(q_frac, r_frac, s_frac)
    }

    /// Función auxiliar privada que garantiza el redondeo geométrico exacto
    fn redondeo_cubico(q_frac: f32, r_frac: f32, s_frac: f32) -> Self {
        let mut rq = q_frac.round();
        let mut rr = r_frac.round();
        let mut rs = s_frac.round();

        let error_q = (rq - q_frac).abs();
        let error_r = (rr - r_frac).abs();
        let error_s = (rs - s_frac).abs();

        // Descartamos el eje con mayor desviación y lo recalculamos desde los otros dos
        if error_q > error_r && error_q > error_s {
            rq = -rr - rs;
        } else if error_r > error_s {
            rr = -rq - rs;
        } else {
            rs = -rq - rr;
        }

        Self {
            q: rq as i32,
            r: rr as i32,
            s: rs as i32,
        }
    }
}
```

---

## Paso 5: Detectar el Clic del Ratón en Bevy (`src/render_system.rs`)

Ahora podemos crear un sistema en Bevy que lea el clic izquierdo del ratón, convierta las coordenadas de la ventana a coordenadas del mundo 2D (por si mueves la cámara) y te diga en consola qué hexágono tocaste:

```rust
use bevy::window::PrimaryWindow;

/// Sistema Update: Detecta sobre qué hexágono hace clic el jugador
pub fn detectar_clic_tablero(
    botones_raton: Res<ButtonInput<MouseButton>>,
    q_ventana: Query<&Window, With<PrimaryWindow>>,
    q_camara: Query<(&Camera, &GlobalTransform)>,
) {
    // Solo ejecutamos la lógica en el fotograma exacto en que se pulsa el botón izquierdo
    if botones_raton.just_pressed(MouseButton::Left) {
        let Ok(ventana) = q_ventana.single() else { return; };
        let Ok((camara, camara_transform)) = q_camara.single() else { return; };

        // 1. Obtenemos la posición del cursor dentro de la ventana
        if let Some(pos_cursor) = ventana.cursor_position() {
            // 2. Traducimos la posición de la ventana a coordenadas del mundo 2D de Bevy
            if let Ok(pos_mundo) = camara.viewport_to_world_2d(camara_transform, pos_cursor) {
                // 3. Aplicamos nuestra fórmula inversa
                let hex_seleccionado = HexCoord::desde_pixel(pos_mundo, RADIO_HEXAGONO);
                
                println!("🖱️ Clic en píxel {:?} ➔ Hexágono: {:?}", pos_mundo, hex_seleccionado);
            }
        }
    }
}
```

*(Recuerda registrar `detectar_clic_tablero` dentro de `.add_systems(Update, (...))` en tu archivo `src/main.rs`).*

## Claves Técnicas de este Sistema
* **`viewport_to_world_2d`:** Las ventanas de sistema operativo miden los píxeles desde la esquina superior izquierda `(0, 0)`, mientras que Bevy pone el `(0, 0)` en el centro de la pantalla. Esta función nativa de Bevy hace esa conversión por ti automáticamente, incluso si haces zoom o desplazas la cámara por el mapa.
* **Precisión en bordes:** Gracias a `redondeo_cubico`, incluso si el jugador hace clic a un milímetro de la punta entre tres hexágonos distintos, el código nunca devolverá una coordenada inválida ni romperá la simetría matemática del tablero.
