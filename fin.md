# Reto del Brazo 2 — Guía completa · Grupo 5

**Universidad ESAN · Curso de Robótica · Ing. Gonzalo Justiniani**

Este documento es la ruta única, de principio a fin, para el Reto del Brazo 2. Integra la
configuración de la Raspberry, el armado del workspace, los dos modos de atención del
broker, la corrida de los clientes, la medición y la verificación de la cinemática.

Todo lo que aparece aquí corresponde **al código que hay en este repositorio**, que ya
trae la cinemática, el broker y sus dos políticas implementados. Donde el enunciado del
curso decía algo distinto, este documento manda.

> **De dónde sale cada cosa.** Los pasos de red y entorno vienen de la guía de
> configuración del curso. Los pasos de compilación, corrida, medición y verificación de
> FK vienen de la guía del reto. La sección *Los dos modos del broker* es propia de esta
> solución: el `broker.py` de este repo tiene una política más y su forma de activarla
> tiene tres trampas que conviene leer antes de medir.

---

## Antes de empezar · lo único que tienes que conseguir del profesor

| Dato | Para qué | Valor |
|---|---|---|
| **IP del Jetson** de tu equipo | Para hacer `ssh` y para el Discovery Server | `172.51.9.___` — **preguntálo** |
| **ROS_DOMAIN_ID** | El canal por el que se hablan los nodos | **`5`** (es el de este grupo) |
| Cuántos integrantes | Uno cliente por persona | de tu equipo |

Anotá la IP del Jetson en un lugar visible. Todo el documento la usa como `$JETSON`.

---

## Qué corre en cada máquina

| Máquina | Qué corre ahí | Cuántos |
|---|---|---|
| **El Jetson** | El driver `sync_plan_nx` — el único que abre el puerto serie | Uno por equipo |
| **El Jetson** | El broker `arm_broker` — el único que publica en `/joint_states` | Uno por equipo |
| **Tu Raspberry** | Tu cliente `arm_client` y tus comandos | Uno por integrante |

El broker va **dentro del Jetson**, junto al driver, y no en las Raspberry. Es el que
traduce "pedidos en cola" en "una pose a la vez". Las Raspberry solo mandan goals.

**Antes de que alguien mande la primera pose, espacio despejado alrededor del brazo.**

---

## Paso 1 · Conectar la Raspberry a la red del laboratorio

Por cable o por la WiFi del laboratorio. Comprobá qué dirección te tocó:

```bash
ip -4 addr show scope global
```

Tiene que salir una dirección del rango del laboratorio, por ejemplo `172.51.9.30`.

Si sale algo que empieza por `169.254`, no recibió dirección: revisá el cable o la WiFi
antes de seguir. Anotá tu IP; la vas a necesitar si algo falla.

---

## Paso 2 · Comprobar que llegás al Jetson

```bash
JETSON=172.51.9.__      # reemplazá por la IP real de tu equipo

ping -c 3 $JETSON
```

Si no responde, el problema es de red y no de ROS 2. Avisale al profesor antes de seguir:
nada de lo que viene va a funcionar.

Guardá la IP para no escribirla cada vez (esto lo vuelve a hacer el paso 4, pero
podés hacerlo ya):

```bash
echo "export JETSON=$JETSON" >> ~/.bashrc
```

---

## Paso 3 · Instalar lo que hace falta

```bash
sudo apt update
sudo apt install -y \
  ros-humble-rosidl-default-generators \
  ros-humble-rosbag2 \
  ros-humble-rosbag2-storage-default-plugins \
  ros-humble-sensor-msgs \
  ros-humble-demo-nodes-cpp \
  python3-colcon-common-extensions \
  python3-matplotlib
```

Para qué sirve cada uno en **este** reto:

