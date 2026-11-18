# Plan de trabajo · Proyecto final MGP

**Sistema multi-agente de análisis de riesgo de crédito al consumo**

Entrega: **domingo 29 de noviembre** (repositorio + landing page + informe) · Defensa: **1-2 de diciembre**
Fecha de este plan: 18 de noviembre · **Quedan 11 días**

---

## 1. Punto de partida: qué tenemos de verdad

Inventario real del repositorio a 18 de noviembre, comprobado ejecutando el código.

### Lo que funciona y se reutiliza tal cual

| Pieza | Estado | Valor para el proyecto final |
|---|---|---|
| `tool_use/credit_math.py` | Cuota francesa, TAE por bisección, coste total, ratio. Determinista y con tests | **Columna vertebral.** Es la verdad terreno numérica y las herramientas del agente de cálculo |
| `tool_use/tools.py` | `get_euribor` (API del BCE), `loan_metrics`, `save_credit_report` | Herramientas ya validadas con JSON Schema desde Pydantic |
| `tool_use/executor.py` | Bucle de 5 pasos propio, multi-step y paralelo | Base del agente de cálculo |
| `rag/` | Corpus Ley 16/2011 + Directiva 2023/2225, chunking semántico, BM25 + denso + híbrido calibrado (λ=0.6) | Agente normativo casi hecho. Recall@5 0.74 / MRR 0.63 |
| `agent/react_agent.py` | Bucle ReAct propio, 597 líneas, robusto a errores de formato | **Es el sistema B de la comparación**, gratis |
| `api/` | FastAPI con 4 endpoints, contratos en `schemas.py`, Docker | Donde enchufamos el endpoint nuevo |
| Tests | **256 pasan en verde** | Red de seguridad para refactorizar rápido |
| `EXPERIMENTS.md` | Cuaderno honesto, con limitaciones admitidas | Modelo de tono para el informe |

### Lo que falta o está roto (hay que resolverlo el día 1)

1. **Los pesos entrenados no están en el repositorio.** `rlm/weights/` solo tiene `smoke_lora`. `sft_lora` y `grpo_lora` están en el `.gitignore`, así que el cerebro `rlm` **no se puede ejecutar**. Hay que pedírselos al tercer miembro del equipo o reentrenar (≈3 h de GPU). Sin ellos cae la ablación del agente de cálculo pequeño.
2. **El corpus está vacío** (`rag/corpus/` solo tiene el README). Se regenera con `uv run python -m rag.fetch_corpus`, es reproducible, pero hay que hacerlo.
3. **El entorno estaba desactualizado.** Hace falta `uv sync --extra train --extra rag --extra agent`, no solo `--extra train`.
4. **Choque de puertos.** El servicio central de inferencia escucha en `127.0.0.1:8000` y nuestra API usa el 8000 por defecto. **Mover la API al 8001.**
5. **El servicio de 27B no responde en esta sesión** (`connection refused` en `127.0.0.1:8000/v1/models`). La GPU visible es una H200 de 143 GB prácticamente libre. Hay que confirmar con el equipo docente si el servicio hay que levantarlo, si vive en otro nodo, o si tenemos que servirnos nosotros el modelo.
6. **Todo el código actual carga modelos en proceso con transformers.** Para usar el 27B hay que escribir un cliente contra la API tipo OpenAI. Es la primera tarea de código del proyecto.

---

## 2. Estrategia: dónde están los puntos

El enunciado es explícito: *"Es más importante la calidad del análisis que el rendimiento final del sistema"*. Lo que se valora:

> claridad del problema · diseño razonado · **comparación entre alternativas** · **análisis de trade-offs** · **ablaciones** · métricas adecuadas

Cuatro de esos seis criterios son de **evaluación**, no de implementación. De ahí las tres decisiones que gobiernan todo el plan:

**Decisión 1 — El sistema se construye sobre el 27B vía API, no entrenando nada.**
Entrenar consume días de GPU y no puntúa. El 27B es gratis, es mucho mejor que el 1.7B de la práctica y permite concurrencia real. El trabajo de fase 1 (SFT+GRPO) **no se tira**: se convierte en la ablación más interesante que tenemos (¿compensa un 0.6B afinado como agente de cálculo frente al 27B?), y su respuesta previsiblemente negativa es un buen resultado, no un fracaso.

