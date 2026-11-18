# Sistema multi-agente de análisis de riesgo de crédito al consumo

## 1. Descripción del problema

Estudiar una solicitud de préstamo al consumo exige combinar cálculo financiero exacto (cuota, TAE, coste total, ratio de endeudamiento), conocimiento normativo (crédito al consumo, transparencia, usura) y criterio para decidir y justificar.

Proponemos un sistema que, dada una solicitud (datos del solicitante y del préstamo), genere un **informe de riesgo** con:

- recomendación (aprobar / contraoferta / denegar),
- cálculos trazables,
- citas a la normativa aplicada.

## 2. Motivación y aplicación

Los LLM fallan justo en lo que más importa aquí: calculan mal (la TAE es una ecuación implícita que no se resuelve "de cabeza") y pueden inventar citas legales.

Es un dominio donde casi todo es verificable automáticamente, lo que permite evaluar con rigor, y donde un error tiene coste real.

Partimos de trabajo previo del equipo: un verificador de cálculos de crédito y un modelo de razonamiento entrenado con SFT + GRPO (práctica ARCA).

## 3. Arquitectura propuesta (tentativa)

Un **orquestador** descompone la solicitud y lanza subagentes en paralelo:

- **Agente de cálculo**: herramientas deterministas (cuota francesa, TAE por bisección, ratios), sin aritmética mental.
- **Agente normativo (RAG)**: corpus de normativa (p. ej. Ley 16/2011 de contratos de crédito al consumo, Directiva (UE) 2023/2225, directrices EBA de concesión de préstamos, Ley de Represión de la Usura) con recuperación híbrida (densa + BM25) y citas.
- **Agente de solvencia**: aplica una política de riesgo (umbrales de endeudamiento, perfil del solicitante) y propone la decisión.
- **Agente crítico/verificador**: recalcula las cifras y comprueba que cada afirmación normativa esté respaldada por un fragmento recuperado; puede devolver el caso a los demás agentes.

El orquestador sintetiza el informe final.

**Extensión opcional**: entrada multimodal (nómina escaneada → extracción de ingresos).

## 4. Modelos y frameworks

- **Qwen3.8-27B-FP8** (servicio central) como LLM de los agentes.
- Un **modelo pequeño** (Qwen3-0.6B/1.7B) con SFT + GRPO, entrenado en el perfil MIG, como candidato a agente de cálculo barato.
- **Stack**: Python, FastAPI, ChromaDB, rank-bm25, sentence-transformers, smolagents u orquestación propia con function calling, pytest.

## 5. Plan preliminar de evaluación

**Datos.** Benchmark sintético de ~200 solicitudes con verdad terreno calculable (generador de la práctica ampliado, decisión fijada por política determinista) y ~30 preguntas normativas con pasaje de referencia.

**Métricas.**

- exactitud numérica (coincidencia a 2 decimales),
- accuracy y F1 de la decisión,
- precisión/recall de citas y fidelidad al contexto recuperado,
- coste (tokens y latencia por solicitud).

**Alternativas comparadas.**

| | Sistema |
|---|---|
| A | LLM único sin herramientas |
| B | Agente único ReAct con herramientas y RAG |
| C | Sistema multi-agente |

**Ablaciones.** Sin crítico, sin RAG, sin herramientas de cálculo, y agente de cálculo pequeño fine-tuned frente al 27B.