| Paquete | Para qué lo necesita el reto |
|---|---|
| `rosidl-default-generators` | Compila `arm_broker_interfaces`, que define la acción y el mensaje propios |
| `rosbag2` | Grabar la evidencia: el paso de medición no existe sin él |
| `rosbag2-storage-default-plugins` | El plugin `sqlite3` que abre `exportar_csv.py`. Sin él, el exportador falla al abrir el bag |
| `colcon-common-extensions` | Compila los dos paquetes del repo |
| `sensor_msgs` | El `JointState` que el broker publica en `/joint_states` |
| `demo-nodes-cpp` | Un par de nodos de prueba para comprobar la conexión |
| `matplotlib` | Genera la figura comparativa. Sin él, `metricas.py` imprime los números y no el gráfico |

Comprobá que quedaron:

```bash
ros2 --version
command -v colcon >/dev/null && echo "colcon OK"
ls /opt/ros/humble/share/rosidl_default_generators >/dev/null && echo "rosidl OK"
ls -d /opt/ros/humble/share/rosbag2* >/dev/null && echo "rosbag2 OK"
python3 -c "import matplotlib" 2>/dev/null && echo "matplotlib OK"
```

---

## Paso 4 · Configurar el entorno

Cuatro líneas en tu `~/.bashrc`. El dominio de este grupo es **5**:

```bash
echo 'source /opt/ros/humble/setup.bash' >> ~/.bashrc
echo 'export ROS_DOMAIN_ID=5'            >> ~/.bashrc
echo 'export ROS_LOCALHOST_ONLY=0'       >> ~/.bashrc
echo "export JETSON=$JETSON"             >> ~/.bashrc
exec bash
```

Verificá:

```bash
echo "$ROS_DISTRO · dominio $ROS_DOMAIN_ID · localhost_only $ROS_LOCALHOST_ONLY"
```

Debe decir `humble · dominio 5 · localhost_only 0`.

**Qué significa cada línea**

`source /opt/ros/humble/setup.bash` carga ROS 2. Sin esto el comando `ros2` no existe.

`ROS_DOMAIN_ID` es el canal por el que se hablan los nodos. Dos máquinas en dominios
distintos no se ven **aunque estén en la misma red y se hagan ping**. Tiene que ser
exactamente el mismo número que el del Jetson.

`ROS_LOCALHOST_ONLY=0` permite que tus nodos salgan de tu máquina. Con el valor en `1`,
ROS 2 funciona pero solo habla consigo mismo, y vas a ver únicamente tus propios nodos.

`JETSON` es la IP de tu equipo, para no escribirla cada vez que hacés `ssh`.

---

## Paso 5 · Reiniciar el demonio de ROS 2

ROS 2 guarda en memoria la lista de lo que vio la última vez. Si cambiás el dominio o las
variables sin reiniciarlo, te sigue mostrando la información vieja.

```bash
ros2 daemon stop
ros2 daemon start
```

Hacé esto cada vez que cambies algo del paso 4.

---

## Paso 6 · Comprobar que ves el robot

Primero, **una persona de tu equipo** levanta el driver dentro del Jetson y deja esa
terminal abierta:

```bash
ssh jetson@$JETSON
source ~/jetcobot_colcon_ws/install/setup.bash
ros2 run jetcobot_driver sync_plan_nx
```

Solo una persona. El brazo tiene un único puerto y no se comparte: si dos lo intentan, el
segundo recibe un error.

Y ahora, desde tu Raspberry:

```bash
ros2 node list
ros2 topic list
ros2 topic info /joint_states -v
```

Lo que debe salir:

```
/control_sync_plan

/joint_states
/parameter_events
/rosout

Type: sensor_msgs/msg/JointState
Subscription count: 1
```

Si aparece `/control_sync_plan`, tu Raspberry ya está conectada al robot.

---

## Paso 7 · Si no aparece nada

Recorré esta tabla en orden. Cada línea descarta una causa.

| # | Comprobá | Comando | Si falla |
|---|---|---|---|
| 1 | Tenés IP del laboratorio | `ip -4 addr show scope global` | Revisá cable o WiFi |
| 2 | Llegás al Jetson | `ping $JETSON` | Es la red. Avisale al profesor |
| 3 | Alguien levantó el driver | `ros2 node list` **dentro del Jetson** | Si allá también sale vacío, nadie lo levantó. No es tu Raspberry |
| 4 | Mismo dominio en las dos | `echo $ROS_DOMAIN_ID` en cada una | Iguálalos y volvé al paso 5 |
| 5 | `ROS_LOCALHOST_ONLY` en 0 | `echo $ROS_LOCALHOST_ONLY` | Con 1 no salís de tu máquina |
| 6 | Demonio reiniciado | `ros2 daemon stop && ros2 daemon start` | La lista vieja engaña |
| 7 | Cortafuegos abierto | `sudo ufw status` | Ver abajo |