**Decisión 2 — La verdad terreno se construye antes que el sistema.**
Es el consejo del propio enunciado de ARCA: *"empezad por el verificador y el conjunto dorado, no por el modelo"*. Sin benchmark no hay comparación, sin comparación no hay nota. Los días 1-3 son de datos.

**Decisión 3 — Se recorta funcionalidad antes que evaluación.**
Si vamos mal de tiempo, cae un agente o caen las ablaciones opcionales, nunca la comparación A/B/C ni el informe.

### Recortes de alcance ya decididos

- **Fuera lo multimodal** (la nómina escaneada de la propuesta). Elegimos la opción "sistema multi-agente", así que no es obligatorio, y abre un frente de incertidumbre (no sabemos si el 27B acepta imágenes) por un retorno bajo. Va al informe como trabajo futuro.
- **Fuera reentrenar nada nuevo.** Solo se reutiliza el adaptador GRPO existente si aparece.
- **El gold set normativo no se reescribe entero**, se amplía con 15 preguntas difíciles escritas a mano sobre las 50 que ya hay.

---

## 3. Arquitectura objetivo

```
                        POST /credit-report
                                │
                    ┌───────────▼────────────┐
                    │   ORQUESTADOR (27B)    │
                    │   plan + síntesis      │
                    └───────────┬────────────┘
                                │  fan-out asyncio (paralelo real)
            ┌───────────────────┼───────────────────┐
            ▼                   ▼                   ▼
    ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
    │ AG. CÁLCULO   │   │ AG. NORMATIVO │   │ AG. SOLVENCIA │
    │ credit_math   │   │ RAG híbrido   │   │ política de   │
    │ herramientas  │   │ λ=0.6 + citas │   │ riesgo + perfil│
    │ deterministas │   │               │   │               │
    └───────┬───────┘   └───────┬───────┘   └───────┬───────┘
            └───────────────────┼───────────────────┘
                                ▼
                    ┌────────────────────────┐
                    │   AGENTE CRÍTICO       │
                    │ · recalcula las cifras │
                    │ · comprueba que cada   │
                    │   afirmación legal     │
                    │   tenga fragmento      │
                    │ · devuelve o aprueba   │──┐ máx. 2 rondas
                    └───────────┬────────────┘  │
                                │◄──────────────┘
                                ▼
                       INFORME DE RIESGO
                 decisión · cálculos · citas · traza
```

### Reglas de diseño que no se negocian

- **Nada de aritmética mental.** El agente de cálculo solo reporta números que salen de `credit_math.py`. El crítico los recalcula de forma independiente.
- **Salida estructurada con Pydantic** en cada frontera entre agentes, usando `response_format` de vLLM. El profesor dio un tutorial de esto (`docs/pydantic_structured_output.ipynb`): se espera que lo usemos.
- **Paralelismo de verdad** con `asyncio` + `httpx`. Es lo que justifica que sea multi-agente y no una tubería: lo medimos en latencia y lo contamos en el informe.
- **Todo se instrumenta**: tokens de entrada y salida, latencia y número de llamadas por agente y por petición. Sin esto no hay eje de coste.
- **El crítico puede devolver el trabajo**, con un máximo de 2 rondas para que el coste no se dispare. El número de rondas es un parámetro, y su efecto es una de las ablaciones.

### La política de riesgo (la clave del diseño)

`multiagent/policy.py` implementa una política determinista que, dada una solicitud, devuelve `aprobar` / `contraoferta` / `denegar`. Es a la vez:

- **la verdad terreno** de la etiqueta de decisión (gratis, para las 200 solicitudes), y
- **un documento del corpus**: la política escrita en prosa entra en RAG junto a la ley.

Ese segundo punto es la jugada importante. Obliga al sistema a *recuperar* las reglas para poder aplicarlas, en vez de llevarlas en el prompt. Así la ablación "sin RAG" degrada la decisión de verdad, y no solo las citas.

Esqueleto de la política (ajustad los umbrales al escribirla):

