# Plan de Ignacio · vía sistema y evaluación

> Plan maestro: [`PLAN_PROYECTO_FINAL.md`](PLAN_PROYECTO_FINAL.md) · Parte común: [`PLAN_SECUENCIAL.md`](PLAN_SECUENCIAL.md) · La otra vía: [`PLAN_JAIME.md`](PLAN_JAIME.md)

**Carga:** es la vía pesada, y de los días 4 a 9 estás en el camino crítico.
**Lee esto antes de empezar:** el plan solo cabe en 11 días si el tercer miembro absorbe trabajo real (gold set, validación del juez, análisis de fallos) o si Jaime recoge un agente en los días 5-6. Si el día 5 vas por detrás, **dilo ese mismo día**, no el 8. El colchón de Jaime existe justo para eso.

## Lo que recibes y lo que entregas

| | Qué | Cuándo | De/para quién |
|---|---|---|---|
| Recibes | `multiagent/schemas.py` congelado | día 1 | reunión conjunta |
| Recibes | **`multiagent/policy.py`** | final del día 2 | Jaime — lo necesitas el día 6 |
| Recibes | **`eval/metrics.py`** | final del día 4 | Jaime — lo necesitas el día 6 |
| Recibes | `bench/cases_dev.jsonl` | final del día 3 | Jaime |
| **Entregas** | Un informe de punta a punta | final del día 5 | validación conjunta |
| **Entregas** | Resultados de la evaluación | día 8-9 | Jaime (landing) y el informe |

---

## Día 1 (mié 18) · Desbloqueo y cliente del 27B

- [ ] **Reunión conjunta** → [`PLAN_SECUENCIAL.md`](PLAN_SECUENCIAL.md). No escribas código antes de congelar `schemas.py`.
- [ ] **Resolver el acceso al 27B.** Hoy da `connection refused` en `127.0.0.1:8000/v1/models`. Averiguar si hay que levantarlo, si vive en otro nodo o si hay que pedir el perfil. Preguntar al equipo docente **hoy**, no el viernes.
- [ ] **Mover la API al 8001** (`ARCA_API_PORT=8001` y `docker-compose.yml`): el 8000 es del servicio de inferencia y chocan.
- [ ] Empezar `multiagent/llm.py`:
  - cliente asíncrono con `httpx` contra `http://127.0.0.1:8000/v1/chat/completions`, modelo `qwen3.8-27b-fp8`
  - reintentos con espera exponencial
  - **instrumentación**: tokens de entrada y salida, latencia y número de llamadas, por agente y por petición. Sin esto no hay eje de coste y la mitad de las métricas se caen
  - `chat_template_kwargs` para activar o desactivar el modo de razonamiento (ver `docs/dgx-instructions.tex`, líneas 132-156)

**Hecho cuando:** una llamada de prueba devuelve texto y el contador de tokens cuadra.

## Día 3 (vie 20) · `llm.py` terminado

- [ ] **Salida estructurada con Pydantic** vía `response_format` de vLLM. El profesor dio un tutorial (`docs/pydantic_structured_output.ipynb`): se espera que lo usemos y ahorra todo el parseo frágil que sufrimos en la fase 4.
- [ ] **Caché en disco** de respuestas por hash del prompt. Te va a salvar: durante el desarrollo repites las mismas llamadas decenas de veces.
- [ ] Un **doble de prueba** (`FakeLLM` con respuestas guionizadas) para que los tests no toquen la GPU. Mismo patrón que el cerebro simulado de la fase 4.

**Hecho cuando:** le pides un modelo Pydantic y te devuelve una instancia válida, y los tests corren sin GPU.

## Día 4 (sáb 21) · Agente de cálculo y agente normativo

- [ ] **Agente de cálculo** (`multiagent/agents/calc.py`): recibe la solicitud, llama a las herramientas de `tool_use/` y devuelve `ResultadoCalculo`. **Regla dura: ningún número que no venga de `credit_math.py`.** Nada de aritmética mental, que es exactamente donde fallaba el 1.7B.
- [ ] **Agente normativo** (`multiagent/agents/norm.py`): consulta con el retriever híbrido de la fase 3 (λ=0.6, chunking semántico 0.7) y devuelve afirmaciones **con el id del fragmento que las respalda**. Reutiliza `rag/retriever.py` y la comprobación de citas de `rag/generate.py`.
- [ ] Subir el `top_k` por defecto respecto a la fase 4: con `top_k=1` el agente cogía el apartado equivocado (está anotado en `EXPERIMENTS.md`).