El punto 3 es el que más confunde: una lista vacía **casi nunca** es un problema de red.
Lo más común es que el driver no esté corriendo.

### Si el cortafuegos está activo

ROS 2 usa puertos UDP que dependen del dominio: `7400 + 250 × dominio`. Con el dominio
**5**, del **8650** en adelante.

```bash
sudo ufw allow from 172.51.0.0/16 to any proto udp port 8650:8750
```

### Si todo lo anterior está bien y sigue sin verse

Preguntale al profesor si el laboratorio usa **Discovery Server**. En ese caso hacen falta
dos variables más y un archivo que se copia desde el Jetson:

```bash
sudo apt install -y ros-humble-rmw-fastrtps-cpp
scp jetson@$JETSON:~/super_client_configuration_file.xml ~/

export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
export ROS_DISCOVERY_SERVER=$JETSON:11811
export FASTRTPS_DEFAULT_PROFILES_FILE=$HOME/super_client_configuration_file.xml

ros2 daemon stop; ros2 daemon start
ros2 node list
```

El archivo XML es obligatorio en esta modalidad. Sin él vas a ver solo `/parameter_events`
y `/rosout`, que son tus propios tópicos, y ninguno del robot.

---

## Paso 8 · Armar el workspace y compilar

Todo el reto vive en **un solo workspace: `~/rb2_ws`**. Este es el único `source` que
necesitás en cada terminal; no hay kit ni scripts conectores.

Situate en la raíz de este repositorio (donde están las carpetas `src`, `analisis` y
`herramientas`) y copiá los dos paquetes:

```bash
mkdir -p ~/rb2_ws/src
cp -r src/* ~/rb2_ws/src/
```

Compilá. El paquete de interfaces tarda unos minutos: está generando código a partir de
las definiciones de la acción y del mensaje.

```bash
cd ~/rb2_ws
colcon build --symlink-install
source install/setup.bash
```

`--symlink-install` no es obligatorio, pero conviene: hace que los archivos Python de
`~/rb2_ws/src/arm_broker/arm_broker/` sean enlaces a los del repo, así que si tocás
`fk.py` o `politicas.py` no hace falta recompilar. Sin esa bandera, `colcon build` copia
los archivos y tenés que volver a compilar cada vez que los cambiás.

Comprobá que quedó:

```bash
ros2 pkg list | grep arm_broker
ros2 interface show arm_broker_interfaces/action/MoveArm
ros2 interface show arm_broker_interfaces/msg/QueueState
```

Y en cualquier terminal nueva, desde entonces y hasta el final de la sesión:

```bash
source ~/rb2_ws/install/setup.bash
```

---

## Paso 9 · Los dos modos del broker

Esta es la parte que cambia respecto del enunciado base, así que leé con atención.

### 9.1 Qué es cada modo

El broker tiene **exactamente dos modos de atención**, registrados en `POLITICAS`
(`src/arm_broker/arm_broker/politicas.py`):

| Clave que se pasa en `-p politica:=` | Clase | Qué hace |
|---|---|---|
| `fifo` | `FIFO` | Primero que llega, primero sale. `siguiente()` devuelve el índice `0` de la fila |
| `prioridad` | `SegundaPolitica` | **Round-robin entre clientes**: el turno pasa al cliente más antiguo que no sea el último servido |

**El nombre engaña a propósito.** La clave `prioridad` no implementa prioridad por número:
por dentro es round-robin. Se llama así porque el `broker.py` de este repo solo distingue
dos casos, y el segundo tiene que conservar exactamente ese nombre para que el broker le
pase el parámetro de envejecimiento. Si le decís `prioridad` a un nodo que espera
prioridad por número, la lectura de la clave es lo que hay que tener en cuenta; el
comportamiento real está en `SegundaPolitica.siguiente()`.