| Condición | Decisión |
|---|---|
| DTI = (cuota + otras deudas) / ingresos > 40 % | denegar |
| TAE > 2× el tipo medio de referencia (usura) | denegar |
| Contrato temporal con antigüedad < 6 meses y DTI > 30 % | denegar |
| 35 % < DTI ≤ 40 % | contraoferta (alargar plazo o bajar importe hasta DTI ≤ 35 %) |
| Resto | aprobar |

### Módulos nuevos

```
multiagent/
  llm.py            cliente asíncrono OpenAI-compatible + instrumentación
  schemas.py        Pydantic: Solicitud, ResultadoCalculo, ResultadoNormativo,
                    ResultadoSolvencia, VeredictoCritico, InformeRiesgo
  policy.py         política de riesgo determinista (verdad terreno)
  agents/{calc,norm,solvency,critic}.py
  orchestrator.py   fan-out, bucle de crítica, síntesis
  baselines.py      sistema A (LLM pelado) y adaptador del sistema B
bench/
  generate.py       genera las 200 solicitudes con verdad terreno
  cases.jsonl       dev (50) / test (150)
  gold_norm.jsonl   50 preguntas existentes + 15 nuevas difíciles
eval/
  run.py            lanza A/B/C y ablaciones con concurrencia
  metrics.py        exactitud numérica, accuracy/F1, citas, coste, bootstrap
  judge.py          juez de fidelidad (27B)
web/                landing page estática
```

---

## 4. Plan de evaluación

Es la parte que más nota da. Se diseña ahora y no se toca después.

### Sistemas comparados

| | Sistema | Qué aísla |
|---|---|---|
| **A** | Una sola llamada al 27B, sin herramientas ni RAG | El suelo: cuánto de esto sabe hacer un LLM solo |
| **B** | Agente único ReAct (el de la fase 4) con herramientas + RAG, cerebro 27B | ¿Aporta algo descomponer en varios agentes, o basta un bucle? |
| **C** | Sistema multi-agente con orquestador y crítico | Nuestra propuesta |

El sistema B es prácticamente gratis: `agent/react_agent.py` ya existe y solo necesita el cerebro nuevo. Es la comparación más exigente y la más honesta.

### Ablaciones sobre C

1. **sin crítico** — ¿cuánto corrige de verdad y a qué coste en tokens?
2. **sin RAG** (política y ley en el prompt) — ¿recuperar aporta sobre meter el texto en contexto?
3. **sin herramientas de cálculo** (el agente de cálculo razona a mano) — debería hundirse; confirma que las herramientas son la pieza crítica
4. **secuencial frente a paralelo** — mide solo latencia, misma calidad; cuantifica el beneficio de la arquitectura
5. **agente de cálculo pequeño** (0.6B + GRPO de la fase 1) frente al 27B — el trade-off coste/calidad, y el cierre del arco de la práctica

Las ablaciones 1-3 y 5 corren sobre un subconjunto de 100 casos para que quepan en el presupuesto de GPU; se dice en el informe.

### Métricas

| Eje | Métrica |
|---|---|
| Numérico | Exactitud a 2 decimales por campo (cuota, TAE, coste total, DTI) y exactitud conjunta |
| Decisión | Accuracy y F1 macro sobre 3 clases |
| Citas | Precisión y recall frente al pasaje dorado; porcentaje de citas inexistentes (ya implementado) |
| Fidelidad | Juez LLM: ¿cada afirmación normativa está respaldada por el fragmento citado? |
| Coste | Tokens de entrada/salida, latencia p50 y p95, número de llamadas al LLM por informe |

**Rigor estadístico** (esto distingue un trabajo correcto de uno excelente):

- **Partición dev/test desde el principio.** 50 casos para iterar prompts, 150 intocables hasta la ejecución final. En `EXPERIMENTS.md` ya señalamos dos veces que métricas de desarrollo nos salieron optimistas; esta vez lo evitamos por diseño y lo contamos.
- **Intervalos de confianza por bootstrap** sobre las diferencias entre sistemas. Es barato y evita afirmar mejoras que caben en el ruido.
- **El juez es el mismo modelo que genera.** Hay sesgo de autoevaluación: hay que decirlo en el informe y validar el juez contra 20 casos anotados a mano.

### Presupuesto de cómputo (importante)

