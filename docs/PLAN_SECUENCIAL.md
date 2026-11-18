# Plan secuencial · lo que hacemos juntos y en orden

> Plan maestro: [`PLAN_PROYECTO_FINAL.md`](PLAN_PROYECTO_FINAL.md) · Vías paralelas: [`PLAN_JAIME.md`](PLAN_JAIME.md) · [`PLAN_IGNACIO.md`](PLAN_IGNACIO.md)

Aquí está todo lo que **no** se puede paralelizar: el arranque conjunto, los puntos de sincronización y el embudo final. Si algo de este fichero se retrasa, se retrasa la entrega; lo de los ficheros individuales se puede reorganizar.

```
 Día   1   2   3   4   5   6   7  │  8   9  10  11
Jaime  ■   ■   ■   ■   ·   ·   ·  │  ·   ■   ■   ■
Igna.  ■   ■   ■   ■   ■   ■   ■  │  ■   ■   ■   ·
       ── PARALELO ───────────────┤── EMBUDO ──────
                        congelación (25 nov)
```

**Entrega: domingo 29 de noviembre** (Moodle) · **Defensa: 1-2 de diciembre**

---

## Día 1 (mié 18) · Arranque conjunto — BLOQUEANTE

Nadie escribe código del proyecto antes de terminar esto. Una mañana bien invertida compra siete días de trabajo sin bloqueos.

### 1. Congelar `multiagent/schemas.py`

Los modelos Pydantic que cruzan las fronteras entre agentes. **Una vez fijados no se tocan**: son el contrato que permite que cada uno trabaje contra un doble de prueba.

- [ ] `Solicitud` — ingresos netos, otras deudas mensuales, importe, plazo en meses, tipo nominal, comisión de apertura, tipo de contrato, antigüedad laboral
- [ ] `ResultadoCalculo` — cuota, TAE, coste total, DTI, y de qué herramienta sale cada número
- [ ] `ResultadoNormativo` — lista de afirmaciones, cada una con el id del fragmento que la respalda
- [ ] `ResultadoSolvencia` — decisión, motivos, y condición corregida si es contraoferta
- [ ] `VeredictoCritico` — aprobado o devuelto, con la lista de problemas encontrados
- [ ] `InformeRiesgo` — la salida final, con traza de agentes y coste

### 2. Cerrar los umbrales de la política

Son la verdad terreno. Si cambian después del día 3, **todos los números del informe dejan de valer**.

- [ ] DTI máximo para aprobar (propuesta: 35 %) y para denegar (40 %)
- [ ] Umbral de usura (propuesta: TAE > 2× el tipo medio de referencia)
- [ ] Regla de contrato temporal (propuesta: antigüedad < 6 meses y DTI > 30 % → denegar)

### 3. Desbloquear la infraestructura

- [ ] **Jaime:** pedir a Guzmán los pesos `sft_lora` y `grpo_lora` (no están en git) · regenerar el corpus (`uv run python -m rag.fetch_corpus`)
- [ ] **Ignacio:** resolver el acceso al 27B (hoy da `connection refused`) · mover la API al puerto 8001
- [ ] **Los dos:** `uv sync --extra train --extra rag --extra agent && uv run pytest -q` → 256 en verde

### 4. Decidir qué hace el tercer miembro

**No es opcional con este reparto**: es lo que mantiene a Ignacio dentro del calendario. Lo más separable, unos 3 días que no bloquean a nadie:

- [ ] 15 preguntas normativas difíciles escritas a mano (multi-salto, que necesiten dos fragmentos)
- [ ] Validación del juez contra 20 casos anotados
- [ ] Análisis de fallos y entradas de `EXPERIMENTS.md`

---

## Puntos de sincronización (días 2-7)

Fuera de estos momentos, cada uno va por libre con su fichero.