Por qué se eligió round-robin y no las otras dos opciones, y cómo se lee la métrica de
equidad, está en el docstring de `politicas.py`. En corto: genera equidad de Jain ≈ 1.0
por construcción, no depende de que los números de prioridad estén "bien elegidos", y no
degrada el p95 agregado por diferencias grandes entre clientes.

### 9.2 Cómo se lanza cada modo

En el **Jetson**, con el driver `sync_plan_nx` ya corriendo:

```bash
# Modo 1 — FIFO
ros2 run arm_broker broker --ros-args -p politica:=fifo

# Modo 2 — round-robin
ros2 run arm_broker broker --ros-args -p politica:=prioridad -p tau_envejecimiento_s:=8.0
```

El broker responde en el log cuál quedó:

```
arm_broker listo · política=fifo      · cola_max=20 · único publicador de /joint_states
arm_broker listo · política=prioridad · cola_max=20 · único publicador de /joint_states
```

Si esa línea no aparece, el broker no arrancó: mirá el error de arriba.

### 9.3 Los seis parámetros del broker

Todos se cambian por línea de comandos con `--ros-args -p`. Estos son los defaults tal
como están declarados:

| Parámetro | Default | Qué decide |
|---|---|---|
| `politica` | `fifo` | Qué modo de atención. Solo admite `fifo` o `prioridad` |
| `tau_envejecimiento_s` | `8.0` | Segundos de envejecimiento. **Solo se lee si `politica:=prioridad`**, y en round-robin se guarda pero no se usa |
| `cola_max` | `20` | Tope de pedidos esperando. Un goal que llega con la cola llena se rechaza con motivo `cola llena` |
| `paso_max_rad` | `1.2` | Salto articular máximo desde la pose actual. Más que eso se rechaza con motivo `paso articular excesivo` |
| `duracion_movimiento_s` | `3.0` | Cuánto tarda un movimiento completo |
| `pasos_interpolacion` | `10` | En cuántos pasos se interpola cada movimiento. Menos pasos = feedback menos fino y movimiento más a saltos |

Ejemplo con todo tocable a la vez:

```bash
ros2 run arm_broker broker --ros-args \
  -p politica:=prioridad -p tau_envejecimiento_s:=8.0 \
  -p cola_max:=20 -p paso_max_rad:=1.7 -p duracion_movimiento_s:=3.0
```

### 9.4 Tres trampas antes de medir

**La política no se cambia en caliente.** Se lee en el `__init__`, cuando se construye el
objeto. `ros2 param set /arm_broker politica round_robin` no sirve: el broker ni siquiera
declara ese parámetro y el `siguiente()` que se está usando no cambia. Para pasar de un
modo a otro hay que **terminar el broker (`Ctrl-C`) y relanzarlo** con el otro
`-p politica:=`.

**`tau` no hace nada en round-robin.** Se acepta el parámetro y se guarda en
`SegundaPolitica.__init__`, pero el algoritmo no consulta `self.tau` en ningún momento: no
hay envejecimiento activo. Pasar `-p tau_envejecimiento_s:=30` con `politica:=prioridad`
produce exactamente la misma corrida que con `8.0`. Y con `politica:=fifo` el parámetro se
acepta y se ignora, también sin error.

**No se puede agregar un tercer modo sin tocar el broker.** `POLITICAS` es un diccionario
libre, pero `broker.py` construye la política con una sola línea que decide entre "le paso
`tau`" y "no le paso nada" según si el nombre es exactamente `prioridad`. Si agregás una
clave nueva sin tocar esa línea, el broker levanta la clase **sin argumentos** y falla en
cuanto uses un `__init__` que los pida. Si vas a cambiar el modo, cambialo en las dos
partes y dejalo escrito.

### 9.5 La convención de prioridad

`MoveArm.action` declara `uint8 priority`, de **0 a 255, donde mayor número = más
urgente**. Consecuencias prácticas:

- En el modo `fifo` el número **no** influye en el orden: entra quien llegó primero.
- En el modo `prioridad` (round-robin) el número **tampoco** influye en el orden: lo que
  pesa es el `client_id` al que se le dio el turno la vez anterior.
- El número sí se **publica y se mide**: `QueueState.queued_priorities` lo lleva, y
  `analisis/metricas.py` agrupa las esperas por número de prioridad.
- Por eso el índice de inanición que calcula `metricas.py` es la espera máxima del
  **número más bajo**: es la espera del cliente al que menos urgencia se le dio.

Los clientes **no cambian** entre modos. El mismo comando sirve para las dos corridas; lo
único que cambia es el broker.

---

## Paso 10 · Correr el broker y los clientes

### 10.1 Generar la carga

Desde la raíz del repo (los scripts resuelven rutas relativas a sí mismos, así que hay
que correrlos desde acá y no desde `~/rb2_ws`):

```bash
python3 herramientas/generar_carga.py --n 40 --semilla 7 --salida carga.csv
```

Sale algo como `40 poses alcanzables en carga.csv (semilla 7)`.

**Misma semilla = mismo archivo.** Usá la misma carga para las dos corridas. Si cada
corrida usa poses distintas, las métricas no se pueden comparar y el ítem 3 no vale nada.
Regenerá `carga.csv` solo si cambiás `--semilla` o `--n`.

### 10.2 Terminal 1 — el broker, en el Jetson

```bash
ssh jetson@$JETSON
source ~/jetcobot_colcon_ws/install/setup.bash     # el workspace del driver
source ~/rb2_ws/install/setup.bash                 # y el del reto
ros2 run arm_broker broker --ros-args -p politica:=fifo 2>&1 | tee broker_fifo.log
```

El `tee` es importante: los rechazos con su motivo salen por el log del broker, y eso es
justo la evidencia que pide el ítem 2. Sin `tee`, se pierde al cerrar la terminal.

### 10.3 Terminales 2 a 5 — un cliente por integrante, desde cada Raspberry

Cada uno abre la suya, en la raíz del repo, y corre:

```bash
source ~/rb2_ws/install/setup.bash

ros2 run arm_broker cliente --ros-args \
  -p client_id:=ana -p priority:=1 -p traza:=carga.csv -p pausa_s:=0.5
```

Con cuatro integrantes, uno cada uno:

| `client_id` | `priority` | Quién |
|---|---|---|
| `ana` | 1 | integrante 1 |
| `bruno` | 2 | integrante 2 |
| `carla` | 3 | integrante 3 |
| `diego` | 4 | integrante 4 |

Los `client_id` **tienen que ser distintos**: es la clave sobre la que alterna el
round-robin. Si dos personas usan el mismo nombre, el modo `prioridad` los trata como el
mismo cliente.

Los números de prioridad distintos no cambian el orden en ninguno de los dos modos (ver
9.5), pero hacen que en las métricas el desglose por prioridad coincida con el desglose
por cliente, que es lo que se quiere ver en el cuadro.

Parámetros del cliente, por si los necesitás:

| Parámetro | Default | Para qué |
|---|---|---|
| `client_id` | `alumno` | Tu nombre en la cola. Distinto por cliente |
| `priority` | `1` | 0 a 255, mayor = más urgente |
| `traza` | *(vacío)* | CSV de poses. Vacío = dos poses fijas de prueba |
| `repeticiones` | `1` | Cuántas veces se repite todo el CSV |
| `pausa_s` | `0.5` | Espera entre pose y pose |

### 10.4 Ver la cola mientras corre

En cualquier terminal del equipo:

```bash
ros2 topic echo /arm/queue_state
```

Vas a ver `executing_client` (quién tiene el brazo), `executing_goal_id`,
`queue_length`, y los cuatro arreglos `queued_*` con lo que está esperando: los ids, los
clientes, las prioridades y los segundos de espera de cada uno.