3 sistemas × 150 casos + 4 ablaciones × 100 casos ≈ **850 ejecuciones**. El sistema C hace del orden de 10-15 llamadas al LLM por caso. Con concurrencia 8-16 contra vLLM es una ejecución nocturna, no una tarde. Por eso:

- Se itera siempre sobre los **50 de dev**, nunca sobre el test.
- La ejecución final se lanza **la noche del 25 al 26**, con margen para repetirla una vez.
- `eval/run.py` guarda resultados parciales por caso y es **reanudable**. Si se corta la sesión de la DGX, no se pierde nada.

---

## 5. Calendario

| Día | Fecha | Hito | Entregable comprobable |
|---|---|---|---|
| 1 | mié 18 | **Desbloqueo** | Servicio 27B confirmado, pesos localizados, puerto movido, esquemas Pydantic congelados |
| 2 | jue 19 | **Verdad terreno I** | `policy.py` + `bench/generate.py` funcionando, 200 casos generados |
| 3 | vie 20 | **Verdad terreno II + cliente** | Gold normativo ampliado, política en el corpus, `llm.py` con instrumentación |
| 4 | sáb 21 | **Agentes** | Cálculo, normativo y solvencia responden con Pydantic sobre casos de dev |
| 5 | dom 22 | **Orquestador** | Fan-out paralelo + síntesis; un informe completo de punta a punta |
| 6 | lun 23 | **Crítico + arnés** | Bucle de crítica cerrado; `eval/run.py` y `metrics.py` corriendo sobre dev |
| 7 | mar 24 | **Sistemas A y B + endpoint** | Las tres alternativas corren con el mismo arnés; `/credit-report` en la API |
| 8 | mié 25 | **CONGELACIÓN DE CÓDIGO** · ejecución final | Nada de funcionalidad nueva. Se lanza A/B/C sobre los 150 de test por la noche |
| 9 | jue 26 | **Ablaciones + landing** | Las 5 ablaciones lanzadas; landing page en pie con los primeros números |
| 10 | vie 27 | **Informe** | Las 3 páginas escritas con las tablas definitivas |
| 11 | sáb 28 | **Pulido y colchón** | README de portfolio, Docker limpio, tests en verde, ensayo de la demo |
| — | dom 29 | **ENTREGA** | Repositorio + landing + informe por Moodle |
| — | lun 30 | Diapositivas | 20 minutos montados y cronometrados |
| — | 1-2 dic | Defensa | — |

**El 25 es la fecha que gobierna el plan.** Si el 25 por la noche no se lanza la evaluación final, no hay informe el 27. Todo lo que amenace esa fecha se recorta.

**Puertas de decisión:**
- **Final del día 3**: si el 27B no está accesible, se decide ya servir nosotros un modelo en la H200 (hay 143 GB) y se ajusta el plan. No se deja para el día 5.
- **Final del día 6**: si el crítico no funciona, se entrega C sin crítico y la ablación 1 se convierte en "no lo conseguimos y por qué" — que también puntúa.

---

## 6. Reparto entre Jaime e Ignacio

El reparto se apoya en **congelar los contratos el día 1**. Si `multiagent/schemas.py` y la firma de `eval/run.py` están fijados desde el principio, cada uno trabaja contra un doble de prueba sin esperar al otro. Es lo que ya hicimos en la fase 4 con el cerebro simulado.

Jaime lleva la **vía ligera** (datos, métricas y presentación) e Ignacio la **vía pesada** (sistema y evaluación). El reparto no es mitad y mitad: está dicho abajo qué consecuencia tiene y cómo se compensa.

### Jaime — vía *datos, métricas y presentación*

Trabajo acotado, determinista y **sin dependencia de la GPU**: cada pieza se termina y se cierra. Unos 5 días de trabajo real repartidos en 11.

