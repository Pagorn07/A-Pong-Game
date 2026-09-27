# Documento de Contexto — Pong (Godot)

## 0. Setup del proyecto

- **Motor:** Godot 4.7.2
- **Repositorio:** https://github.com/Pagorn07/A-Pong-Game

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
  - `GameStateMachine` — nodo hijo directo de `Game.tscn` (no es una escena propia, solo un `Node` con el script `GameStateMachine.gd`, ya que no tiene estructura interna propia). Vive y muere con la partida; el menú no tiene ni necesita estados.
- **GameManager (autoload)** — guarda datos que sobreviven al cambio de escena (modo elegido en el menú: CPU / 2 jugadores). No conoce ni gestiona señales; solo almacena configuración.
**Patrones y técnicas a aplicar (justificados, no por acumular):**
 
1. **State Machine** — nodo `GameStateMachine` dentro de `Game.tscn`. Estados: `READY` (esperando saque), `PLAYING`, `GOAL_SCORED` (breve, opcional fusionar con READY), `GAME_OVER`, `PAUSED`. Controla qué lógica/input está activo en cada momento, evitando banderas booleanas sueltas. Al cambiar de estado emite `EventBus.state_changed.emit(new_state)`.
2. **Signals** — comunicación entre nodos sin acoplamiento directo (ej. `Goal` avisa de un gol sin conocer a quién).
3. **Event Bus** (autoload dedicado solo a señales globales) — practicado deliberadamente aquí, aunque el Pong no lo necesite estrictamente, para dominarlo antes de aplicarlo al juego de pesca (donde el `GameManager` tiende a acumular demasiadas responsabilidades).
   - **Criterio de uso:** una señal va al `EventBus` cuando le interesa a más de un sistema que no tiene por qué conocerse entre sí (ej. `goal_scored`, `state_changed`). Si solo le interesa a su padre directo en la escena, se conecta local, nodo-a-nodo (ej. colisión física `Ball`↔`Paddle`).
   - **Flujo:** el `EventBus` solo *declara* las señales globales (es un tablón de anuncios pasivo, no un intermediario activo). Quien tiene algo que anunciar emite directamente la señal del bus (`EventBus.goal_scored.emit(side)`, sin señal local propia intermedia). Quien está interesado se suscribe por su cuenta en su `_ready()` (`EventBus.goal_scored.connect(_on_goal_scored)`). Ni el emisor conoce a los suscriptores, ni los suscriptores conocen al emisor.
4. **Resources** — `GameConfig.tres`: velocidad inicial de la bola, incremento por golpe, velocidad máxima, puntos para ganar. Editable desde el inspector sin tocar código.
5. **Dependency Injection**:
   - Vía `@export` para dependencias conocidas de antemano (ej. `Ball` recibe su `GameConfig` como recurso arrastrado en el inspector).
   - Vía función `setup()` para dependencias que solo existen en tiempo de ejecución (ej. qué `control_mode` tiene cada `Paddle`, decidido según lo elegido en el menú).
6. **Input Map** (sistema nativo de Godot) — acciones nombradas (`p1_up`, `p1_down`, etc.) asociadas simultáneamente a teclado y mando, para que el control_node de cada pala no necesite saber de qué dispositivo viene el input.
7. **Composición sobre herencia** — una sola escena `Paddle.tscn` y una sola `Goal.tscn`, reutilizadas con comportamiento configurable en vez de duplicar o heredar.
**Nota sobre Walls:** no existen como nodo físico. Los límites superior/inferior de la zona de juego son valores numéricos usados directamente en el código de la bola para decidir cuándo rebota.

## 5. Estructura de carpetas
 
Raíz del repositorio (fuera de `res://`, archivos estándar de Godot/Git, sin contenido de juego):
```
.godot/                    (autogenerada, ignorada por git)
.gitattributes
.gitignore
icon.svg / icon.svg.import
LICENSE
project.godot
pong_documento_contexto.md
README.md
```
 
Dentro de `res://` (contenido real del proyecto):
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
│   │   ├── ScoreUI.tscn (+ ScoreUI.gd)
│   │   └── GameStateMachine.gd (script suelto, hijo de Game.tscn, sin .tscn propio)
├── resources/
│   └── GameConfig.tres (+ GameConfig.gd)
├── assets/
│   ├── sprites/
│   └── sounds/
├── autoloads/
│   ├── GameManager.gd
│   └── EventBus.gd
├── icon.svg
├── pong_documento_contexto.md
└── README.md
```
 
Convención: cada script vive junto a su escena correspondiente (no hay carpeta `scripts/` centralizada). `pong_documento_contexto.md` y `README.md` se mantienen duplicados en la raíz del repo (para que se vean en GitHub) y dentro de `res://` (para tenerlos a mano desde el propio editor de Godot).

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
- [ ] Documento de contexto y README reflejan el estado final del proyecto (sin secciones desactualizadas, valores reales de `GameConfig` anotados, registro de decisiones al día)