**Qué esperar en la profundidad de la cola.** `cliente.py` espera el resultado antes de
mandar la pose siguiente, así que cada cliente tiene **como mucho un pedido en la cola**.
En la práctica `queue_length` se mueve entre 1 y 3, y no vas a ver una cola de 20. Eso es
normal con esta carga; si necesitás una cola más profunda para una demo, subí
`-p repeticiones` o mandad todos los clientes con `-p pausa_s:=0`.

### 10.5 Vas a ver rechazos, y es lo esperado

Con la carga aleatoria de `generar_carga.py` y el default `paso_max_rad:=1.2`, es
**normal** que algunos goals se rechacen: dos poses consecutivas pueden diferir más de
1.2 rad en alguna articulación. Cada rechazo sale con su motivo en `broker_fifo.log`:

```
RECHAZADO [diego p4] · paso articular excesivo: 1.43 rad desde la pose actual, maximo 1.20 rad
RECHAZADO [carla p3] · cola llena: 20 de 20 pedidos
```

Los tres motivos que valen para el ítem 2 son `limites articulares`, `workspace` y `paso
articular excesivo`; el cuarto, `cola llena`, es el que justifica el parámetro `cola_max`.

Los rechazos **no** invalidan la corrida: son parte de lo que hay que demostrar. Si
preferís una corrida limpia para la figura, subí el umbral solo para esa corrida:

```bash
ros2 run arm_broker broker --ros-args -p politica:=fifo -p paso_max_rad:=1.7
```

y anotá que lo cambiaste, porque con otros umbrales las corridas dejan de ser
comparables.

### 10.6 Cuánto dura

Cada pose tarda `duracion_movimiento_s=3.0` más `pausa_s=0.5`, o sea 3.5 s. Con 40 poses
y 4 clientes en paralelo, cada cliente termina en unos 2 minutos 20 y la corrida completa
en **3 a 4 minutos**. Si grabás el bag, empezá a grabar antes de levantar los clientes y
terminá de grabar cuando el último termine.

---

## Paso 11 · Medir: la evidencia del ítem 3

El broker tiene que estar **en el mismo modo** durante toda la corrida, y las dos
corridas tienen que usar **la misma `carga.csv`**.

### 11.1 Correr cada modo por separado

Repetí el paso 10 tres veces, cambiando solo dos cosas:

| Corrida | Broker | Carpeta del bag |
|---|---|---|
| 1 | `-p politica:=fifo` | `fifo` |
| 2 | `-p politica:=prioridad -p tau_envejecimiento_s:=8.0` | `prioridad` |

Entre una y otra: terminá los clientes, `Ctrl-C` el broker, y relanzalo con el otro modo.

### 11.2 Grabar

En una terminal del equipo, con el broker ya corriendo y antes de levantar los clientes:

```bash
cd <raíz del repo>
rm -rf fifo                      # si ya existe de una corrida anterior
ros2 bag record -o fifo /arm/queue_state /joint_states
```

Grabá **una sola vez**. El `bag` lo puede abrir cualquiera de las Raspberry, pero si dos
graban a la vez vas a terminar con dos carpetas y la mitad de los datos.

**Grabá siempre `/arm/queue_state`.** `/joint_states` dice qué poses pasaron, no quién
esperó cuánto. Sin ese tópico no hay métricas. Cuando el último cliente termine, `Ctrl-C`
al grabador para que cierre el bag.

### 11.3 Exportar a CSV

Con ROS sourced en esa terminal:

```bash
python3 analisis/exportar_csv.py fifo       --salida fifo/
python3 analisis/exportar_csv.py prioridad --salida prioridad/
```

Cada una escribe dos archivos: `queue_state.csv` (el que importa para las métricas) y
`joint_states.csv`. Si `queue_state.csv` tiene 0 filas, el `bag` no tiene
`/arm/queue_state` y hay que regrabar.

### 11.4 Métricas y figura

```bash
python3 analisis/metricas.py fifo/queue_state.csv prioridad/queue_state.csv \
  --salida comparacion_politicas.png
```

Imprime, por política: número de pedidos, completados, espera media, p95, máxima, el
desglose por prioridad, el índice de inanición, los goals por cliente y la equidad de
Jain. Y con los dos CSVs arma la figura de dos paneles.

