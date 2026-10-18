# Proyecto Kubernetes 202630 - gestor de Jobs

Estudiante de Uninorte, proyecto de curso. Hablame en español. Debo poder explicar cada linea en la defensa: comenta brevemente que significa cada campo del Job y no cambies la estructura del esqueleto.

## Reglas del proyecto
- Todo el trabajo va en `entrega/gestor_jobs.py`: completar los `TODO` y los `...`. No tocar el menu ni otros archivos salvo que te lo pida.
- Prohibido usar `kubectl` o `subprocess` desde el gestor. El Job se crea con el cliente Python `kubernetes`.
- La tarea y N van en `args` del contenedor, NUNCA en `command` (reemplaza el ENTRYPOINT y falla con "executable file not found").
- Job: `image_pull_policy="Never"` (imagenes locales), `restart_policy="Never"`, `backoff_limit=0`. Namespace `estudiantes-202630`.
- Las tablas NIVELES y TAMANOS ya estan definidas. Una tarea nueva agrega su fila en TAMANOS y su entrada en `catalogo/tareas.json`.

## Entorno (Mac arm64)
- Python: `source .venv/bin/activate`. Ejecutar: `python entrega/gestor_jobs.py`
- Minikube con driver docker. Imagenes: `kubernates-202630-<lenguaje>:1.0` ("kubernates" con "a" es el nombre real).
- Reconstruir una imagen: `eval $(minikube -p minikube docker-env)` y `docker build -t kubernates-202630-<lenguaje>:1.0 imagenes/<lenguaje>`. No usar `minikube image load`.
- Para verificar contra el cluster, dime que comando correr y yo pego el resultado.

## Flujo
- Trabajar en la rama `gestor-jobs`. No hacer push sin que yo lo pida.
- Pendiente despues del gestor: tarea extra en un lenguaje (contenedor + catalogo + TAMANOS), README con la explicacion y link al video de YouTube.
