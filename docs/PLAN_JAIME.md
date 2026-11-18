# Plan de Jaime · vía datos, métricas y presentación

> Plan maestro: [`PLAN_PROYECTO_FINAL.md`](PLAN_PROYECTO_FINAL.md) · Parte común: [`PLAN_SECUENCIAL.md`](PLAN_SECUENCIAL.md) · La otra vía: [`PLAN_IGNACIO.md`](PLAN_IGNACIO.md)

**Carga:** ~5 días de trabajo real repartidos en 11.
**Característica de esta vía:** nada de lo que hay aquí necesita la GPU ni que el sistema exista. Puedes terminarlo todo aunque el 27B no aparezca nunca.

## Lo que recibes y lo que entregas

| | Qué | Cuándo | De/para quién |
|---|---|---|---|
| Recibes | `multiagent/schemas.py` congelado | día 1 | reunión conjunta |
| **Entregas** | **`multiagent/policy.py`** | **final del día 2** | Ignacio lo necesita el día 6 |
| **Entregas** | **`eval/metrics.py`** | **final del día 4** | Ignacio lo necesita el día 6 — *es tu entrega crítica* |
| Entregas | `bench/cases.jsonl` (200 casos) | final del día 3 | la evaluación entera |
| Entregas | La política en prosa en `rag/corpus/` | final del día 3 | el agente normativo |
| Recibes | Resultados de la evaluación | día 8 | Ignacio |

---

## Día 1 (mié 18) · Desbloqueo

- [ ] **Reunión conjunta** → ver [`PLAN_SECUENCIAL.md`](PLAN_SECUENCIAL.md), sección "Día 1". No empieces nada antes.
- [ ] Pedir a Guzmán los pesos `rlm/weights/sft_lora` y `rlm/weights/grpo_lora`. No están en git (`.gitignore`). Sin ellos cae la ablación 5.
- [ ] Regenerar el corpus:
      `uv run python -m rag.fetch_corpus`
- [ ] Confirmar que el entorno está al día:
      `uv sync --extra train --extra rag --extra agent && uv run pytest -q`
      (deben pasar los 256 tests)
- [ ] Diseñar con Ignacio los umbrales de la política (DTI, usura, antigüedad). **Se cierran hoy y no se tocan**: son la verdad terreno, cambiarlos luego invalida todos los números.

## Día 2 (jue 19) · `multiagent/policy.py`

La política determinista que decide `aprobar` / `contraoferta` / `denegar`. Es la verdad terreno de la etiqueta.

- [ ] Función `evaluar(solicitud) -> DecisionPolitica` con la decisión **y los motivos** (lista de reglas disparadas). Los motivos los usará el informe y el agente crítico.
- [ ] Se apoya en `tool_use/credit_math.py` para cuota, TAE, coste total y DTI. **No reimplementes la aritmética**, ya está testeada.
- [ ] Para la contraoferta, devolver también la condición corregida: plazo o importe que deja el DTI ≤ 35 %.
- [ ] `tests/test_policy.py` con casos límite: DTI justo en 35.00 y 40.00, TAE justo en el umbral de usura, ingresos cero, plazo 1 mes, contrato temporal con y sin antigüedad.

**Hecho cuando:** `uv run pytest tests/test_policy.py -q` en verde y la función es pura y determinista (misma entrada → misma salida, sin red ni modelo).

## Día 3 (vie 20) · Benchmark y política en el corpus

### `bench/generate.py`

- [ ] Generador programático de **200 solicitudes** con semilla fija. Campos: ingresos netos, otras deudas, importe, plazo, tipo nominal, comisión de apertura, tipo de contrato, antigüedad laboral.
- [ ] Verdad terreno por caso: los cuatro números (`credit_math`) y la decisión con sus motivos (`policy`).
- [ ] **Distribución equilibrada**: que las tres clases estén representadas. Si el 90 % sale "aprobar", el F1 macro no dice nada. Ajusta los rangos de muestreo hasta tener al menos un 25 % de cada clase.
- [ ] Incluir **casos frontera a propósito**: DTI de 34.9 y 35.1, TAE rozando la usura. Son los que separan un sistema bueno de uno que acierta por inercia.
- [ ] **Partición dev 50 / test 150**, en ficheros distintos. El test **no se mira hasta el día 8**.

### La política como documento del corpus

