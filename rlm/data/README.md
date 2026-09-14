# Datos de la fase 1

Aquí van vuestros problemas. Formato JSONL, una línea por problema, con al menos:

```json
{"question": "Un depósito pierde 18 litros y queda a la mitad...", "answer": "120"}
```

Podéis añadir los campos que necesite vuestro verificador (tests, tolerancia, esquema...):
llegan a las funciones de recompensa como argumentos con nombre.

Ficheros que esperamos ver:

- `train.jsonl`: problemas para SFT y GRPO.
- `test.jsonl`: problemas que el modelo no ha visto nunca, para `rlm/evaluate.py`.
- `sft_traces.jsonl`: trazas del profesor ya verificadas, generadas con `rlm/distill.py`.

Los ficheros grandes no se suben al repositorio: dejad un script que los regenere o un
enlace de descarga en este README.