| Día | Tarea | Depende de |
|---|---|---|
| 1 | *(conjunto)* Congelar `schemas.py` y los umbrales de la política. Localizar los pesos `sft_lora`/`grpo_lora` y regenerar el corpus | — |
| 2 | `policy.py`: política de riesgo determinista + tests de casos límite | — |
| 3 | `bench/generate.py` → 200 solicitudes con verdad terreno, partición dev 50 / test 150. Redactar la política como documento en prosa para el corpus | día 2 |
| 4 | `metrics.py`: exactitud numérica, accuracy y F1 macro, precisión/recall de citas, bootstrap | día 3 |
| 5-6 | **Colchón.** Si Ignacio va justo, aquí se recoge el agente de cálculo o el de solvencia: son envoltorios finos sobre `credit_math.py` y `policy.py`, que ya conoces | — |
| 9 | **Landing page**: problema, arquitectura, ejemplos de funcionamiento, resultados | números del día 8 |
| 10 | Revisión cruzada del informe | borrador de Ignacio |
| 11 | README de portfolio, ensayo de la demo | — |
| 30 nov | Diapositivas | informe cerrado |

### Ignacio — vía *sistema y evaluación*

| Día | Tarea | Depende de |
|---|---|---|
| 1 | *(conjunto)* Contratos. Cliente `llm.py`: asíncrono, reintentos, instrumentación de tokens y latencia. Mover la API al 8001 | — |
| 3 | `llm.py` con salida estructurada Pydantic y caché en disco | — |
| 4 | Agente de cálculo y agente normativo | `schemas.py` |
| 5 | Orquestador: fan-out con `asyncio.gather` y síntesis | día 4 |
| 6 | Agente crítico, bucle de devolución, agente de solvencia | `policy.py` (Jaime, día 2) |
| 7 | Endpoint `/credit-report`, sistemas A y B conectados al arnés | días 5-6 |
| 6-7 | `eval/run.py` (concurrente y reanudable) y `judge.py` | `metrics.py` (Jaime, día 4) |
| 8-9 | Lanzar la ejecución final y las ablaciones; tablas y figuras | código congelado |
| 10 | **Escribir el informe** | día 9 |

### Consecuencia del reparto, dicha claramente

Con este reparto **Ignacio queda en el camino crítico de los días 4 a 9 prácticamente solo**, y ahí es donde se puede romper el calendario. En 11 días eso solo se sostiene si pasa una de estas dos cosas:

1. **El tercer miembro absorbe trabajo real**, no accesorio. Lo más separable y lo que mejor encaja: las **15 preguntas normativas difíciles** escritas a mano, la **validación del juez** contra 20 casos anotados, el **análisis de fallos** y las **entradas de `EXPERIMENTS.md`**. Son unos 3 días de trabajo que no bloquean a nadie.
2. **Jaime recoge un agente en los días 5-6**, que es el hueco natural de su vía. El de cálculo y el de solvencia son los candidatos: el primero envuelve `credit_math.py` y el segundo `policy.py`, los dos ficheros que habrá escrito él mismo.

Si no pasa ninguna de las dos, el recorte que toca es la **ablación 4** (secuencial frente a paralelo) y reducir las ablaciones a 3 sobre 80 casos. Nunca la comparación A/B/C.

### Puntos de sincronización

| Cuándo | Qué se acuerda |
|---|---|
| **Día 1, al empezar** | `schemas.py` y los umbrales de la política. **Bloqueante: no se escribe código antes** |
| Día 3, por la tarde | Puerta de decisión del 27B |
| Día 4, final | Jaime entrega `policy.py` y `metrics.py`; Ignacio los necesita el día 6 |
| Día 5, final | Primer informe de punta a punta sobre un caso de dev; validación conjunta |
| Día 7, final | **Congelación.** A partir de aquí solo correcciones |
| Día 10 | Revisión cruzada del informe |

### Nota sobre el tercer miembro

El informe de ARCA (`report/Final_Report_ARCA_Guzman_Jaime_Ignacio.pdf`) indica un equipo de tres. Hay que cerrar el día 1 qué asume, y con este reparto **ya no es opcional**: es lo que mantiene a Ignacio dentro del calendario.

---

## 6 bis. ¿Es esto paralelo de verdad?

En parte sí y en parte no, y conviene tenerlo claro porque es donde se rompen los calendarios.

### Las dos vías: paralelas hasta el día 7, embudo a partir del 8

```
 Día   1   2   3   4   5   6   7  │  8   9  10  11
      ─────────────────────────────┼────────────────
Jaime  ■   ■   ■   ■   ·   ·   ·  │  ·   ■   ■   ■
Igna.  ■   ■   ■   ■   ■   ■   ■  │  ■   ■   ■   ·
                                  │
       ── PARALELO ───────────────┤── SECUENCIAL ──
                                  │
                        congelación (25 nov)
```

