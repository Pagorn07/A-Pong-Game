# Documento de Contexto — Pong (Godot)

## 1. Objetivo del proyecto

Hacer un juego de 0 a fin, simple pero cuidado visualmente ("un bonito simple"). El objetivo es doble:

- **Terminar** un proyecto pequeño de principio a fin, recuperando la sensación de completar algo.
- Usar este proyecto de bajo riesgo como **caja de práctica de patrones y buenas prácticas** (arquitectura, arte coherente, sonido propio) que luego se puedan aplicar con más soltura al proyecto grande (el juego de pesca).

## 2. Alcance — qué SÍ y qué NO

**SÍ incluye:**
- Pong clásico: dos palas, una bola, marcador, victoria a X puntos.
- Modo jugador vs CPU y modo jugador vs jugador (local).
- Menú inicial con: "Jugar contra CPU", "Jugar contra persona", "Salir".
- Sonidos simples (rebote pala, rebote pared, punto marcado, fin de partida), creados con jsfxr.
- Pausa (reanudar / volver al menú).
- Pantalla de fin de partida (anuncia ganador, opciones de reiniciar o volver al menú).
- Soporte de teclado y mando mediante Input Map.
- Configuración básica de stretch mode para que funcione razonablemente en distintas resoluciones/dispositivos (ej. PC y Steam Deck), sin trabajo adicional de responsive design.

**NO incluye (fuera de alcance deliberadamente):**
- Multijugador online.
- Power-ups o mecánicas añadidas al Pong clásico.
- Música de fondo (no le pega al tono del juego).
- Object pooling, sistema de guardado/carga (no hay progreso persistente que guardar).
- Responsive design avanzado más allá del stretch mode básico de Godot.

## 3. Mecánicas del juego

**Bola:**
- Movimiento y rebote gestionados manualmente por código (no por física del motor), para tener control total sobre el comportamiento.
- Velocidad inicial baja; aumenta un porcentaje cada vez que golpea una pala, hasta un **tope máximo** definido (para evitar velocidad infinita/injugable).
- Rebote contra **paredes** (techo/suelo): reflexión especular + un pequeño ángulo aleatorio (jitter) para romper la sensación de rebote perfectamente matemático, sin afectar a la habilidad del jugador.
- Rebote contra **pala**: el ángulo de salida vertical depende del punto de impacto respecto al centro de la pala (igual que en Arkanoid/Breakout) — no es un rebote espejo. El eje horizontal simplemente se invierte. Esto da profundidad de control al jugador sin necesitar aleatoriedad añadida.
- Saque (inicio de partida o tras cada punto): ángulo aleatorio dentro de un rango razonable (ni completamente horizontal ni casi vertical).
- Tras un punto: la bola vuelve al centro, se espera un breve tiempo (para dar margen al jugador humano), y se relanza con nuevo ángulo aleatorio.

**Palas:**
- Velocidad de movimiento constante (sin aceleración/inercia).
- Movimiento limitado a la zona de juego por código (clamp de posición), no mediante colisión física.
- Misma escena reutilizada para ambas palas (`Paddle.tscn`), con un modo de control configurable (jugador o CPU) en vez de duplicar código.

**CPU:**
- Sigue la posición Y de la bola con un margen de error/retraso, para que no sea perfecta ni imbatible.

## 4. Arquitectura de escenas y nodos

**Escenas principales:**
- `Menu.tscn` — título "PONG" + botones: Jugar contra CPU / Jugar contra persona / Salir.
- `Game.tscn`:
  - `Ball` — movimiento y rebote manual por código.
  - `PaddleLeft` / `PaddleRight` — instancias de la misma escena `Paddle.tscn`, con `control_mode` configurable (PLAYER / CPU).
  - `ScoreUI` — una única escena con dos Labels (marcador izquierda/derecha).
  - `GoalLeft` / `GoalRight` — instancias de la misma escena `Goal.tscn` (Area2D), cada una con un lado asignado; detectan la bola y emiten señal de gol.
- **GameManager (autoload)** — guarda el modo elegido en el menú (CPU / 2 jugadores) y coordina el flujo general de la partida.

**Patrones y técnicas a aplicar (justificados, no por acumular):**