- [ ] Escribir `rag/corpus/politica_riesgo.md`: las mismas reglas, en prosa de manual interno, con la estructura de un documento real (apartados numerados).
- [ ] Comprobar que entra en la ingesta y se trocea bien:
      `uv run python -m rag.evaluate --gold rag/gold/gold.jsonl --k 5`
- [ ] **Por qué importa:** al estar la política en RAG y no en el prompt, el sistema tiene que recuperarla para aplicarla. Eso es lo que hace que la ablación "sin RAG" degrade la decisión y no solo las citas. Es la mejor idea de diseño que tenemos: no te la saltes metiendo las reglas en el prompt.

**Hecho cuando:** `bench/cases_dev.jsonl` (50) y `bench/cases_test.jsonl` (150) existen, con las tres clases representadas, y `politica_riesgo.md` aparece en los fragmentos recuperados para una consulta tipo "¿qué ratio de endeudamiento máximo se admite?".

## Día 4 (sáb 21) · `eval/metrics.py` — *entrega crítica*

Ignacio la necesita el día 6 y solo hay 2 días de holgura. **Esta es la fecha que no puedes mover.**

- [ ] Exactitud numérica a 2 decimales por campo (cuota, TAE, coste total, DTI) y exactitud conjunta.
- [ ] Accuracy y **F1 macro** sobre las 3 clases + matriz de confusión.
- [ ] Precisión y recall de citas frente al pasaje dorado; porcentaje de citas inexistentes (reutiliza la comprobación que ya hay en `rag/generate.py`).
- [ ] Agregados de coste: tokens de entrada/salida, latencia p50 y p95, llamadas al LLM por informe.
- [ ] **Bootstrap** de las diferencias entre sistemas: intervalo al 95 % sobre la diferencia A−B, B−C, etc. Es barato y evita afirmar mejoras que caben en el ruido.
- [ ] `tests/test_metrics.py` con informes **falsos** construidos a mano. No necesitas que el sistema exista: ese es el truco de "construir en paralelo, ejecutar en serie".

**Hecho cuando:** `uv run pytest tests/test_metrics.py -q` en verde y le puedes pasar un JSON de resultados inventado y te devuelve la tabla.

## Días 5-6 (dom 22 – lun 23) · Colchón

Hueco deliberado. Por orden de prioridad:

1. **Si Ignacio va justo, recoge un agente.** El de cálculo envuelve `credit_math.py` y el de solvencia envuelve `policy.py`: los dos ficheros los has escrito tú, eres quien más rápido los hace.
2. Si va bien, adelanta el esqueleto de la landing page (maqueta sin datos).
3. Preparar la figura de arquitectura para el informe y las diapositivas. Se tarda más de lo que parece y la necesitan los dos entregables.

## Días 7-8 (mar 24 – mié 25) · Apoyo y congelación

- [ ] Día 7: revisar que `metrics.py` se come los resultados reales del sistema de Ignacio sin sorpresas.
- [ ] **Día 8: congelación de código.** Ver [`PLAN_SECUENCIAL.md`](PLAN_SECUENCIAL.md).

## Día 9 (jue 26) · Landing page

El enunciado pide que muestre cuatro cosas, y se enseña en la defensa:

- [ ] **El problema abordado**
- [ ] **La arquitectura del sistema** (el diagrama del plan maestro, redibujado en condiciones)
- [ ] **Ejemplos de funcionamiento**: un informe de riesgo real completo, con su traza de agentes. Vale una grabación si la demo en vivo es frágil
- [ ] **Resultados obtenidos**: la tabla A/B/C y la figura de coste/calidad

Recomendación: estática en `web/`, publicada con GitHub Pages. Sin backend, no se cae en la defensa. `agent/ui.py` ya existe como visor de trazas y sirve para la demo en vivo, por separado.

## Días 10-11 (vie 27 – sáb 28) · Cierre

- [ ] Revisión cruzada del informe de Ignacio. Léelo buscando **afirmaciones sin número detrás**.
- [ ] README de portfolio: qué hace, para quién, demo, diagrama, tabla de métricas, cómo levantarlo con una orden.
- [ ] `docker compose up api` en limpio + `uv run pytest` en verde.
- [ ] Ensayo de la demo en vivo, cronometrado.

## 30 nov · Diapositivas

20 minutos: problema, arquitectura, decisiones de diseño, resultados, conclusiones, y se enseña la landing page.
