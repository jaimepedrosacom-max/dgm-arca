# Trabajar en la DGX

La DGX del departamento es una máquina compartida con varias GPUs NVIDIA. La vamos a usar
desde la primera sesión, y estas notas son para que la primera vez no se convierta en una
tarde de instalar cosas. Leedlas enteras antes de conectaros.

## Conectarse

```bash
ssh <usuario>@dgx.comillas.edu
```

Vuestro usuario y la forma de obtener acceso os la doy en clase. Una vez dentro, trabajáis
en vuestro directorio personal. Comprobad que Docker y las GPUs están disponibles:

```bash
nvidia-smi                         # driver, GPUs, memoria, quién está usando qué
docker --version
docker compose version
```

## La prueba de la primera sesión, paso a paso

1. **Clonad el repositorio** (o haced `git pull` si ya lo tenéis):

   ```bash
   git clone https://github.com/kendrickcetina/dgm-arca.git
   cd dgm-arca
   ```

2. **Elegid vuestra GPU.** Mirad `nvidia-smi`, buscad una libre, y anotad su índice. Todo
   lo que lancéis debe llevar `CUDA_VISIBLE_DEVICES` con ese índice para no pisar a nadie:

   ```bash
   export CUDA_VISIBLE_DEVICES=2          # la GPU 2, por ejemplo
   ```

3. **Apuntad la caché de modelos a un disco compartido**, si os lo indico, para no
   descargar el mismo modelo veinte veces:

   ```bash
   export HF_CACHE_DIR=/ruta/compartida/hf_cache    # si no, se usa ./hf_cache
   ```

4. **Construid la imagen.** La primera vez tarda: descarga torch con CUDA y el resto de
   librerías, y la imagen final ocupa unos 17 GB. Las siguientes veces Docker reutiliza las
   capas y solo copia el código:

   ```bash
   docker compose build
   ```

5. **Comprobad que el contenedor ve la GPU:**

   ```bash
   docker compose run --rm check-gpu
   ```

   Tenéis que ver vuestra GPU en la tabla, `bf16 supported: True` y un número de TFLOP/s
   razonable. Si el script dice que no ve CUDA, id a la sección de problemas.

6. **Lanzad el smoke test:**

   ```bash
   docker compose run --rm smoke
   ```

   En otra terminal, `watch -n 2 nvidia-smi` para ver la memoria y la utilización de vuestra
   GPU subir. En 10-15 minutos veréis una línea por paso con las recompensas, y al final la
   comparación antes/después. Lo que tiene que pasar está explicado en
   [`smoke/README.md`](../smoke/README.md).

7. **Levantad la API** y comprobad que responde:

   ```bash
   docker compose up -d api
   curl -s localhost:8000/health | python3 -m json.tool
   docker compose down
   ```

   Todas las fases saldrán como `pending`. Es lo esperado: todavía no habéis hecho nada.

Si los siete pasos funcionan, el entorno está listo y podéis empezar la fase 1.

## Trabajar sin Docker

También podéis usar `uv` directamente en la DGX, sin contenedor:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
uv sync --extra train --extra rag --extra agent
uv run arca-check-gpu
uv run arca-smoke
```

`uv.lock` fija torch con CUDA 12.8 en Linux, así que no hay que elegir ruedas a mano. Para
la entrega, eso sí, la API tiene que levantar con `docker compose up api`.

## Entrenamientos largos

Un entrenamiento de la fase 1 puede durar horas. No lo dejéis atado a vuestra sesión SSH:

```bash
docker compose run -d --name arca-train-<equipo> train \
    uv run python -m rlm.train_grpo --data rlm/data/train.jsonl --steps 500
docker logs -f arca-train-<equipo>          # seguir el progreso
```

O, sin Docker, con `tmux` o `nohup`. Guardad checkpoints a menudo (`--save-steps`) y anotad
en `EXPERIMENTS.md` qué lanzasteis, cuándo y con qué parámetros.

## Convivir en una máquina compartida

- Una GPU por equipo salvo que os diga lo contrario. Siempre `CUDA_VISIBLE_DEVICES`.
- Mirad `nvidia-smi` antes de lanzar nada. Si una GPU tiene memoria ocupada, es de alguien.
- No dejéis procesos zombis: `docker ps` y `docker rm -f` de lo vuestro al terminar.
- Los modelos pequeños (0.6B-1.7B) caben de sobra en una GPU con LoRA y bf16. Si os quedáis
  sin memoria, antes de pedir otra GPU bajad `num_generations`, `max_completion_length` o
  activad gradient checkpointing (ya está activado en los scripts).
- La caché de Hugging Face puede crecer mucho. Si compartimos disco, respetad la ruta común.

## Cuando algo falla

**`check_gpu.py` no ve CUDA dentro del contenedor.** Casi siempre es una de tres cosas:
`nvidia-container-toolkit` no está instalado o Docker no lo tiene configurado (probad
`docker run --rm --gpus all nvidia/cuda:12.8.1-base-ubuntu22.04 nvidia-smi`), la variable
`CUDA_VISIBLE_DEVICES` está vacía o apunta a una GPU que no existe, o el driver es más
antiguo que la versión de CUDA de torch (necesitáis driver 570 o superior para CUDA 12.8;
lo veis en la primera línea de `nvidia-smi`).

**`CUDA out of memory`.** Bajad `--num-generations`, `--max-completion-length` o el tamaño
del modelo. Y comprobad con `nvidia-smi` que no estáis compartiendo GPU sin saberlo.

**Descargas lentas o límite de peticiones a Hugging Face.** Poned vuestro `HF_TOKEN` en
`.env` (es gratis) y usad la caché compartida.

**`docker compose` no reconoce `deploy.resources.reservations.devices`.** Necesitáis Compose
v2. `docker compose version` debe decir 2.x. Si solo tenéis `docker-compose` (v1), avisadme.

**El smoke test arranca pero la recompensa de formato no se mueve de cero en 40 pasos.**
Mirad las respuestas que imprime al final. Si están todas truncadas (`clipped 1.000` en cada
línea), el modelo no llega a cerrar la etiqueta: subid `--max-completion-length`. Si el
modelo no es Qwen3, comprobad que su tokenizer no elimina `<think>` al decodificar.

**Cualquier otra cosa.** Copiad el error completo, el comando exacto y la salida de
`check_gpu.py`, y traedlo a clase o al canal de la asignatura.
