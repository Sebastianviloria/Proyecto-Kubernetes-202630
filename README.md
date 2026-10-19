# Proyecto Kubernetes 202630 — Jobs parametrizados desde línea de comandos

Proyecto **grupal**: se trabaja en los grupos que se han venido usando durante el curso y
se entrega antes de la clase de la semana 12 (ver [Entrega](#entrega)).

Ya han trabajado con manifiestos `Job` de Kubernetes. En este proyecto el grupo usará esa
misma idea desde una herramienta de línea de comandos escrita en Python: en vez de editar
un YAML para cada ejecución, el programa presenta un menú donde el usuario elige el
lenguaje, la tarea y la complejidad, y crea el `Job` usando el cliente oficial de
Kubernetes. El mismo menú permite revisar el estado de las tareas lanzadas y ver sus
logs.

Kubernetes crea entonces un Pod y ejecuta el contenedor elegido con los argumentos
suministrados. El recorrido completo que el grupo debe entender —y saber defender— es:

```text
gestor_jobs.py -> Job de Kubernetes -> Pod -> contenedor con argumentos -> logs
```

No se incluye interfaz web, múltiples nodos, RBAC propio ni selección manual de nodos.
Esas capas quedan para la versión completa.

## Qué se entrega y qué construye el grupo

Se entrega como infraestructura, ya funcional:

- `imagenes/`: tres imágenes (Python, Java y C). Las tres implementan el mismo conjunto de
  tareas y el mismo contrato: reciben `<tarea> <N>` como argumentos, escriben logs en
  stdout y cierran con un JSON de una línea.
- `catalogo/tareas.json`: catálogo modificable de lenguajes, imágenes y tareas admitidas.
- `k8s/00-namespace.yaml`: namespace dedicado para los Jobs.
- `scripts/preparar-imagenes.ps1`: construye las imágenes y las carga en Minikube.

Construye el grupo, sobre el esqueleto de `entrega/`:

- `entrega/gestor_jobs.py`, un programa de menú con cinco capacidades:
  1. selección de lenguaje, tarea y complejidad;
  2. creación del `Job` mediante el cliente Python de Kubernetes;
  3. revisión del estado de las tareas lanzadas;
  4. consulta de los logs de una tarea;
  5. limpieza de las tareas ya terminadas, para dejar el namespace en orden.

- Una tarea adicional, en el lenguaje que el grupo prefiera. Implica ajustar el programa
  del contenedor que corresponda, reconstruir y cargar la imagen, añadir la tarea en
  `catalogo/tareas.json` y su fila en la tabla `TAMANOS` del gestor.

### Las tareas que traen los contenedores

| tarea | qué hace | parámetro N |
|---|---|---|
| `hola` | saluda e imprime datos del entorno de ejecución | no usa N |
| `ordenar` | genera N enteros aleatorios y los ordena, verificando el resultado | cantidad de elementos |
| `fib` | calcula fib(N) recursivamente, sin memoización: satura una CPU | N |
| `matriz` | multiplica dos matrices densas de N×N con triple bucle | dimensión |

Las cuatro están implementadas en los tres lenguajes, así que la misma tarea puede
compararse entre Python, Java y C. La tarea `hola` es la más reveladora en esa
comparación: el arranque de la JVM se nota frente a Python y a un binario de C.

### Qué significa la complejidad

La complejidad elegida (`baja`, `media`, `alta`) fija dos cosas a la vez, ambas en el
gestor:

- **los recursos** que el Job le pide a Kubernetes: `requests` y `limits` de CPU y
  memoria, en la tabla `NIVELES`;
- **el tamaño del trabajo**, el valor de N que recibe la tarea, en la tabla `TAMANOS`.

Por eso un nivel alto no sólo tarda más: también reserva más CPU y permite más memoria.
Bajar el `mem_limit` por debajo de lo que necesita la tarea es la forma de provocar un
`OOMKilled` y verlo reflejado en el estado del Job.

El esqueleto ya trae la estructura del programa y el andamiaje del cliente de Kubernetes
armado: el bucle del menú, los listados numerados, la carga del catálogo, el objeto `Job`
con sus clases anidadas y la búsqueda del Pod por etiqueta. Lo pendiente está marcado con `TODO` y con `...` en los
campos a completar, de modo que el trabajo sea entender el `Job` y no pelear con Python.

No está permitido resolver el proyecto invocando `kubectl` desde Python: el `Job` debe
crearse con el cliente `kubernetes`.

Como la entrega se defiende en grupo, todos los integrantes deben poder explicar el
código y el recorrido completo, sin importar quién escribió cada parte.

## Preparación

Ejecuten los siguientes comandos desde la carpeta raíz del proyecto:

```powershell
minikube start
kubectl apply -f .\k8s\00-namespace.yaml
.\scripts\preparar-imagenes.ps1
```

Instalen luego las dependencias de Python:

```powershell
pip install -r .\entrega\requirements.txt
```

Los nombres de imagen son deliberadamente locales (`kubernates-202630-*`) y no se publica
ninguna imagen durante el proyecto.

`preparar-imagenes.ps1` construye directamente contra el demonio Docker de Minikube, así
que **hay que volver a ejecutarlo cada vez que se cambie el programa de un contenedor**.
No usen `minikube image load` para actualizar una imagen: no reemplaza un tag que ya
existe, y el clúster seguiría ejecutando la versión anterior sin avisar.

## Ejecución

Una vez implementado el gestor, se ejecuta sin argumentos:

```powershell
python .\entrega\gestor_jobs.py
```

El menú ofrece:

```text
=== Gestor de Jobs - namespace estudiantes-202630 ===
  1. Ejecutar una tarea nueva
  2. Revisar el estado de las tareas
  3. Ver los logs de una tarea
  4. Limpiar las tareas terminadas
  5. Salir
```

La opción 1 pide lenguaje, tarea y complejidad en listas numeradas —el menú de
complejidad muestra el N y la memoria que implica cada nivel— y al terminar imprime el
nombre del Job creado con sus recursos. La opción 2 lista todas las tareas del namespace con su estado.
La opción 3 deja elegir una de esas tareas y muestra la salida de su contenedor. La
opción 4 borra las tareas ya terminadas —completadas o fallidas— tras pedir confirmación,
y deja intactas las que siguen corriendo.

Prueben las tres tareas incluidas variando la complejidad, y contrasten lo que muestra el
menú con lo que reporta el clúster:

```powershell
kubectl get jobs,pods -n estudiantes-202630
kubectl logs -n estudiantes-202630 job/<nombre-del-job>
```

## Entrega

El proyecto es grupal y se trabaja en los grupos que se han venido usando durante el
curso. La entrega vence **antes de la clase de la semana 12**.

1. Link al repositorio público entregado por la actividad del catálogo, antes de la clase
   de la semana 12.
2. En el README del repositorio se debe explicar la implementación.
3. En el README se debe agregar un link a un video de YouTube con el demo del proyecto.
4. El grupo completo debe estar disponible para defender la entrega.

## Implementación del grupo

### Cómo funciona `gestor_jobs.py`

El recorrido completo es:

`gestor_jobs.py` (cliente Python `kubernetes`, sin `kubectl`) → `Job` (batch/v1) → Pod creado desde la plantilla del Job → contenedor con argumentos `[tarea, N]` → el programa imprime en stdout (la última línea es un JSON) → el gestor lee los logs del Pod, que encuentra por la etiqueta `job-name=<nombre>`.

Las 7 funciones que completan el esqueleto:

| función | qué hace |
|---|---|
| `seleccionar` | Muestra los menús de lenguaje, tarea y complejidad. Toma la imagen del catálogo, arma `args = [tarea, str(N)]` con la tabla `TAMANOS` y devuelve la fila de `NIVELES` del nivel elegido. |
| `crear_job` | Arma el contenedor `V1Container` (nombre `tarea`, `image_pull_policy="Never"`, `args`, `V1ResourceRequirements` con requests y limits), el `V1PodSpec(restart_policy="Never")`, el `V1JobSpec(backoff_limit=0)` y el `V1Job`. Lo envía con `create_namespaced_job` al namespace `estudiantes-202630`. |
| `describir_estado` | Traduce el `status` de un Job a texto según sus contadores `active`, `succeeded` y `failed` (con `or 0`, porque valen `None` mientras Kubernetes no los fija): "en ejecucion", "termino correctamente", "fallo" o "pendiente (el Pod todavia no ha arrancado)". |
| `listar_jobs` | Llama a `list_namespaced_job` y devuelve `[(nombre, estado)]`, usando `describir_estado` para cada Job. |
| `consultar_logs` | Busca el Pod con `list_namespaced_pod` y `label_selector=job-name=<nombre>`, toma el primero y lee su log con `read_namespaced_pod_log(_preload_content=False)`, decodificando `.data` en UTF-8. |
| `jobs_terminados` | Devuelve los nombres de los Jobs sin Pods activos y con al menos un Pod correcto o fallido; los que siguen corriendo no aparecen. |
| `eliminar_job` | Llama a `delete_namespaced_job` con `V1DeleteOptions(propagation_policy="Background")`, para que Kubernetes borre también el Pod del Job. |

Además, `consultar_estado(nombre)` es un ayudante que lee un Job con `read_namespaced_job_status` y reutiliza `describir_estado`.

### Por qué esos campos del Job

- **La tarea y N van en `args`, no en `command`.** `command` reemplaza el `ENTRYPOINT` de la imagen (en `imagenes/python/Dockerfile` es `ENTRYPOINT ["python", "/app/tareas.py"]`) y el contenedor falla con "executable file not found". `args` se añade al `ENTRYPOINT`, así el programa recibe `<tarea> <N>`.
- **`backoff_limit=0`**: cero reintentos. Si la tarea falla (por ejemplo, `OOMKilled`), el Job queda como fallido de inmediato.
- **`restart_policy="Never"`**: el Pod no reinicia el contenedor, así el fallo queda visible en lugar de ocultarse en reinicios.
- **`image_pull_policy="Never"`**: las imágenes `kubernates-202630-*` son locales en Minikube y no existen en ningún registry, así que Kubernetes no debe intentar descargarlas.

### Tarea extra: `primos`

- **Qué hace:** cuenta los primos hasta N por división por prueba, sin criba, a propósito, para consumir CPU. Imprime el progreso cuando N ≥ 100000 y su JSON final trae `n`, `cuenta` y `ultimo`.
- **Tamaños** (tabla `TAMANOS`): baja N=20000, media N=300000, alta N=2000000. En la prueba, alta tardó unos 11 segundos.
- **Archivos tocados:**
  - `imagenes/python/tareas.py`: la función `primos` y su entrada en `TAREAS`.
  - `catalogo/tareas.json`: `"primos"` en las tareas de python.
  - `entrega/gestor_jobs.py`: la fila `primos` en `TAMANOS`.
- Solo existe en Python; después de cambiarla hay que reconstruir la imagen en Minikube.

> Probado en macOS (Apple Silicon); los comandos para Windows están en la sección de Preparación.

## Video del demo

Link: https://youtu.be/AL-KuDAMS-Y
