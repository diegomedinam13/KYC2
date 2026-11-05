# Reporte de consumo de tokens y costos — Proyecto KYC (PROJECT V1)

Corte: 2026-07-13. Fuente: transcripts locales de Claude Code de este proyecto
(23 sesiones, incluye subagentes). Deduplicado por request; los tokens son de
API reales. Los costos son **aproximados**, calculados a precios de lista del
API de Anthropic (jul 2026); si la cuenta opera con suscripcion (Max/Team), el
costo marginal real lo cubre la suscripcion y esto representa el equivalente API.

## Totales

| Concepto | Tokens | Costo aprox |
|---|---|---|
| Subida nueva (input directo) | 23 121 | $0.23 |
| Escritura de cache (1.25x input) | 4 659 379 | $43.47 |
| Lectura de cache (0.1x input) | 77 867 441 | $69.64 |
| **Bajada** (output generado) | **456 363** | **$19.96** |
| **Total** (650 requests) | — | **≈ $133** |

La lectura de caché es el rubro mayor: 78M de tokens releídos, pero a ~10% de
la tarifa — sin caché ese mismo tráfico habría costado ~$700 solo de input.
Nota: si las escrituras usaron TTL de 1 hora (2x en vez de 1.25x), el total
sube a ≈ $153.

## Uso y costo por modelo

Precios por millón de tokens: Fable 5 $10 in / $50 out · Opus 4.7 y 4.8 $5/$25 · Sonnet 5 $2/$10 (precio introductorio vigente hasta 2026-08-31; lista: $3/$15).

| Modelo | Requests | Subida nueva* | Bajada | Cache releido | Costo aprox |
|---|---|---|---|---|---|
| Fable 5 (sesion principal) | 278 | 2 629 982 | 347 335 | 64 397 940 | **$114.58** |
| Opus 4.7 (subagentes) | 105 | 1 018 534 | 95 686 | 4 181 271 | $10.85 |
| Opus 4.8 (subagentes) | 119 | 515 467 | 4 114 | 4 322 791 | $5.49 |
| Sonnet 5 (subagentes) | 148 | 518 517 | 9 228 | 4 965 439 | $2.38 |
| **Total** | **650** | **4 682 500** | **456 363** | **77 867 441** | **≈ $133.30** |

\* input directo + escritura de cache. Con Sonnet 5 a precio de lista el total
sube ~$1.20. Los subagentes despachados como "opus" resolvieron a 4.7 o 4.8
segun disponibilidad; se reportan como los registro la API.

## Distribucion y lecturas

- **Fable 5 concentra el 86% del costo** ($114.58): es la sesion orquestadora —
  cada turno re-lee todo el contexto acumulado (64M de cache releido) y genero
  el 76% del output (planes, orquestacion, los fixes del escaneo).
- Los 3 modelos de subagentes juntos costaron ~$18.72: la estrategia de
  despachar implementacion/reviews a Opus/Sonnet en vez de a Fable fue
  efectiva en costo.
- La sesion de hoy (plan + 11 tasks subagent-driven + escaneo v2) concentro
  ~75% del consumo total del proyecto; las 22 sesiones previas (investigacion,
  Obsidian, spikes) el resto.
- Costo por entregable: el prototipo completo (backend + frontend + escaneo +
  docs, ~20 commits) salio en ≈ $133 de computo equivalente API.

## Auditoria de contexto y optimizacion (2026-07-13)

Auditoria de que carga/lee una sesion de Claude Code en este repo y donde se va
el gasto real.

### Donde NO esta el gasto

Los `.md` de arranque son baratos: CLAUDE.md (~1.0k tokens, siempre cargado) +
HANDOFF.md (~1.3k) + progress.md (~0.7k) + bitacora Obsidian con addendums
(~2.5k) ≈ **5.5k tokens por sesion** — menos del 0.5% de una sesion larga.
Optimizarlos ahorra poco por si mismo; el valor esta en que no manden a leer
cosas caras.

### Donde SI esta el gasto (en orden)

1. **Sesiones-maraton**: la sesion del 2026-07-13 concentro ~75% del gasto
   total del proyecto. Cada turno relee el contexto acumulado (64M de cache
   releido en Fable 5); una sesion que acumula 150k tokens de contexto y da 100
   turnos relee 15M aunque sea a tarifa de cache.
2. **Artefactos gordos leidos al contexto principal**: review packages de
   75–96 KB, diffs de 118–138 KB en `.superpowers/sdd/`, y el plan de 72 KB en
   `docs/superpowers/plans/`. Un solo read de esos = 18–35k tokens que luego se
   releen cada turno restante de la sesion.
3. **Riesgo latente**: `frontend/public/opencv.js` pesa **9 MB (~2.2M tokens)**
   — un Read o un grep con contenido sobre el revienta el contexto de golpe.
   Igual los `package-lock.json` (250 KB + 108 KB).

### Cambios aplicados en esta auditoria

1. **CLAUDE.md**: `HANDOFF.md` pasa a ser el unico documento de arranque; la
   bitacora Obsidian queda como consulta puntual + registro de sesiones (antes
   era lectura obligatoria: ~2.5k tokens menos por sesion y menos indireccion).
2. **CLAUDE.md § Economia de contexto**: reglas permanentes anti-quema
   (opencv.js/package-locks nunca al contexto, sdd/ solo progress.md, plan por
   secciones, trabajo pesado a subagentes, /clear entre tareas).
3. **Arnes headless al repo**: `frontend/scripts/worker-harness-test.cjs`
   (verificado 12/12 desde su nueva ubicacion; 27/27 tests de frontend sin
   afectarse). Antes vivia en scratchpad volatil — recrearlo costaba 10–20k
   tokens cada vez que se borrara.

### Recomendaciones pendientes (decision de Mario)

- **Purgar `.superpowers/sdd/`** salvo `progress.md`: ~300 KB de briefs,
  reports, review-packages y diffs ya consumidos (gitignored, sin uso futuro):
  `find .superpowers/sdd -type f ! -name progress.md ! -name .gitignore -delete`
- **Habitos que ya demostraron ahorro** (sostenerlos): despachar a subagentes
  Opus/Sonnet — hicieron 372 de 650 requests por el 14% del costo; sesion nueva
  o `/clear` por tarea no relacionada, en vez de estirar una sesion-maraton.
- Si el costo de Fable preocupa, orquestar sesiones normales con Opus 4.8
  ($5/$25 vs $10/$50) y reservar Fable para planeacion/revision final.