**Los nombres de carpeta no son cosméticos.** `metricas.py` saca la etiqueta de cada
política del nombre de la carpeta que contiene el CSV. Si el bag de round-robin se llama
`rr`, la figura va a decir `rr`, no `prioridad`. Usá exactamente `fifo` y `prioridad`.

**Cómo leer la figura.** Con esta carga los cuatro clientes mandan la misma cantidad de
goals, así que la equidad de Jain da cerca de 1.0 en las dos políticas: el panel de
reparto no las va a diferenciar. La diferencia real está en el otro panel, la de esperas,
y en el índice de inanición. Eso es un resultado, no un fallo de la medición: el
round-robin no se distingue por repartir unequitativamente, sino por no dejar a nadie
esperando de más.

### 11.5 Lo que hay que poder sostener

Un bag de `/joint_states` no registra quién publicó cada mensaje, así que por sí solo no
demuestra que hubo publicadores concurrentes. Lo que sí lo demuestra es `/arm/queue_state`:
si al correlacionar las dos grabaciones hay **un único `executing_goal_id` en cada
instante**, la exclusión mutua se cumplió. Eso es un argumento que se puede dar con los
CSV en la mano, no con una intuición.

---

## Paso 12 · Verificar la cinemática directa

Este paso es físico: no se puede resolver sin el JetCobot delante. Y es al revés que los
demás — **corre en el Jetson, con el puerto serie libre**, o sea con `sync_plan_nx`
**detenido** y el broker parado.

```bash
ssh jetson@$JETSON
cd <ruta del repo en el Jetson>

python3 herramientas/verificar_fk.py                    # mueve el brazo a 4 poses
python3 herramientas/verificar_fk.py --solo-leer        # no mueve: solo compara donde está
```

Opciones: `--puerto /dev/ttyUSB0`, `--baud 1000000`, `--velocidad 30`, `--espera 5.0`.

La salida es una tabla de pose, posición calculada por `fk(q)`, posición que reporta el
brazo, y el error:

```
pose        FK (x,y,z)                robot (x,y,z)              error
cero        (  281.7,   0.0,  134.8)   (  280.4,   1.2,  136.0)    1.8 OK
...
error medio 3.2 mm   máximo 7.1 mm   criterio del reto: ≤ 10 mm
```

**Criterio del reto: error ≤ 10 mm.** Si no lo cumple, no lo arregles a ojo:

- Si el error es **grande y constante** en todas las poses, el problema son los
  `theta_offset` de la tabla `DH`.
- Si **crece con la distancia**, el problema son los `a_i`.

La tabla vive en `src/arm_broker/arm_broker/fk.py`, en el bloque `DH`. Está en convención
DH estándar, en radianes, en el orden `(alpha, a_mm, d_mm, theta_offset)`. Corregila ahí,
volvé a compilar (`cd ~/rb2_ws && colcon build --symlink-install`) y repetí la
verificación.

---

## Antes de cada sesión de trabajo

**En cada terminal nueva**, un `source`:

```bash
source ~/rb2_ws/install/setup.bash
```

El dominio y `JETSON` ya quedaron en el `~/.bashrc` del paso 4, no hay que volver a ponerlos.

Orden típico de una sesión:

```bash
# 1. En el Jetson, una persona
source ~/jetcobot_colcon_ws/install/setup.bash
ros2 run jetcobot_driver sync_plan_nx

# 2. En el Jetson, otra terminal
source ~/rb2_ws/install/setup.bash
ros2 run arm_broker broker --ros-args -p politica:=fifo 2>&1 | tee broker_fifo.log

# 3. En cada Raspberry, desde la raíz del repo
ros2 run arm_broker cliente --ros-args -p client_id:=ana -p priority:=1 -p traza:=carga.csv
```

---

## Reglas del laboratorio

**El brazo es un recurso compartido.** Un solo programa puede abrir su puerto. Antes de
levantar algo que lo use, confirmá con tu equipo que nadie más lo tiene. El puerto serie
lo tiene el driver; el broker publica, no lo abre.