1. **State Machine** — estados: `READY` (esperando saque), `PLAYING`, `GOAL_SCORED` (breve, opcional fusionar con READY), `GAME_OVER`, `PAUSED`. Controla qué lógica/input está activo en cada momento, evitando banderas booleanas sueltas.
2. **Signals** — comunicación entre nodos sin acoplamiento directo (ej. `Goal` avisa de un gol sin conocer a quién).
3. **Event Bus** (autoload dedicado solo a señales globales) — practicado deliberadamente aquí, aunque el Pong no lo necesite estrictamente, para dominarlo antes de aplicarlo al juego de pesca (donde el `GameManager` tiende a acumular demasiadas responsabilidades).
4. **Resources** — `GameConfig.tres`: velocidad inicial de la bola, incremento por golpe, velocidad máxima, puntos para ganar. Editable desde el inspector sin tocar código.
5. **Dependency Injection**:
   - Vía `@export` para dependencias conocidas de antemano (ej. `Ball` recibe su `GameConfig` como recurso arrastrado en el inspector).
   - Vía función `setup()` para dependencias que solo existen en tiempo de ejecución (ej. qué `control_mode` tiene cada `Paddle`, decidido según lo elegido en el menú).
6. **Input Map** (sistema nativo de Godot) — acciones nombradas (`p1_up`, `p1_down`, etc.) asociadas simultáneamente a teclado y mando, para que el control_node de cada pala no necesite saber de qué dispositivo viene el input.
7. **Composición sobre herencia** — una sola escena `Paddle.tscn` y una sola `Goal.tscn`, reutilizadas con comportamiento configurable en vez de duplicar o heredar.

**Nota sobre Walls:** no existen como nodo físico. Los límites superior/inferior de la zona de juego son valores numéricos usados directamente en el código de la bola para decidir cuándo rebota.

## 5. Estructura de carpetas

```
res://
├── scenes/
│   ├── menu/
│   │   └── Menu.tscn (+ Menu.gd)
│   ├── game/
│   │   ├── Game.tscn (+ Game.gd)
│   │   ├── Ball.tscn (+ Ball.gd)
│   │   ├── Paddle.tscn (+ Paddle.gd)
│   │   ├── Goal.tscn (+ Goal.gd)
│   │   └── ScoreUI.tscn (+ ScoreUI.gd)
├── resources/
│   └── GameConfig.tres (+ GameConfig.gd)
├── assets/
│   ├── sprites/
│   └── sounds/
└── autoloads/
    ├── GameManager.gd
    └── EventBus.gd
```

Convención: cada script vive junto a su escena correspondiente (no hay carpeta `scripts/` centralizada).

## 6. Arte y sonido

- **Resolución interna de referencia:** baja resolución tipo retro (ej. 320x180), escalada hacia arriba mediante stretch mode de Godot. Refuerza la estética minimalista y es más simple de dibujar.
- **Paleta de colores:** estilo clásico Pong (1972) — fondo negro, blanco para palas/bola/marcador/línea central. Un tercer color de acento (amarillo brillante, ej. `#F7D51D`) usado con moderación para detalles puntuales (marcador al sumar punto, posible flash al acelerar la bola).
- **Herramientas:**
  - **Piskel** (gratis, navegador) para sprites en pixel art.
  - **jsfxr** (gratis, navegador) para efectos de sonido retro/8-bit.

## 7. Criterios de "terminado"

- [ ] Menú funcional con las 3 opciones (CPU / 2 jugadores / Salir)
- [ ] Partida jugable completa: saque, rebotes (pared con jitter, pala con ángulo por punto de impacto), aceleración progresiva con tope máximo
- [ ] Sistema de puntuación funcionando (ambos lados) y condición de victoria a X puntos
- [ ] CPU funcional con el retraso/margen de error definido
- [ ] Pausa funcional (reanudar / volver al menú)
- [ ] Pantalla de fin de partida: anuncia qué jugador ha ganado, con opciones de "Reiniciar partida" y "Volver al menú"
- [ ] Sonidos básicos integrados (rebote pala, rebote pared, punto, fin de partida) hechos con jsfxr
- [ ] Arte coherente aplicado (paleta blanco/negro/acento, resolución definida) sustituyendo cualquier placeholder
- [ ] Funciona correctamente en al menos 2 resoluciones distintas probadas (stretch mode)
- [ ] Sin bugs conocidos que rompan una partida (ej. bola atraviesa pala, marcador no sube, etc.)