**Días 1-7: paralelo real.** Las dos vías tocan ficheros distintos y ninguna necesita que la otra haya terminado. La condición es la congelación de contratos del día 1: con `schemas.py` fijado, Ignacio escribe los agentes contra datos de ejemplo y Jaime escribe las métricas contra informes de ejemplo, sin que ninguno espere.

**Días 8-11: secuencial, y no hay forma de evitarlo.** Hay una cadena que no se puede comprimir:

```
código congelado → ejecución final → tablas → informe → diapositivas
```

Cada eslabón necesita el anterior terminado. Por eso la congelación del día 25 no es una sugerencia: es lo que da tiempo a que el embudo quepa.

### Las tres dependencias reales entre vosotros

Solo hay tres sitios donde uno bloquea al otro. Fuera de ellos sois independientes:

1. **`schemas.py` (día 1)** bloquea a los dos. Es la razón de que sea una reunión conjunta y lo primero del plan.
2. **`policy.py` (Jaime, día 2) → agente de solvencia (Ignacio, día 6).** Holgura de 4 días, cómoda.
3. **`metrics.py` (Jaime, día 4) → `eval/run.py` (Ignacio, día 6).** Holgura de 2 días, ajustada: es la entrega que Jaime no puede retrasar.

Y una regla que las resume: **construir en paralelo, ejecutar en serie**. `metrics.py` se puede escribir y testear entero con informes falsos antes de que exista un solo informe de verdad. Solo *correrlo* sobre resultados reales depende del sistema.

### Y las fases de ARCA, ¿son paralelas?

Dos respuestas, según a qué te refieras:

- **Como trabajo nuestro: ya están hechas.** Las cuatro fases de la práctica (razonamiento, herramientas, RAG, agente) son el material de partida, no tareas pendientes. En el proyecto final no se rehacen: se reutilizan como piezas.
- **Como ejecución del sistema: los tres subagentes sí corren en paralelo, el crítico no.** El orquestador lanza cálculo, normativo y solvencia a la vez con `asyncio.gather`, y el crítico solo puede empezar cuando los tres han terminado, porque su trabajo es revisarlos. Esa forma (abanico paralelo seguido de un cuello secuencial) es justo lo que mide la **ablación 4**: cuánta latencia ahorra el abanico y cuánta se la come el crítico.

---

## 7. Protocolo de trabajo

- **Ramas**: `jaime-final-project` (integración) + `ignacio-final-project`. Merge a integración al menos cada dos días; nunca un merge grande el día 8.
- **Tests obligatorios** en `policy.py`, `metrics.py` y los esquemas. Son el suelo sobre el que se apoya todo lo demás; un error ahí invalida cada número del informe.
- **`EXPERIMENTS.md` se escribe el mismo día del experimento**, no al final. Las entradas fechadas son criterio de corrección explícito, y las que dicen "probamos esto y no funcionó" valen más que las que dicen "todo bien".
- **`git push` al terminar cada sesión.** La DGX no tiene copias de seguridad.
- **Sin secretos en el repositorio.** El `.env` está en `.gitignore`; hay que mantenerlo así.

---

## 8. Riesgos

| Riesgo | Probabilidad | Mitigación |
|---|---|---|
| El 27B no está accesible | Media — hoy da `connection refused` | Puerta de decisión el día 3: servir un modelo propio en la H200 (143 GB libres) |
| Los pesos de fase 1 no aparecen | Media | Cae solo la ablación 5. Se dice en el informe. No reentrenar salvo que sobre tiempo |
| La evaluación no cabe en el tiempo de GPU | Media | Arnés reanudable, concurrencia, ablaciones sobre 100 casos, ejecución nocturna con un día de margen |
| El crítico dispara el coste sin mejorar nada | Media | Es un resultado publicable, no un fallo. Máximo 2 rondas |
| Sesiones de 24 h de la DGX que se cortan | Alta | Resultados parciales en disco, checkpoints, push diario |
| Informe apretado en 3 páginas | Alta | Las tablas y figuras cargan el peso; el detalle vive en `EXPERIMENTS.md` y se referencia |