**Hecho cuando:** los dos responden con su modelo Pydantic sobre 5 casos de dev.

## Día 5 (dom 22) · Orquestador

- [ ] `multiagent/orchestrator.py`: descompone la solicitud, lanza los tres subagentes con **`asyncio.gather`** (paralelo real, no un bucle `for`) y sintetiza el `InformeRiesgo`.
- [ ] Registrar la **traza completa**: qué agente, qué entrada, qué salida, cuántos tokens, cuánto tardó. Es lo que enseña la landing page y lo que necesita el análisis de fallos.
- [ ] Flag `--secuencial` desde el principio: es la **ablación 4** y cuesta cinco minutos ahora, una tarde después.

**Hecho cuando:** un caso de dev produce un informe completo de punta a punta. **Lo validáis los dos juntos esta noche** (punto de sincronización).

## Día 6 (lun 23) · Crítico, solvencia y arnés

- [ ] **Agente de solvencia** (`multiagent/agents/solvency.py`): aplica la política *recuperada del corpus* y propone la decisión con motivos. Usa `policy.py` de Jaime **solo para la verdad terreno en los tests**, nunca dentro del agente — si el agente llama a la política directamente no estás midiendo nada.
- [ ] **Agente crítico** (`multiagent/agents/critic.py`): recalcula las cifras de forma independiente y comprueba que cada afirmación normativa tenga fragmento que la respalde. Devuelve o aprueba. **Máximo 2 rondas.**
- [ ] `eval/run.py`: concurrencia 8-16, **reanudable** (resultado por caso en disco, si se corta la sesión de la DGX no se pierde nada), selector de sistema y de ablación.
- [ ] `eval/judge.py`: juez de fidelidad con el 27B.

**Puerta de decisión:** si el crítico no funciona al final del día, **se entrega C sin crítico** y la ablación 1 pasa a ser "no lo conseguimos y por qué". Eso también puntúa. No lo arrastres al día 7.

## Día 7 (mar 24) · Sistemas A y B, endpoint

- [ ] **Sistema A**: una sola llamada al 27B pidiendo el informe entero, sin herramientas ni RAG. Es media hora de trabajo y es el suelo de la comparación.
- [ ] **Sistema B**: `agent/react_agent.py` con el 27B como cerebro. Ya existe y está depurado; es la comparación más exigente y la más honesta. **No lo reescribas**, solo enchufa el cerebro.
- [ ] Endpoint `POST /credit-report` en `api/app.py`, con la traza en la respuesta.
- [ ] Los tres sistemas corriendo bajo el **mismo arnés y las mismas métricas**. Si A, B y C no se miden idénticamente, la comparación no vale.

**Final del día 7: CONGELACIÓN.** A partir de aquí solo correcciones.

## Días 8-9 (mié 25 – jue 26) · Ejecución y ablaciones

- [ ] **Noche del 25:** lanzar A/B/C sobre los **150 de test**. Es la primera vez que se tocan. Dejarlo corriendo con `run_in_background` y resultados parciales en disco.
- [ ] Día 9: las 5 ablaciones sobre 100 casos — sin crítico, sin RAG, sin herramientas, secuencial, agente de cálculo pequeño (0.6B+GRPO, si aparecieron los pesos).
- [ ] Generar tablas y figuras. Pasárselas a Jaime para la landing el mismo día 9.
- [ ] **Validar el juez** contra 20 casos anotados a mano. El juez es el mismo modelo que genera: hay sesgo de autoevaluación y hay que decirlo en el informe.

Presupuesto: ~850 ejecuciones, del orden de 10-15 llamadas al LLM cada una en el sistema C. Con concurrencia es una noche, no una tarde. Hay margen para repetir **una** vez.

## Día 10 (vie 27) · Informe

3 páginas. Es poco: las tablas y figuras cargan el peso y el detalle se referencia a `EXPERIMENTS.md`.

- [ ] definición del problema · diseño del sistema · decisiones de arquitectura · experimentos · métricas · análisis de resultados
- [ ] **Discusión de escalabilidad** (la pide el enunciado explícitamente): qué haría falta para producción y limitaciones actuales
- [ ] Todo número con su intervalo de confianza. Toda afirmación con su número.

Jaime lo revisa el mismo día.

## Día 11 (sáb 28) · Cierre

- [ ] Entradas de `EXPERIMENTS.md` **fechadas**. Las que dicen "probamos esto, no funcionó, creemos que por aquello" valen más que las que dicen "todo bien".
- [ ] Análisis de fallos: 5 casos concretos leídos de verdad, no agregados.
