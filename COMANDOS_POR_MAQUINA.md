# Todos los comandos, por máquina

Hoja de referencia con **todos** los comandos del Reto 2, separados por máquina.
Cada bloque dice en qué máquina va, qué hacer, y qué tiene que verse.

Este documento **no explica los porqué**: para eso están
[`DONDE_VA_CADA_ARCHIVO.md`](DONDE_VA_CADA_ARCHIVO.md),
[`ROS2_DESDE_CERO_Y_PROCEDIMIENTO.md`](ROS2_DESDE_CERO_Y_PROCEDIMIENTO.md) y
[`COMO_HACER_ITEMS_2_Y_3.md`](COMO_HACER_ITEMS_2_Y_3.md). Acá van solo comandos.

---

## Índice

| # | Bloque | Máquina | Cuándo |
|---|---|---|---|
| 0 | [Placeholders](#0--placeholders-que-tienes-que-reemplazar) | — | Antes de empezar |
| 1 | [Instalación, una sola vez](#1--instalación-una-sola-vez-en-las-5-máquinas) | Las 5 | Una vez |
| 2 | [Extra del Jetson](#2--extra-del-jetson-driver-y-paquetes-del-ítem-3) | Jetson | Una vez |
| 3 | [Verificación](#3--verificación-de-la-máquina) | Las 5 | Después del bloque 1 |
| 4 | [Red](#4--red-y-cortafuegos) | Las 5 | Una vez |
| 5 | [`carga.csv`](#5--cargacsv-generar-y-repartir) | Jetson + 4 Pi | Antes del ítem 3 |
| 6 | [Actualizar el código](#6--actualizar-el-código-cuando-cambia-algo) | Las 5 | Después de cada `git pull` |
| 7 | [Probar un cliente sin broker](#7--probar-un-cliente-sin-broker-la-prueba-de-fuego) | 4 Pi | Antes de la corrida |
| 8 | [Correr el reto](#8--correr-el-reto-6-terminales) | Las 5 | Ítem 2 y 3 |
| 9 | [Probar los tres rechazos](#9--probar-los-tres-rechazos) | Jetson | Ítem 2 |
| 10 | [Analizar los resultados](#10--analizar-los-resultados-del-ítem-3) | Jetson | Ítem 3 |
| 11 | [Diagnóstico](#11--diagnóstico-rápido) | Las 5 | Cuando algo falla |
| 12 | [Cheat sheet](#12--cheat-sheet-completo) | — | Consulta |

---

## 0 · Placeholders que tienes que reemplazar

| Texto | Qué es | Cómo lo obtienes |
|---|---|---|
| `TU_USUARIO_PI` | Tu usuario en la Raspberry | `whoami` en la Pi |
| `IP_JETSON` | La IP del Jetson | En el Jetson: `hostname -I` |
| `TU_USUARIO_GIT` | Tu usuario de GitHub | Es el de la URL del repo |
| `DRIVER_WS` | La carpeta del driver | Bloque 1, paso 1 |

**Las 5 máquinas** = 1 Jetson + 4 Raspberry. Lo que va en el Jetson va en el
Jetson; lo que va en "las 4 Pi" se repite en cada Raspberry.

---

## 1 · Instalación, una sola vez, en las 5 máquinas

> **En:** Jetson, y luego Raspberry 1, 2, 3 y 4. Idéntico en las cinco.

### Paso 1 · Detectar la carpeta del driver

```bash
ls -d ~/ros2_ws_brazo ~/jetcobot_colcon_ws 2>/dev/null
```

Debe salir **exactamente uno** de los dos. Anotalo, es tu `DRIVER_WS`.

Si no sale ninguno:

```bash
ls -d ~/ros2_ws* ~/jetcobot* 2>/dev/null
```

### Paso 2 · Clonar el repo

```bash
cd ~
rm -rf rb2 rb2_ws
git clone https://github.com/TU_USUARIO_GIT/rb2-turno-del-brazo.git rb2
cd rb2
ls
```

**Qué debe verse:** `README.md`, `src`, `herramientas`, `analisis`, `docs`,
`CAMBIOS.md`.

> **Cuidado con el `rm -rf rb2 rb2_ws`:** borra el workspace compilado. Solo
> dejalo en la primera instalación. Si ya tenés cosas adentro y solo querés
> actualizar, saltá al [bloque 6](#6--actualizar-el-código-cuando-cambia-algo).

### Paso 3 · Copiar `src/` al workspace

```bash
mkdir -p ~/rb2_ws/src
cp -r src/* ~/rb2_ws/src/
ls ~/rb2_ws/src
```

**Qué debe verse:** `arm_broker` y `arm_broker_interfaces`. Si solo ves uno, el
`src` está incompleto.

### Paso 4 · Compilar

```bash
cd ~/rb2_ws && colcon build
```

Tarda entre 2 y 15 minutos en el Jetson; unos segundos en las Pi.

**Qué debe verse:**

```
Starting >>> arm_broker_interfaces
Finished <<< arm_broker_interfaces [12.4s]
Finished <<< arm_broker [0.8s]
Summary: 2 packages finished
```

Si ves `Failed <<<`:

```bash
cat ~/rb2_ws/log/latest_build/*/stderr.log | tail -40
```

### Paso 5 · Confirmar que compilaste TU código

```bash
wc -c ~/rb2_ws/src/arm_broker/arm_broker/broker.py
```

**Debe decir `16242`.** Si dice `8003`, clonaste el andamiaje del curso y no tu
solución. Volvé al paso 2.

### Paso 6 · Activar

```bash
source /opt/ros/humble/setup.bash
source ~/rb2_ws/install/setup.bash
ros2 pkg list | grep arm_broker
```

**Debe verse:** `arm_broker` y `arm_broker_interfaces`.

### Paso 7 · Dejar el `source` en el `.bashrc`

Ahora que el workspace existe, el `source` no tira error. Escribilo una vez:

```bash
grep -q 'ROS_DOMAIN_ID' ~/.bashrc || echo 'export ROS_DOMAIN_ID=47' >> ~/.bashrc
grep -q 'ROS_LOCALHOST_ONLY' ~/.bashrc || echo 'export ROS_LOCALHOST_ONLY=0' >> ~/.bashrc
grep -q 'RMW_IMPLEMENTATION' ~/.bashrc || echo 'export RMW_IMPLEMENTATION=rmw_fastrtps_cpp' >> ~/.bashrc
grep -q 'JETSON=' ~/.bashrc || echo 'export JETSON=IP_JETSON' >> ~/.bashrc
grep -q 'rb2_ws' ~/.bashrc || echo 'source ~/rb2_ws/install/setup.bash' >> ~/.bashrc
exec bash
```

> Los `grep -q ... ||` hacen que el bloque sea **idempotente**: podés pegarlo las
> veces que quieras sin duplicar líneas. Cambiale el `47` por el dominio que te
> haya dado el profe si no es el grupo 5, y `IP_JETSON` por la IP real.

### Paso 8 · Reiniciar el índice de ROS

```bash
ros2 daemon stop
ros2 daemon start
```

**Cada vez que cambies una variable de entorno.** Sin esto, ROS te sigue
mostrando la configuración vieja.

---

## 2 · Extra del Jetson: driver y paquetes del ítem 3

> **En:** solo el Jetson. **Nunca** en una Raspberry.

```bash
# --- Agregar el driver al bashrc ---
DRIVER=$(ls -d ~/ros2_ws_brazo ~/jetcobot_colcon_ws 2>/dev/null | head -1)
echo "Driver ws: $DRIVER"
grep -q 'DRIVER_WS' ~/.bashrc || echo "source $DRIVER/install/setup.bash" >> ~/.bashrc

# --- Activar todo y reiniciar el índice ---
source /opt/ros/humble/setup.bash
source $DRIVER/install/setup.bash
source ~/rb2_ws/install/setup.bash
ros2 daemon stop; ros2 daemon start

# --- Paquetes que hacen falta para medir el ítem 3 ---
sudo apt install -y ros-humble-rosbag2 ros-humble-rosbag2-storage-default-plugins python3-matplotlib
```

**Verificar:**

```bash
ros2 pkg list | grep jetcobot
```

Debe aparecer algo con `jetcobot`.

> **Nunca corras `colcon build` en `DRIVER_WS`.** Ese workspace es del curso y ya
> está compilado. Si lo recompilás y algo sale mal, rompiste el driver.

---

## 3 · Verificación de la máquina

> **En:** las 5. Pegalo tal cual y fijate que las 4 líneas salgan bien.

```bash
echo "dominio=$ROS_DOMAIN_ID  localhost=$ROS_LOCALHOST_ONLY  distro=$ROS_DISTRO"
ros2 pkg list | grep arm_broker
ros2 interface show arm_broker_interfaces/action/MoveArm | head -6
wc -c ~/rb2_ws/src/arm_broker/arm_broker/broker.py
ping -c 2 $JETSON
```

**Lo esperado:**

```
dominio=47  localhost=0  distro=humble
arm_broker
arm_broker_interfaces
float64[] joint_positions
string    client_id
uint8     priority
16242
--- 172.51.9.5 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss
```

| Si falla esto | Causa |
|---|---|
| `dominio=` vacío | No pegaste el paso 7 del bloque 1 |
| `pkg list` vacío | No compilaste, o falta el `source` |
| `interface show` vacío | No compilaste `arm_broker_interfaces` |
| `16242` no aparece | Clonaste el andamiaje |
| Ping sin respuesta | **Es la red.** Avisá al profe; nada de ROS va a funcionar |

---

## 4 · Red y cortafuegos

> **En:** las 5. Una sola vez.

```bash
# --- Estado del cortafuegos ---
sudo ufw status

# --- Si dice "Status: active", abrir el rango del dominio ---
sudo ufw allow from 172.51.0.0/16 to any proto udp port 19150:19250

# --- Verificar ---
sudo ufw status
```

El rango `19150:19250` corresponde a `ROS_DOMAIN_ID=47`. La fórmula es
`7400 + 250 × dominio`. Si tu dominio es otro, recalculá: `7400 + 250 × X`.

Si el laboratorio usa **Discovery Server**, en vez de lo anterior:

```bash
sudo apt install -y ros-humble-rmw-fastrtps-cpp
scp jetson@$JETSON:~/super_client_configuration_file.xml ~/
grep -q 'ROS_DISCOVERY_SERVER' ~/.bashrc || echo "export ROS_DISCOVERY_SERVER='$JETSON':11811" >> ~/.bashrc
grep -q 'FASTRTPS' ~/.bashrc || echo 'export FASTRTPS_DEFAULT_PROFILES_FILE=$HOME/super_client_configuration_file.xml' >> ~/.bashrc
exec bash
ros2 daemon stop; ros2 daemon start
```

Preguntá al profe antes de hacer esto. El archivo XML es obligatorio en esa
modalidad.

---

## 5 · `carga.csv`: generar y repartir

### 5.1 · Generar (solo Jetson)

```bash
cd ~/rb2
source ~/rb2_ws/install/setup.bash
python3 herramientas/generar_carga.py --n 40 --semilla 7 --salida carga.csv
wc -l carga.csv
md5sum carga.csv
```

**Qué debe verse:**

```
40 poses alcanzables en carga.csv (semilla 7)
40 carga.csv
<hash>  carga.csv
```

Anotá ese hash. Lo vas a comparar contra el de las 4 Pi.

> **Tiene que correr desde `~/rb2`.** El script busca tu código con una ruta
> relativa a su propia ubicación (`../src/arm_broker`). Desde otro lado no
> encuentra `fk.py`.

### 5.2 · Subirlo al repo (solo Jetson)

```bash
cd ~/rb2
git add carga.csv
git commit -m "carga oficial: 40 poses, semilla 7"
git push
```

> **No pongas `carga.csv` en el `.gitignore`.** Las 4 Pi la necesitan y esta es
> la forma limpia de llevarles.

### 5.3 · Bajarlo (cada Raspberry)

```bash
cd ~/rb2 && git pull
wc -l carga.csv
md5sum carga.csv
```

**Los cuatro hashes tienen que ser idénticos al del Jetson.** Si no coinciden,
las dos corridas no son comparables.

---

## 6 · Actualizar el código cuando cambia algo

> **En:** las 5. Después de cada `git pull`.

```bash
cd ~/rb2 && git pull
cp -r ~/rb2/src/* ~/rb2_ws/src/
cd ~/rb2_ws && colcon build
source ~/rb2_ws/install/setup.bash
ros2 daemon stop; ros2 daemon start
wc -c ~/rb2_ws/src/arm_broker/arm_broker/broker.py
```

**El `cp` y el `colcon build` no son opcionales.** `git pull` actualiza el repo,
pero el broker y el cliente se ejecutan desde `~/rb2_ws/`, que es una copia. Sin
recompilar seguís corriendo el código viejo, y como el broker arranca igual,
nada te avisa.

Si tocaste algún `.py` y querés iterar rápido, en el bloque 4 usá en su lugar:

```bash
colcon build --symlink-install
```

Los `.py` quedan enlazados y los cambios se ven sin recompilar. El precio: si
borrás un archivo, el enlace se rompe y hay que volver a compilar.

---

## 7 · Probar un cliente sin broker (la prueba de fuego)

> **En:** cada Raspberry, **antes** de la corrida. No necesita el broker ni la
> red andando.

```bash
source ~/rb2_ws/install/setup.bash
ros2 run arm_broker cliente --ros-args -p client_id:=prueba
```

**Qué debe verse:** el cliente busca la acción durante 15 segundos, no la
encuentra porque el broker todavía no está arriba, y se sale:

```
[prueba] esperando al broker...
[prueba] El broker no aparece. ¿Está corriendo?
```

**Eso es el resultado correcto.** Demuestra que el paquete se instaló bien en esa
máquina.

| Lo que ves | Qué significa | Qué hacer |
|---|---|---|
| `esperando al broker...` y después `El broker no aparece` | **Bien.** La Pi funciona | Nada |
| `ModuleNotFoundError: No module named 'arm_broker_interfaces'` | No compilaste las interfaces en la Pi | `cd ~/rb2_ws && colcon build` |
| `ModuleNotFoundError: No module named 'arm_broker'` | No compilaste, o falta el `source` | Bloque 1, pasos 4 y 6 |
| `ros2 run arm_broker cliente: command not found` | Falta `setup.py` en el workspace | `cp -r ~/rb2/src/* ~/rb2_ws/src/` y recompilar |
| Imprime `pose 0` y `pose 1`, y termina | **Le falta `carga.csv`** | Bloque 5.3 |

---

## 8 · Correr el reto: 6 terminales

Abrí 6 terminales. En el Jetson, con SSH múltiple o con `tmux`:

```bash
sudo apt install -y tmux
tmux new -s rb2
#   Ctrl+B  luego  %   -> divide verticalmente
#   Ctrl+B  luego  "   -> divide horizontalmente
#   Ctrl+B  luego  flecha -> te movés entre ventanas
```

Si usás SSH desde Windows, abrí 4 sesiones `ssh jetson@IP_JETSON` en 4 pestañas.

| Terminal | Máquina | Qué corre |
|---|---|---|
| T1 | Jetson | El driver |
| T2 | Jetson | El broker, con log de rechazos |
| T3 | Jetson | La telemetría |
| T4 | Jetson | La grabadora (ítem 3) |
| T5, T6, T7, T8 | Una Pi cada una | Un cliente cada una |

### T1 · Jetson · el driver

```bash
source /opt/ros/humble/setup.bash
source ~/DRIVER_WS/install/setup.bash
ros2 run jetcobot_driver sync_plan_nx
```

No imprime casi nada. **Eso es correcto**: está esperando órdenes. Se queda
corriendo y no vuelve al prompt.

> **Una sola persona levanta el driver.** El puerto serie es único.

### T2 · Jetson · el broker

```bash
source ~/rb2_ws/install/setup.bash
ros2 run arm_broker broker --ros-args -p politica:=fifo 2>&1 | tee ~/rb2/rechazos.log
```

**Qué debe verse:**

```
[INFO] [arm_broker]: arm_broker listo · política=fifo · cola_max=20 · único publicador de /joint_states
```

El `tee` guarda los rechazos con su motivo. **Es el entregable del ítem 2.**

### T3 · Jetson · la telemetría

```bash
source ~/rb2_ws/install/setup.bash
ros2 topic echo /arm/queue_state
```

**Al arrancar:**

```
stamp: ...
executing_client: ''
executing_goal_id: ''
queue_length: 0
total_accepted: 0
total_rejected: 0
total_completed: 0
```

Vacía al principio, y está bien.

### T4 · Jetson · la grabadora (solo ítem 3)

```bash
source ~/rb2_ws/install/setup.bash
ros2 bag record -o ~/rb2/corrida_fifo /arm/queue_state /joint_states
```

**Grabá los dos tópicos.** Grabando solo `/joint_states` no hay métricas.

### T5, T6, T7, T8 · Cada Pi · un cliente

Raspberry 1:
```bash
source ~/rb2_ws/install/setup.bash
ros2 run arm_broker cliente --ros-args \
  -p client_id:=uno \
  -p priority:=3 \
  -p traza:=/home/TU_USUARIO_PI/rb2/carga.csv
```

Raspberry 2: mismo comando con `client_id:=dos`.
Raspberry 3: `client_id:=tres`.
Raspberry 4: `client_id:=cuatro`.

**Ponele `priority` distinto a cada uno** si querés que la prioridad se note.

> **El `~` no se expande** dentro de `-p traza:=~/rb2/carga.csv`. Usá la ruta
> absoluta, como está arriba.

**Qué debe verse en cada Pi:**

```
[uno] esperando al broker...
[uno] QUEUED pos=2 t=0.3s
[uno] pose 0: success=True espera=4.2s ejec=3.0s total=7.4s — pose alcanzada en 10 pasos
```

El cliente tiene que imprimir `pose 0` hasta `pose 39`. Si el último es
`pose 1`, le falta `carga.csv`.

**Lanzá los cuatro a la vez**, dentro de unos segundos. Esa simultaneidad es lo
que genera la contención que mide el ítem 3.

### Antes de cada corrida: los dos chequeos

En T3, cortá el `echo` y corré:

```bash
ros2 topic info /joint_states -v
```

`Publisher count` **debe ser 1**. Más de uno = 5 puntos perdidos.

Y con el `echo` corriendo, mirá que `executing_goal_id` **nunca** aparezca
también dentro de `queued_goal_ids`.

---

## 9 · Probar los tres rechazos

> **En:** Jetson, T3. Con el broker recién arrancado y el brazo en cero.

Cortá el `echo` con `Ctrl+C` y mandá los tres:

**a) Fuera de límites articulares**
```bash
ros2 action send_goal /move_arm arm_broker_interfaces/action/MoveArm \
  "{joint_positions: [3.5, 0.0, 0.0, 0.0, 0.0, 0.0], client_id: 'prueba', priority: 1}"
```

En T2 (`rechazos.log`):
```
RECHAZADO [prueba p1] · limites articulares: 1_Joint fuera de rango: 3.500 rad, límite [-2.93, 2.93]
```

**b) Paso articular excesivo**
```bash
ros2 action send_goal /move_arm arm_broker_interfaces/action/MoveArm \
  "{joint_positions: [1.3, 0.0, 0.0, 0.0, 0.0, 0.0], client_id: 'prueba', priority: 1}"
```
```
RECHAZADO [prueba p1] · paso articular excesivo: 1.30 rad desde la pose actual, maximo 1.20 rad
```

**c) Fuera del workspace**
```bash
ros2 action send_goal /move_arm arm_broker_interfaces/action/MoveArm \
  "{joint_positions: [0.0, 1.5, 2.4, 0.0, 0.0, 0.0], client_id: 'prueba', priority: 1}"
```
```
RECHAZADO [prueba p1] · workspace: efector a 71 mm de la base, demasiado cerca
```

> Si no sale ese texto, el brazo ya se movió. Reiniciá el broker (Ctrl+C y
> relanzalo) y probá otra vez.

---

## 10 · Analizar los resultados del ítem 3

> **En:** Jetson.

### 10.1 · Cortar todo

`Ctrl+C` en T4 (la grabadora), después en T2 (el broker) y en T1 (el driver).

### 10.2 · Exportar los bags a CSV

```bash
cd ~/rb2
source ~/rb2_ws/install/setup.bash
python3 analisis/exportar_csv.py corrida_fifo --salida fifo/
python3 analisis/exportar_csv.py corrida_rr  --salida roundrobin/
```

**Qué debe verse, para cada una:**

```
Tópicos en el bag:
  /arm/queue_state  (arm_broker_interfaces/msg/QueueState)
  /joint_states  (sensor_msgs/msg/JointState)

queue_state.csv   2400 filas
joint_states.csv  1600 filas
```

> **Usá `--salida roundrobin/`, no `prioridad/`.** El nombre de la carpeta es la
> etiqueta que aparece en la figura. Si decís `prioridad/`, el gráfico va a decir
> "prioridad" y nadie va a saber que mediste round-robin.

### 10.3 · Métricas y figura

```bash
python3 analisis/metricas.py fifo/queue_state.csv roundrobin/queue_state.csv
```

**Qué debe verse:**

```
=== fifo
  pedidos observados : 160
  completados        : 158
  espera media       : 7.45 s
  espera p95         : 11.20 s
  espera máxima      : 12.90 s
  por prioridad:
    prioridad 0: n=40  media= 9.10s  p95=12.90s  máx=12.90s
  índice de inanición: 12.90 s
  goals por cliente  : {'cuatro': 40, 'dos': 40, 'tres': 40, 'uno': 40}
  equidad de Jain    : 1.000

Figura: comparacion_politicas.png
```

Los números son un ejemplo de formato. La estructura es lo que tiene que
coincidir: 160 pedidos, 4 prioridades, el índice de inanición, los goals por
cliente y el Jain.

### 10.4 · La segunda corrida

**Idéntica, cambiando una sola cosa:**

```bash
# T2
ros2 run arm_broker broker --ros-args -p politica:=prioridad 2>&1 | tee ~/rb2/rechazos_rr.log

# T4
ros2 bag record -o ~/rb2/corrida_rr /arm/queue_state /joint_states
```

Los cuatro clientes quedan **exactamente iguales**, con la misma `carga.csv`.

> `politica:=prioridad` es el nombre de la clave en `politicas.py`, y lo que
> implementa es **round-robin**. No es un error: es lo que está escrito.

### 10.5 · Subir los resultados

```bash
cd ~/rb2
git add carga.csv rechazos.log comparacion_politicas.png fifo/ roundrobin/
git commit -m "Resultados del item 3: dos corridas, CSV, figura comparativa"
git push
```

Subí los CSV (son pocos KB). **Los bags no**, pesan mucho.

---

## 11 · Diagnóstico rápido

| Síntoma | Causa | Solución |
|---|---|---|
| `ros2 node list` vacío | Descubrimiento, o el driver no está arriba | `ros2 daemon stop; ros2 daemon start`. Si sigue vacío, el driver no está corriendo: pregunta en el Jetson |
| `ros2 run arm_broker broker: command not found` | Falta el `source` | `source ~/rb2_ws/install/setup.bash` |
| `ros2 run jetcobot_driver ...: not found` | Falta sourcear el driver | `source ~/DRIVER_WS/install/setup.bash` |
| `Failed <<<` en el build | Error de compilación | `cat ~/rb2_ws/log/latest_build/*/stderr.log \| tail -40` |
| `ModuleNotFoundError: arm_broker_interfaces` | Compilaste solo `arm_broker` | `rm -rf ~/rb2_ws/build ~/rb2_ws/install && cd ~/rb2_ws && colcon build` |
| `El broker no aparece` | El broker no arrancó, o no sourceás el mismo workspace en las dos máquinas | Revisá T2 |
| `Publisher count: 2` | **Alguien más publica en `/joint_states`** | Buscá al culpable: driver duplicado, un `ros2 topic pub`, un test. **5 puntos** |
| `Permission denied` en `/dev/ttyUSB0` | El driver ya tiene el puerto | Solo el driver toca el brazo |
| Modifiqué un `.py` y no cambia | No recompilaste | Bloque 6 |
| El cliente imprime solo `pose 0` y `pose 1` | Le falta `carga.csv` | Bloque 5.3 |
| No veo a los otros integrantes | `ROS_DOMAIN_ID` distinto | `echo $ROS_DOMAIN_ID` en las 5 y igualá a 47 |
| Ping funciona pero no hay nodos | Cortafuegos | Bloque 4 |
| Ping no funciona | **Es la red** | Avisá al profe |

### Limpieza de último recurso

```bash
# En el Jetson y en cada Pi
rm -rf ~/rb2_ws/build ~/rb2_ws/install ~/rb2_ws/log
cd ~/rb2_ws && colcon build
source install/setup.bash
ros2 daemon stop
rm -rf ~/.ros
ros2 daemon start
```

`~/.ros` es la caché de descubrimiento de ROS 2. Borrarla es seguro.

---

## 12 · Cheat sheet completo

### Instalar (una vez)

| Comando | Máquina |
|---|---|
| `ls -d ~/ros2_ws_brazo ~/jetcobot_colcon_ws 2>/dev/null` | Jetson |
| `cd ~ && git clone <url> rb2` | Las 5 |
| `mkdir -p ~/rb2_ws/src && cp -r src/* ~/rb2_ws/src/` | Las 5 |
| `cd ~/rb2_ws && colcon build` | Las 5 |
| `source /opt/ros/humble/setup.bash` | Las 5 |
| `source ~/rb2_ws/install/setup.bash` | Las 5 |
| `source ~/DRIVER_WS/install/setup.bash` | **Solo Jetson** |
| `ros2 daemon stop; ros2 daemon start` | Las 5 |
| `sudo ufw allow from 172.51.0.0/16 to any proto udp port 19150:19250` | Las 5 |
| `sudo apt install -y ros-humble-rosbag2 python3-matplotlib tmux` | Jetson |

### Correr

| Comando | Máquina |
|---|---|
| `ros2 run jetcobot_driver sync_plan_nx` | Jetson |
| `ros2 run arm_broker broker --ros-args -p politica:=fifo` | Jetson |
| `ros2 run arm_broker cliente --ros-args -p client_id:=uno -p traza:=/ruta/carga.csv` | Cada Pi |
| `ros2 bag record -o ~/rb2/corrida_fifo /arm/queue_state /joint_states` | Jetson |

### Mirar

| Comando | Qué te dice |
|---|---|
| `ros2 node list` | Qué nodos ve esta máquina |
| `ros2 topic list` | Qué tópicos hay |
| `ros2 topic echo /arm/queue_state` | La fila, en tiempo real |
| `ros2 topic info /joint_states -v` | **Cuántos publican. Debe ser 1** |
| `ros2 action list` | Debe estar `/move_arm` |
| `ros2 action info /move_arm` | Detalle de la acción |
| `ros2 param list /arm_broker` | Los parámetros del broker |
| `ros2 interface show arm_broker_interfaces/action/MoveArm` | La definición de la acción |

### Probar

| Comando | Qué hace |
|---|---|
| `ros2 action send_goal /move_arm arm_broker_interfaces/action/MoveArm "{...}"` | Manda un goal a mano |

### Medir

| Comando | Dónde |
|---|---|
| `python3 herramientas/generar_carga.py --n 40 --semilla 7 --salida carga.csv` | Jetson, desde `~/rb2` |
| `python3 herramientas/verificar_fk.py` | Jetson, driver apagado |
| `python3 analisis/exportar_csv.py <bag> --salida <carpeta>` | Jetson, desde `~/rb2` |
| `python3 analisis/metricas.py a/queue_state.csv b/queue_state.csv` | Jetson, desde `~/rb2` |

### Comprobar

| Comando | Qué verifica |
|---|---|
| `wc -c ~/rb2_ws/src/arm_broker/arm_broker/broker.py` | Que tenés tu código: **16242** |
| `wc -l carga.csv` | Que la carga tiene **40** líneas |
| `md5sum carga.csv` | Que las 5 copias son **idénticas** |
| `echo $ROS_DOMAIN_ID` | Que las 5 están en el mismo dominio |