## 8. Convenciones
 
### 8.1 Commits
 
Conventional Commits simplificado. Un commit por cambio lógico, no por sesión de trabajo.
 
Prefijos usados: `feat`, `fix`, `refactor`, `docs`, `chore`, `art` (assets/sprites/sonido).
 
```
feat: bola con rebote en paredes y jitter
fix: pala se sale del área de juego
refactor: mover clamp de posición a función propia
docs: actualizar README con instrucciones de build
chore: configurar input map
art: sprites de palas y bola en paleta final
```
 
### 8.2 Código (GDScript)
 
- **Nombres:** `PascalCase` para clases/nodos (`PaddleLeft`), `snake_case` para variables/funciones (`control_mode`, `_on_ball_entered`), `CONSTANTE_GRITADA` para constantes (`MAX_SPEED`).
- **Señales:** nombradas en pasado (`goal_scored`, no `on_goal` ni `score_goal`) — convención oficial de Godot, evita confundir la señal con su handler.
- **Orden dentro de un script:** `@export` vars → variables normales → `_ready()` / `_process()` → funciones públicas → funciones privadas (prefijo `_`).
- **Tipado estático siempre que se pueda** (`var speed: float = 100.0`, `func move(delta: float) -> void:`).
- **Indentación:** tabs.
### 8.3 Escenas, nodos y assets
 
- **Escenas:** `PascalCase.tscn` (`Paddle.tscn`, `ScoreUI.tscn`).
- **Nodos dentro de una escena:** `PascalCase`, nombrados según lo que representan (no nombres por defecto tipo `Sprite2D2`).
- **Assets:** `snake_case` descriptivo (`paddle_hit.wav`, `ball_bounce_wall.wav`, `paddle_idle.png`), para que sean identificables sin abrir el archivo.
### 8.4 Ramas
 
Proyecto en solitario: commits directos a `main` son aceptables. Si se quiere practicar ramas cortas por feature, convención `feat/nombre-corto` (ej. `feat/ball-movement`, `feat/cpu-ai`), fusionadas a `main` al terminar la feature.
 
## 9. Registro de decisiones
 
Cuando un número o una decisión de diseño cambie respecto a lo definido en las secciones anteriores durante la implementación (un valor de `GameConfig` que se ajusta por sensación de juego, un estado de la state machine que se fusiona con otro, un patrón que se descarta), se anota aquí con fecha y motivo breve. Sirve como memoria de "qué se decidió y por qué", útil tanto para retomar el proyecto más adelante como para no repetir los mismos errores en el proyecto grande.
 
Formato:
 
```
- AAAA-MM-DD: <qué cambió> — <por qué>
```
 
Ejemplos (borrar cuando haya entradas reales):
- 2026-XX-XX: GOAL_SCORED se fusiona con READY, no hacía falta como estado separado.
- 2026-XX-XX: velocidad inicial bajada de 250 a 180px/s, se sentía injugable para el jugador humano.
---
*(Sin entradas todavía)*

## 10. Orden de implementación
 
Cada paso debe dejar algo ejecutable y verificable en el editor antes de pasar al siguiente, evitando construir piezas que dependan de algo que aún no se puede probar.
 
- [ ] 1. `GameConfig.tres` + `GameConfig.gd` — fijar los números (velocidad inicial, incremento, tope, puntos para ganar), aunque sean provisionales.
- [ ] 2. `Ball` moviéndose y rebotando en paredes (sin palas todavía) — movimiento manual, jitter, límites hardcodeados.
- [ ] 3. `Paddle` con input y clamp — solo movimiento de un jugador, sin CPU aún.
- [ ] 4. Rebote `Ball` ↔ `Paddle` con ángulo según punto de impacto — primer core jugable (un jugador contra pared).
- [ ] 5. `Goal` + señal por `EventBus` + marcador básico — primer punto real anotado.
- [ ] 6. `GameStateMachine` — envolver lo anterior en estados (`READY` / `PLAYING` / `GOAL_SCORED`) en vez de lógica "siempre activa".
- [ ] 7. CPU para la segunda pala.
- [ ] 8. Menú + `GameManager` decidiendo modo (CPU / 2 jugadores).
- [ ] 9. Pausa, pantalla de fin de partida, sonidos, arte final.
Nota: este orden es una guía, no un contrato rígido — si al implementar surge una razón de peso para alterarlo, se anota en la sección 9 (Registro de decisiones).