**Si algo no funciona, no cambies permisos ni detengas servicios.** Los robots tienen
configuración puesta a propósito, y lo que parece un obstáculo puede ser una protección.
Levantá la mano.

**Avisá antes de mover el brazo.** En voz alta, y mirá que no haya manos cerca.

**No modifiques permisos ni services del Jetson para esquivar un error de red.** Casi
siempre es el dominio, el `ROS_LOCALHOST_ONLY` o el demonio.

---

## Resumen · cheat sheet

```bash
# Una sola vez
sudo apt install -y ros-humble-rosidl-default-generators ros-humble-rosbag2 \
  ros-humble-rosbag2-storage-default-plugins ros-humble-sensor-msgs \
  python3-colcon-common-extensions python3-matplotlib
echo 'source /opt/ros/humble/setup.bash' >> ~/.bashrc
echo 'export ROS_DOMAIN_ID=5'            >> ~/.bashrc
echo 'export ROS_LOCALHOST_ONLY=0'       >> ~/.bashrc
echo "export JETSON=$JETSON"             >> ~/.bashrc
exec bash
ros2 daemon stop; ros2 daemon start
mkdir -p ~/rb2_ws/src && cp -r src/* ~/rb2_ws/src/
cd ~/rb2_ws && colcon build --symlink-install

# En cada terminal
source ~/rb2_ws/install/setup.bash

# En el Jetson: driver y broker (uno por equipo)
source ~/jetcobot_colcon_ws/install/setup.bash   # solo en la terminal del driver
ros2 run jetcobot_driver sync_plan_nx
ros2 run arm_broker broker --ros-args -p politica:=fifo       # modo 1
ros2 run arm_broker broker --ros-args -p politica:=prioridad  # modo 2 (round-robin)

# En cada Raspberry, desde la raíz del repo
python3 herramientas/generar_carga.py --n 40 --semilla 7 --salida carga.csv
ros2 run arm_broker cliente --ros-args -p client_id:=ana -p priority:=1 -p traza:=carga.csv

# Grabar y medir
ros2 bag record -o fifo /arm/queue_state /joint_states
python3 analisis/exportar_csv.py fifo --salida fifo/
python3 analisis/metricas.py fifo/queue_state.csv prioridad/queue_state.csv

# Verificar la cinemática (Jetson, puerto libre)
python3 herramientas/verificar_fk.py
```

---

## Índice de troubleshooting

| Síntoma | Causa probable | Qué hacer |
|---|---|---|
| `política desconocida: X` | Nombre mal escrito | Solo `fifo` o `prioridad`. Ver 9.4 |
| Cambiaste `politica` y nada pasó | Se lee en el `__init__` | `Ctrl-C` y relanzá el broker. Ver 9.4 |
| `tau_envejecimiento_s` no hace nada | Es round-robin, no hay envejecimiento | Esperado. Ver 9.4 |
| El broker levanta y los clientes dicen "El broker no aparece" | Faltan 15 s o están en dominio distinto | `echo $ROS_DOMAIN_ID` en las dos máquinas |
| Cola siempre de 1 a 3 | `cliente.py` espera el resultado antes de seguir | Esperado. Ver 10.4 |
| Muchos rechazos de "paso articular excesivo" | `paso_max_rad=1.2` con poses aleatorias | Esperado, es evidencia. Ver 10.5 |
| `queue_state.csv` con 0 filas | El bag no tiene `/arm/queue_state` | Regrabá. Ver 11.3 |
| La figura dice `rr` en vez de `prioridad` | El nombre de la carpeta | Renombrá la carpeta a `prioridad` |
| `ModuleNotFoundError: rosbag2_py` | Terminal sin ROS | `source /opt/ros/humble/setup.bash` |
| `ModuleNotFoundError: arm_broker` en los scripts | Corriste el script desde otra carpeta | Correlos desde la raíz del repo |
| Error de FK grande y constante | Offsets de la tabla `DH` | Ajustar `theta_offset`. Ver paso 12 |
| `ros2 node list` vacío | El driver no está corriendo | Paso 6, punto 3 de la tabla |
