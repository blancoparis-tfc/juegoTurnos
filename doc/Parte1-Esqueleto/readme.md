# 🦴 Parte 1 — Esqueleto del juego

Guía paso a paso para construir el esqueleto de **Hex-Tactics**, un juego táctico por turnos sobre cuadrícula hexagonal, escrito en **Rust** con el motor **Bevy 0.20**.

Al terminar esta parte tendrás un prototipo jugable con: tablero hexagonal 3x3, movimiento y combate por clics, cola de iniciativa por velocidad, contraataques, combate a distancia, bonos de terreno y clima, ciclo día/noche, subida de nivel, campaña persistente (`.ron`) y un HUD en pantalla.

---

## 📋 Prerrequisitos

* **Rust** estable (el proyecto usa `edition = 2024`). Instálalo desde [rustup.rs](https://rustup.rs) si no lo tienes.
* **Bevy 0.20.1** (se descarga solo como dependencia al compilar).
* Tiempo para la primera compilación: Bevy compila muchas dependencias la primera vez (varios minutos). Las siguientes tardan segundos gracias al perfil `dev` optimizado del paso 001.

> 💡 Concepto previo recomendado: si nunca has visto ECS, lee primero [`doc/conceptos/ecs-bevy.md`](../conceptos/ecs-bevy.md) (entidades, componentes, sistemas, recursos y eventos).

---

## 🧭 Cómo seguir la guía

1. **Sigue los documentos en orden** (001 → 019). Cada paso asume que los anteriores ya están aplicados.
2. **Estructura de cada documento:**
   * *Explicación* del concepto o la fórmula (con ejemplos numéricos cuando hay matemáticas).
   * *Código* con el archivo destino indicado (p. ej. `src/hex_grid.rs`): se añade o sustituye dentro del proyecto creado en el paso 001.
   * *Ejemplos prácticos*: qué debe verse en consola o en pantalla al ejecutar.
3. **Comprueba después de cada paso** con `cargo run --features bevy/dynamic_linking` que el juego sigue funcionando y que los ejemplos prácticos se cumplen.
4. Los mensajes de la consola (con emojis) son la forma principal de depurar durante el desarrollo: si un paso dice "deberías ver `🏃 Unidad movida a ...`" y no aparece, algo no está registrado.

---

## 🗺️ Hoja de ruta: los 19 pasos

### 🧱 Fase 1 — Base del proyecto (001–003)

| Paso | Documento | Qué consigues |
|:----:|-----------|---------------|
| 001 | [Guía de configuración](001-guia-configuracion.md) | Proyecto `cargo new`, dependencias (Bevy, `pathfinding`, `serde`, `ron`), perfiles de compilación rápidos y registro básico de recursos, eventos y sistemas en `main.rs`. |
| 002 | [Lógica hexagonal](002-logica-hexagonal.md) | Estructura `HexCoord` con coordenadas cúbicas (`q + r + s = 0`), distancia sin raíces cuadradas, los 6 vecinos (pointy-topped), tipos de terreno y el tablero como `HashMap`. |
| 003 | [Arquitectura ECS](003-Arquitectura-ecs.md) | Componentes (`Unidad`, `PosicionHex`, `Estadisticas`, `PuntosAccion`, `EconomiaUnidad`), recursos globales (`TesoreriaJugador`, `ColaIniciativa`) y los primeros sistemas: línea de tiempo por velocidad, movimiento y economía de ronda. |

### ⚔️ Fase 2 — Reglas de combate (004)

| Paso | Documento | Qué consigues |
|:----:|-----------|---------------|
| 004 | [Sistema de combate y eventos](004-sistema-combate-eventos-bevy.md) | `EventoAtaque` / `EventoUnidadEliminada`, triángulo de combate (+25%), bono de altura (+15%), flanqueo colectivo que anula la defensa, XP por combate y limpieza de bajas. |

### 🎮 Fase 3 — Pantalla y interacción (005–009)

| Paso | Documento | Qué consigues |
|:----:|-----------|---------------|
| 005 | [Renderizado hexagonal](005-Renderizado-hexagonal.md) | Fórmula hex → píxel (`a_pixel`), dibujo del tablero 3x3 con `RegularPolygon`, capas Z (suelo 0 / indicadores 5 / fichas 10) y sincronización reactiva con `Changed<PosicionHex>`. |
| 006 | [Fórmula inversa y mouse picking](006-Formula-inversa-y-mouse-picking.md) | Píxel → hex con *redondeo cúbico* (nunca se rompe `q + r + s = 0`) y detección de clics con `viewport_to_world_2d`. |
| 007 | [Movimiento con ratón](007-movimiento-unidades-con-raton.md) | `EventoOrdenMovimiento` desacopla entrada y reglas; validación triple: la casilla existe, está a distancia 1 y hay PA disponible. |
| 008 | [Indicadores visuales de rango](008-Indicadores-visuales-de-rango.md) | Anillos cian (`Annulus` semitransparente) sobre las casillas válidas; reactivo: solo se redibujan cuando cambia posición, PA o unidad activa. |
| 009 | [Fin de turnos y ciclo de rondas](009-Fin-de-turnos-y-ciclo-de-rondas.md) | Tecla `Espacio` → `EventoFinTurno` → avanza la `ColaIniciativa`; al completarse la cola se dispara `EventoFinRondaGlobal` (economía + recarga de PA). |

### 🛡️ Fase 4 — Combate completo (010–013)

| Paso | Documento | Qué consigues |
|:----:|-----------|---------------|
| 010 | [Colisiones, bloqueo y combate contextual](010-colisiones-bloqueo-unidades-y-combate-contextual.md) | Un mismo clic decide: casilla libre → moverse, aliado → bloqueo, enemigo → atacar. Anillos rojos sobre enemigos atacables. Se añade el primer enemigo. |
| 011 | [Limpieza de cola de iniciativa](011-limpieza-cola-iniciativa.md) | Al morir una unidad se retira de la cola y el índice de turno se reajusta (evita *index out of bounds* y turnos fantasma). |
| 012 | [Sistema de contraataques](012-sistema-contrataques.md) | El defensor herido devuelve el golpe al 75% de potencia; la bandera `es_contraataque` y `commands.queue` evitan bucles infinitos. |
| 013 | [Combate a distancia](013-Sistema-combate-distancia.md) | Campo `rango_ataque`, función `obtener_casillas_en_rango` (radio hexagonal exacto) y anillos rojos en todo el alcance; los arqueros no sufren contraataque a distancia. |

### 🌍 Fase 5 — Entorno de batalla (014)

| Paso | Documento | Qué consigues |
|:----:|-----------|---------------|
| 014 | [Bonos de terreno, día/noche y clima](014-bono-terreno-ciclo-dia-noche-clima.md) | Recurso `EntornoBatalla`: cobertura de terreno (+5 bosque, +8 montaña), lluvia (-20% daño a distancia), niebla y noche (alcance reducido a 1), alternando cada 3 rondas. |

### 📈 Fase 6 — Campaña y progresión (015–018)

| Paso | Documento | Qué consigues |
|:----:|-----------|---------------|
| 015 | [Persistencia de campaña](015-persistencia-camapaña-serde-ron.md) | Guardado/carga con **Serde + RON** (`F5` / `F9`): veteranos supervivientes, oro y rasgos en `campana_guardada.ron`. |
| 016 | [Despliegue desde la reserva](016-despliegue-desde-la-reserva-de-veteranos.md) | Al arrancar se carga el `.ron` y se instancian los veteranos con sus heridas y XP (fatiga: -2 velocidad), o se crea la escuadra inicial si es la primera partida. |
| 017 | [Subida de nivel](017-sistema-subida-nivel-level-up.md) | Umbral `100 × (nivel + 1)`, mejoras permanentes de vida/daño/defensa (héroes crecen más) y curación en el campo; detectado con `Changed<Experiencia>`. |
| 018 | [Desbloqueo de rasgos por nivel](018-desbloqueo-rasgos-por-nivel.md) | Nivel 2: segunda especialización táctica; Nivel 4 (solo héroes): rasgo maestro. Amplía el triángulo de combate y las opciones de flanqueo. |

### 🖥️ Fase 7 — Interfaz (019)

| Paso | Documento | Qué consigues |
|:----:|-----------|---------------|
| 019 | [Interfaz visual (HUD)](019-interfaz-visual-pantalla.md) | HUD con `bevy::ui`: barra superior (oro, ronda, momento, clima) y panel de la unidad activa (nivel, XP, PV, PA, stats, rasgos), ambos reactivos. |

---

## 🏗️ Resultado final

### Estructura de módulos

```text
hex_tactics/
├── Cargo.toml
├── campana_guardada.ron   <-- generado al pulsar F5
└── src/
    ├── main.rs               <-- Punto de entrada y registro de recursos, eventos y sistemas
    ├── hex_grid.rs           <-- Coordenadas cúbicas, distancias, radios, conversión a píxel
    ├── game_systems.rs       <-- Componentes, turnos, cola de iniciativa, economía
    ├── combat_system.rs      <-- Rasgos, contraataques, combate a distancia, XP, niveles
    ├── render_system.rs      <-- Tablero, mouse picking, anillos de rango
    ├── environment_system.rs <-- Día/noche, clima y bonos de terreno
    ├── save_system.rs        <-- Persistencia Serde/RON y despliegue de veteranos
    └── ui_system.rs          <-- HUD con bevy::ui
```

### Recursos globales

| Recurso | Propósito |
|---------|-----------|
| `TesoreriaJugador` | Oro de la partida |
| `ColaIniciativa` | Orden de turnos (por velocidad) y puntero actual |
| `TableroTactico` | Casillas existentes y su terreno (`HashMap<HexCoord, TipoTerreno>`) |
| `EntornoBatalla` | Ronda actual, día/noche y clima |
| `DatosCampana` | Progreso persistente entre batallas |

### Controles finales

| Entrada | Acción |
|---------|--------|
| **Clic izquierdo** | Mover (casilla libre cian) / Atacar (enemigo rojo) |
| **Espacio** | Terminar el turno actual |
| **F5** | Guardar campaña |
| **F9** | Cargar campaña |

### Eventos que conectan el sistema

`EventoOrdenMovimiento` → `EventoAtaque` → (`EventoUnidadEliminada` → limpieza de cola) → `EventoFinTurno` → `EventoFinRondaGlobal` (economía, recarga de PA, avance de entorno) → `EventoSubidaNivel` (mejoras y desbloqueos) → `EventoGuardarCampana` / `EventoCargarCampana`.

---

## ✅ Checklist de verificación final

Después del paso 019, el juego debe:

- [ ] Arrancar con la escuadra inicial (Héroe + Arquero) o con los veteranos del último guardado.
- [ ] Mostrar los anillos cian (movimiento) y rojos (ataque) de la unidad activa.
- [ ] Resolver combates con triángulo de rasgos, altura, flanqueo y contraataques.
- [ ] Avanzar turnos con `Espacio`, cerrar rondas con economía y alternar día/noche cada 3 rondas.
- [ ] Subir de nivel al superar el umbral de XP y desbloquear rasgos en los hitos.
- [ ] Guardar y restaurar la campaña con `F5` / `F9`.
- [ ] Mostrar todo el estado en el HUD sin refrescar a mano.

---

## 📌 Notas

* La dependencia `pathfinding` ya está incluida en el `Cargo.toml` del paso 001, aunque el movimiento actual es paso a paso (distancia 1); servirá para rutas largas en partes futuras.
* Los mensajes de la consola (🏃, ⚔️, 💰, 🌙, 💾…) son la traza de depuración de cada sistema: úsalos para verificar que cada paso quedó bien integrado.
* Si vas a retomar el proyecto más adelante, empieza por este README, revisa el estado de `main.rs` (donde están registrados todos los sistemas) y sigue los pasos que falten.