| Cuándo | Qué pasa | Si falla |
|---|---|---|
| **Día 2, final** | Jaime entrega `policy.py` | Holgura de 4 días. Tolerable |
| **Día 3, tarde** | **Puerta de decisión del 27B.** Si no está accesible, se decide **hoy** servir un modelo propio en la H200 (143 GB libres) | No se deja para el día 5. Decidir tarde cuesta dos días |
| **Día 4, final** | **Jaime entrega `metrics.py`** | Solo 2 días de holgura. **Es la entrega que no se puede mover** |
| **Día 5, final** | Primer informe de punta a punta sobre un caso de dev. **Lo validan los dos juntos** | Si Ignacio va por detrás, **se dice hoy**. Jaime recoge un agente con su colchón de los días 5-6 |
| **Día 6, final** | **Puerta de decisión del crítico.** Si no funciona, se entrega C sin crítico y la ablación 1 pasa a ser "no lo conseguimos y por qué" | No se arrastra al día 7 |
| **Día 7, final** | **CONGELACIÓN DE CÓDIGO** | Ver abajo |

---

## El embudo (días 8-11) · no se puede comprimir

```
congelación → ejecución final → tablas → informe → entrega → diapositivas
   día 7         noche 8          9        10        11        30 nov
```

Cada eslabón necesita el anterior terminado. Esto es lo que gobierna todo el plan: **la congelación del día 25 no es una sugerencia, es lo que da tiempo a que el embudo quepa.**

### Día 7 (mar 24), final · Congelación de código

- [ ] Los tres sistemas (A, B, C) corren bajo el **mismo arnés y las mismas métricas**
- [ ] `uv run pytest` en verde
- [ ] Merge de las dos ramas a `jaime-final-project`
- [ ] **A partir de aquí solo correcciones.** Ninguna funcionalidad nueva, por buena que parezca la idea

### Día 8 (mié 25) · Ejecución final

- [ ] Por la noche: A/B/C sobre los **150 casos de test**. Primera y única vez que se tocan
- [ ] En segundo plano, con resultados parciales en disco y reanudable
- [ ] Hay margen para repetir **una** vez. No más

### Día 9 (jue 26) · Ablaciones y landing en paralelo

- [ ] **Ignacio:** las 5 ablaciones sobre 100 casos; tablas y figuras
- [ ] **Jaime:** landing page, con los números de Ignacio según van saliendo
- [ ] Último punto donde los dos trabajan a la vez

### Día 10 (vie 27) · Informe

- [ ] Ignacio escribe las 3 páginas
- [ ] Jaime revisa el mismo día, buscando **afirmaciones sin número detrás**
- [ ] No olvidar la **discusión de escalabilidad**: el enunciado la pide explícitamente (producción y limitaciones)

### Día 11 (sáb 28) · Pulido y colchón

- [ ] `docker compose up api` en una máquina limpia
- [ ] README de portfolio
- [ ] `uv run pytest` en verde, sin secretos en el repositorio, sin marcadores pendientes
- [ ] Ensayo de la demo, cronometrado
- [ ] **Este día es colchón. Si se consume en terminar cosas, vamos tarde**

### Día 12 (dom 29) · ENTREGA

- [ ] Repositorio + landing page + informe por Moodle

### 30 nov – 1 dic · Defensa

- [ ] 20 minutos: problema, arquitectura, decisiones de diseño, resultados, conclusiones + enseñar la landing page
- [ ] Ensayo cronometrado
- [ ] Preguntan a cualquiera de los tres sobre cualquier parte. **Repasad la vía del otro**, no solo la vuestra

---

## Reglas comunes

- **Ramas:** `jaime-final-project` (integración) e `ignacio-final-project`. Merge al menos cada dos días. Nunca un merge grande el día 8.
- **`git push` al terminar cada sesión.** La DGX no tiene copias de seguridad.
- **`EXPERIMENTS.md` se escribe el mismo día del experimento**, no al final. Es criterio de corrección explícito.
- **Sin secretos en el repositorio.** El `.env` está en `.gitignore`; que siga así.
- **Si algo se recorta, se recorta funcionalidad, nunca evaluación.** El orden de sacrificio: ablación 4 (secuencial/paralelo) → ablación 5 (modelo pequeño) → el crítico → reducir ablaciones a 80 casos. **La comparación A/B/C no se toca nunca.**
