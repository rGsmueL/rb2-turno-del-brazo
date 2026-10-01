# Dónde va cada archivo: Jetson y Raspberry

Este documento responde una sola pregunta, que es la que más confunde al empezar:

> ¿Copio el `src` en el Jetson o en las Raspberry de los clientes?

**La respuesta: en los dos. En las cinco máquinas.**

No es una recomendación, es una consecuencia de cómo está construido el código.
Y lo importante es que la distribución real de archivos **no es** "el código va a
un lado y los datos al otro". El código va **completo a todas partes**. Lo único
que se reparte son los datos y los resultados.

---

## La respuesta, en una tabla

| Carpeta o archivo | Jetson | Las 4 Raspberry | Razón |
|---|:---:|:---:|---|
| `src/arm_broker/` completo | Sí | Sí | `broker` y `cliente` son dos ejecutables del **mismo** paquete |
| `src/arm_broker_interfaces/` completo | Sí | Sí | Genera la acción que el cliente importa. Hay que compilarlo en cada máquina |
| `herramientas/generar_carga.py` | Sí | No | Genera `carga.csv`. Se corre una vez |
| `herramientas/verificar_fk.py` | Sí | No | Necesita `/dev/ttyUSB0` y `pymycobot`. Solo el Jetson tiene el brazo |
| `analisis/exportar_csv.py` | Sí | No | Necesita `rosbag2_py` y los bags, que se graban en el Jetson |
| `analisis/metricas.py` | Sí | No | Necesita `matplotlib`. Genera la figura del ítem 3 |
| `carga.csv` | Sí (el original) | Sí (una copia) | El cliente la abre con `open()` en su propia máquina |
| `rechazos.log` | Sí | No | Lo escribe el broker, que corre en el Jetson |
| `corrida_fifo/`, `corrida_rr/` (bags) | Sí | No | Los graba el Jetson |
| `fifo/`, `roundrobin/` (CSV) | Sí | No | Salen de los bags |
| `comparacion_politicas.png` | Sí | No | Es el entregable del ítem 3 |
| `*.md`, `docs/`, `CAMBIOS.md` | Opcional | Opcional | No se ejecutan. Solo sirven para leer |
| Workspace del driver (`ros2_ws_brazo` o `jetcobot_colcon_ws`) | Sí | **No** | Es del curso. Es el único que abre el puerto serie del brazo |

Las cinco máquinas ejecutan estas dos líneas, sin excepción:

```bash
cp -r src/* ~/rb2_ws/src/
cd ~/rb2_ws && colcon build
```

---

## 1 · Por qué en las dos: la prueba en el código

No es una costumbre, es cómo quedó armado el paquete. Mirá estas líneas de
`src/arm_broker/setup.py`, líneas 19 a 23:

```python
entry_points={
    'console_scripts': [
        'broker = arm_broker.broker:main',
        'cliente = arm_broker.cliente:main',
    ],
},
```

Leelo despacio. Los dos ejecutables que necesitas, `broker` y `cliente`, están
declarados **dentro del mismo paquete**, que se llama `arm_broker`. No hay un
"paquete del broker" y un "paquete del cliente". Hay uno solo, con dos comandos
dentro.

Y el paquete se instala **entero**:

```python
packages=find_packages(exclude=['test']),
```

`find_packages` encuentra el directorio `arm_broker/` y se lo lleva completo. No
hay forma de instalar "la mitad del paquete". O instalas `arm_broker` con sus
cuatro módulos (`broker.py`, `cliente.py`, `fk.py`, `politicas.py`), o no
instalas nada y `ros2 run arm_broker cliente` responde `command not found`.

### La segunda mitad: la acción se genera localmente

Con lo anterior ya basta para justificar la Raspberry. Pero hay una segunda
razón, y es la que se lleva la culpa cuando falla.

`src/arm_broker/arm_broker/cliente.py`, línea 14:

```python
from arm_broker_interfaces.action import MoveArm
```

Ese `MoveArm` **no es un archivo que exista en el repo**. Es código que genera
`colcon build` a partir de la declaración en
`src/arm_broker_interfaces/action/MoveArm.action`, y lo genera **en la máquina
donde compilas**. El que produce ese código es
`src/arm_broker_interfaces/CMakeLists.txt`:

```cmake
rosidl_generate_interfaces(${PROJECT_NAME}
  "action/MoveArm.action"
  "msg/QueueState.msg"
  DEPENDENCIES action_msgs builtin_interfaces
)
```

Entonces la cadena completa es esta:

```
CMakeLists.txt  --colcon build-->  MoveArm.py  -->  import en cliente.py
   (archivo de texto)               (generado)      (necesario en la Pi)
```

**La conclusión:** la Raspberry necesita el paquete `arm_broker_interfaces` y
necesita compilarlo ella misma. No basta con que el Jetson lo tenga compilado.
El código generado no viaja por la red; se genera en local.

Por eso el orden en cada máquina es siempre el mismo:

```bash
mkdir -p ~/rb2_ws/src
cp -r src/* ~/rb2_ws/src/          # los dos paquetes, completos
cd ~/rb2_ws && colcon build        # aquí se genera MoveArm.py
source ~/rb2_ws/install/setup.bash
```

---

## 2 · Tabla maestro, archivo por archivo

Acá está el detalle completo. Para cada archivo: dónde va, y qué pasa si lo
ponés donde no es.

### Los dos paquetes: van a las 5 máquinas

| Archivo | Jetson | Pi | Qué es |
|---|:---:|:---:|---|
| `src/arm_broker/package.xml` | Sí | Sí | Declara que el paquete depende de `rclpy`, `sensor_msgs` y `arm_broker_interfaces` |
| `src/arm_broker/setup.py` | Sí | Sí | Declara los dos ejecutables. **Sin este archivo no hay `ros2 run`** |
| `src/arm_broker/setup.cfg` | Sí | Sí | Configuración de instalación de Python |
| `src/arm_broker/resource/arm_broker` | Sí | Sí | Marcador vacío. ROS 2 lo usa para detectar el paquete |
| `src/arm_broker/arm_broker/__init__.py` | Sí | Sí | Vacío. Marca el directorio como paquete Python |
| `src/arm_broker/arm_broker/cliente.py` | Sí | Sí | El cliente. **Lo ejecuta la Pi** |
| `src/arm_broker/arm_broker/broker.py` | Sí | Sí | El broker. **Lo ejecuta el Jetson** |
| `src/arm_broker/arm_broker/fk.py` | Sí | Sí | Cinemática directa. La usa el broker para admitir o rechazar |
| `src/arm_broker/arm_broker/politicas.py` | Sí | Sí | FIFO y round-robin. La usa el broker |
| `src/arm_broker_interfaces/package.xml` | Sí | Sí | Declara las dependencias de generación |
| `src/arm_broker_interfaces/CMakeLists.txt` | Sí | Sí | **Genera el código de la acción.** Imprescindible en la Pi |
| `src/arm_broker_interfaces/action/MoveArm.action` | Sí | Sí | La definición de la acción. Se transforma en `MoveArm.py` al compilar |
| `src/arm_broker_interfaces/msg/QueueState.msg` | Sí | Sí | La telemetría de la cola. Se transforma en `QueueState.py` al compilar |

Fíjate en la fila de `fk.py`. **Va también en la Raspberry**, aunque la
Raspberry nunca calcula una FK. Va porque `broker.py` la importa, y
`broker.py` está en el mismo paquete. Si borrás `fk.py` de la Pi buscando
"ahí no se usa", la Pi sigue compilando, pero el día que quieras correr el broker
desde la Pi por cualquier razón, falla con `ModuleNotFoundError: arm_broker.fk`.

**Por qué va `broker.py` en la Pi si ahí solo corre el cliente:** por el punto 1.
El paquete es indivisible. Copiar solo `cliente.py` funciona hoy, pero rompe el
`git pull`, rompe la uniformidad de versiones entre máquinas, y te va a morder
en la sustentación. Está desarrollado en la sección 8.

### `herramientas/`: solo en el Jetson

| Archivo | Jetson | Pi | Por qué |
|---|:---:|:---:|---|
| `herramientas/generar_carga.py` | Sí | No | Importa `fk` y escribe `carga.csv`. Se corre **una vez** en el Jetson |
| `herramientas/verificar_fk.py` | Sí | No | Abre `/dev/ttyUSB0` e importa `pymycobot`. **Solo el Jetson tiene el brazo conectado** |

`verificar_fk.py` es el ítem 1 (medir la FK contra el robot). Corre en el Jetson
y **con el driver apagado**, porque los dos hablarían del puerto serie a la vez.

### `analisis/`: solo en el Jetson

| Archivo | Jetson | Pi | Por qué |
|---|:---:|:---:|---|
| `analisis/exportar_csv.py` | Sí | No | Importa `rosbag2_py`. Convierte los bags, que están en el Jetson |
| `analisis/metricas.py` | Sí | No | Importa `matplotlib`. Genera `comparacion_politicas.png` |

Los dos necesitan que las corridas ya hayan pasado. Son la etapa de "analizar",
y el análisis se hace donde están los datos.

### Datos y resultados: solo en el Jetson

| Archivo | Jetson | Pi | Por qué |
|---|:---:|:---:|---|
| `carga.csv` (original) | Sí | Copia | Ver la sección 6, que es importante |
| `rechazos.log` | Sí | No | Lo genera el broker con `tee`. Entregable del ítem 2 |
| `corrida_fifo/` | Sí | No | Bag de la corrida 1 |
| `corrida_rr/` | Sí | No | Bag de la corrida 2 |
| `fifo/` | Sí | No | `queue_state.csv` y `joint_states.csv` de la corrida 1 |
| `roundrobin/` | Sí | No | Ídem de la corrida 2 |
| `comparacion_politicas.png` | Sí | No | La figura. Entregable del ítem 3 |

### Documentación: donde quieras, o en ninguna parte

`README.md`, `CAMBIOS.md`, `docs/`, `LEEME.md`, y los otros `.md` no se ejecutan.
No hacen falta en ninguna máquina para que nada funcione. Se quedan en tu PC y
en el repo de GitHub.

Si querés leer la documentación desde la Raspberry, copialos. Pero acordate de que
`git clone` baja todo el repo, así que van a estar igual. **No es un error que
estén ahí.**

---

## 3 · Lo que NO va a la Raspberry

### El workspace del driver: solo Jetson

Este es el punto que más se malinterpreta, así que va con detalle.

El driver del brazo (`jetcobot_driver`) **no está en este repo**. Vive en otro
workspace del curso, que ya viene compilado. Ese workspace va **exclusivamente en
el Jetson**.

**Nunca compiles el driver en una Raspberry.** Ni por error, ni "para ver si
funciona". Si corrés `colcon build` ahí y algo sale mal, rompiste el driver de tu
equipo para la clase siguiente.

Lo único que la Raspberry necesita del driver es poder **escuchar** lo que este
publica. Y para eso no hace falta el driver instalado: basta con que el driver
esté corriendo en el Jetson y los dos estén en la misma red con el mismo
`ROS_DOMAIN_ID`.

### El driver nunca se lanza en una Raspberry

Hay una sola interfaz física hacia el brazo: `/dev/ttyUSB0`, y está en el Jetson.
Por lo tanto:

- `ros2 run jetcobot_driver sync_plan_nx` se ejecuta **una sola vez**, en el
  Jetson, y por **una sola persona**.
- Aunque por un error el paquete `jetcobot_driver` estuviera instalado en la
  Raspberry, **nadie lo lanza ahí**. No rompería nada, pero tampoco sirve de nada.

### El `bag` no se graba en la Raspberry

`ros2 bag record` se lanza en el **Jetson**, y graba dos tópicos:
`/arm/queue_state` y `/joint_states`.

¿Por qué no en la Raspberry? Por dos razones:

1. La telemetría de la cola la publica el broker, que está en el Jetson. Es
   mucho más simple grabar junto al broker.
2. `exportar_csv.py` necesita `rosbag2_py`, y así el bag y el script están en la
   misma máquina, sin tener que mover un archivo grande por la red.

### `carga.csv` se **genera** en el Jetson pero **se necesita** en la Pi

Esta fila de la tabla es la única que va en las dos columnas, y merece su propia
sección porque es la causa más común de que un cliente no haga nada.

---

## 4 · Tabla por proceso: qué ejecuta cada uno y qué necesita

Los seis procesos del reto, con el paquete que ejecuta cada uno y los imports que
lo sostienen. Esta tabla es la que conecta la parte conceptual con la de archivos.

| # | Proceso | Máquina | Se lanza con | Módulo que corre | Imports que necesita |
|---|---|---|---|---|---|
| 1 | Driver | Jetson | `ros2 run jetcobot_driver sync_plan_nx` | Del workspace del curso | Los suyos, y `/dev/ttyUSB0` |
| 2 | Broker | Jetson | `ros2 run arm_broker broker` | `arm_broker/broker.py` | `rclpy`, `sensor_msgs.msg.JointState`, `arm_broker.fk`, `arm_broker.politicas`, `arm_broker_interfaces` |
| 3 | Cliente 1 | Raspberry 1 | `ros2 run arm_broker cliente -p client_id:=uno` | `arm_broker/cliente.py` | `rclpy`, `arm_broker_interfaces.action.MoveArm` |
| 4 | Cliente 2 | Raspberry 2 | `... -p client_id:=dos` | El mismo | Los mismos |
| 5 | Cliente 3 | Raspberry 3 | `... -p client_id:=tres` | El mismo | Los mismos |
| 6 | Cliente 4 | Raspberry 4 | `... -p client_id:=cuatro` | El mismo | Los mismos |
| 7 | Grabadora | Jetson | `ros2 bag record -o <carpeta> /arm/queue_state /joint_states` | `rosbag2` | Ninguno de tu repo |

Leé la fila 2 de nuevo: el broker importa **`fk` y `politicas`**. Leé la fila 3:
el cliente **no** importa nada de eso. Esa diferencia es la base de la sección 8.

Y algo que conviene notar: `arm_broker/package.xml` declara sus dependencias así:

```xml
<depend>rclpy</depend>
<depend>sensor_msgs</depend>
<depend>arm_broker_interfaces</depend>
```

**`jetcobot_driver` no aparece.** Tu paquete no depende del driver. El broker se
comunica con el brazo por el tópico `/joint_states`, no importando nada del
driver. Por eso el broker compila y arranca igual en una Raspberry: no le importa
que el driver exista o no, le importa que el driver esté **corriendo** y
publicando.

---

## 5 · El flujo de copia real

### La forma recomendada: Git en las cinco máquinas

El repo se clona completo en las 5. Es lo que hace que las cinco tengan
exactamente el mismo código, y por eso el paso B4.4 del manual general te pide
recompilar después de cada `git pull`.

**En cada una de las 5 máquinas** (Jetson y las 4 Raspberry):

```bash
cd ~
git clone https://github.com/TU_USUARIO/rb2-turno-del-brazo.git rb2
cd rb2
mkdir -p ~/rb2_ws/src
cp -r src/* ~/rb2_ws/src/
cd ~/rb2_ws && colcon build
source ~/rb2_ws/install/setup.bash
```

`git clone` baja **todo**: los dos paquetes, `herramientas/`, `analisis/`, los
`.md`. Y está bien que sea así, porque lo que importa es que `src/` esté
completo en todas partes. Los archivos de más no molestan: no se ejecutan solos.

La única asimetría real está en la línea del `source`, y ya la tenías vista:

- En el **Jetson**, además: `source <TU-CARPETA-DRIVER>/install/setup.bash`
- En las **Raspberry**, **no** esa línea, porque el driver no está ahí.

### La forma alternativa: copia manual

Si todavía no tenés Git andando, podés copiar la carpeta `src` con un pendrive o
por red. La estructura final tiene que ser la misma:

```
~/rb2_ws/
├── src/
│   ├── arm_broker/              <- completo
│   └── arm_broker_interfaces/   <- completo
├── build/
├── install/
└── log/
```

El error clásico de la copia manual es copiar **una sola carpeta** de las dos, o
copiar `arm_broker/` sin `arm_broker_interfaces/`. En ambos casos el
`colcon build` puede no dar error, y el fallo aparece después como
`ModuleNotFoundError`.

### Por qué copiar y no trabajar directo en `~/rb2_ws/src/`

Es el mismo motivo por el que existe la separación en los dos manuales: si
compilaras dentro del repo, las carpetas `build/`, `install/` y `log/` quedarían
dentro de tu carpeta versionada, y `git status` se llenaría de basura. Con la
copia, el repo queda limpio y podés borrar `build/`, `install/` y `log/` sin
miedo.

### El matiz de `carga.csv` y el `.gitignore`

Un detalle que aparece acá y que conviene resolver ahora. Si vas a crear un
`.gitignore` en el repo, **no incluyas `carga.csv`**.

La razón es que la forma limpia de llevar `carga.csv` a las 4 Raspberry es
`git pull`:

```bash
# En el Jetson, una sola vez que la carga esté congelada
git add carga.csv
git commit -m "carga oficial: 40 poses, semilla 7"
git push

# En cada Raspberry
git pull
```

Si `carga.csv` está en el `.gitignore`, ese `git pull` no lo baja, y te queda
la copia manual por red o pendrive, que es lo que querías evitar.

Lo que sí va en el `.gitignore`:

```
__pycache__/
build/
install/
log/
```

Y si en algún momento volvés a tener una carpeta duplicada dentro del repo, como
la que movimos hoy, agregala también. Git no sube carpetas vacías, pero sí sube
todo el contenido de una carpeta con archivos.

---

## 6 · `carga.csv` aparte: por qué la Pi la necesita

Este es el punto que más confunde, porque `carga.csv` se genera en el Jetson y
sin embargo la Raspberry la necesita.

### Lo que viaja por la red, y lo que no

Cuando un cliente manda un goal, lo que viaja en el mensaje es **solo esto**, y
son seis números. De `MoveArm.action`:

```
float64[] joint_positions
string    client_id
uint8     priority
```

Seis flotantes, un texto y un entero. **El archivo `carga.csv` no viaja.** Nunca
viaja.

### Entonces, ¿de dónde saca el cliente las poses?

Las lee del disco, en su propia máquina. `cliente.py`, líneas 40 a 42:

```python
with open(ruta, newline='') as f:
    filas = [r for r in csv.reader(f) if r and not r[0].lstrip().startswith('#')]
return [[float(v) for v in fila[:6]] for fila in filas]
```

`open(ruta)`. Un `open` común, de Python, sobre un archivo local. La Raspberry
lee `carga.csv` de su propio disco, con su propio sistema de archivos. No pide
el archivo a nadie.

### Qué pasa si a la Pi le falta `carga.csv`

Este es el fallo, y es silencioso. Si el cliente se lanza sin el archivo, el
parámetro `traza` viene vacío, y `cliente.py` líneas 36 a 39 hace esto:

```python
def cargar(self, ruta):
    if not ruta:
        return [[0.3, 0.0, 0.0, 0.0, 0.0, 0.0],
                [-0.3, 0.0, 0.0, 0.0, 0.0, 0.0]]
```

Si la ruta está vacía, **no da error**. Se inventa dos poses de prueba y manda
esas dos. Vas a ver al cliente funcionando, pidiendo goals,e imprimiendo
`espera=...s ejec=3.0s`, y todo parece perfecto. Pero mandó 2 poses, no 40, y el
ítem 3 no tiene datos.

**Cómo detectarlo antes de la corrida:** el cliente tiene que imprimir
`pose 0` hasta `pose 39`. Si el último es `pose 1`, te falta `carga.csv`.

### Las tres verificaciones de la carga

En el Jetson, una vez generada:

```bash
python3 herramientas/generar_carga.py --n 40 --semilla 7 --salida carga.csv
wc -l carga.csv
md5sum carga.csv
```

`wc -l` tiene que dar 40 líneas. Guardá ese hash.

En cada Raspberry, después del `git pull`:

```bash
wc -l ~/rb2/carga.csv
md5sum ~/rb2/carga.csv
```

**Los cuatro hashes tienen que ser idénticos al del Jetson.** Si no coinciden,
las dos corridas no son comparables, porque cada cliente estuvo mandando poses
distintas.

---

## 7 · La trampa de `termin/src/`

Esta es la más peligrosa de todas, porque no da ningún error.

En tu carpeta hay **dos** carpetas `src`, y no son lo mismo.

| | `termin/src/` | `termin/solucionOO+/src/` |
|---|---|---|
| Qué es | El **andamiaje** que te entregó el curso | **Tu solución** |
| `broker.py` | 8003 bytes | **16242 bytes** |
| `fk.py` | 2615 bytes | **6350 bytes** |
| `politicas.py` | 1777 bytes | **5142 bytes** |
| `cliente.py` | 3308 bytes | 3308 bytes (idéntico) |
| `package.xml`, `setup.py`, `CMakeLists.txt` | Iguales | Iguales |
| `MoveArm.action`, `QueueState.msg` | Iguales | Iguales |

Fijate en la última fila. **Los archivos de andamiaje son idénticos a los
tuyos.** Por eso es tan fácil confundirse: si abrís `termin/src/`, el `setup.py`
es el mismo, el `CMakeLists.txt` es el mismo, y todo parece correcto.

Y sin embargo, `broker.py` de una tiene 8003 bytes y el tuyo tiene 16242. La
diferencia es todo tu trabajo del ítem 2: los cuatro bloques de la solución, la
admisión con FK, los rechazos con motivo, la exclusión mutua, las políticas.

### Qué pasa si copiás el `src` equivocado

Nada visible. `colcon build` termina con `Finished <<<`, `ros2 run arm_broker
broker` arranca, se ve el mensaje de listo, `/arm/queue_state` publica a 5 Hz.

Pero el broker no admite goals con FK, no rechaza por paso articular, no
registra motivos, y el ítem 2 vale cero. Todo lo demás parece funcionar.

**Esto no se detecta mirando la terminal del broker. Se detecta mirando qué
archivo estás copiando.**

### Cómo evitarlo

Tres formas, de la más simple a la más robusta:

1. **Nunca copies desde `termin/src/`.** Tu código está en
   `termin/solucionOO+/src/`. Son dos carpetas al mismo nivel, y solo una tiene
   lo tuyo.

2. **Verificá el tamaño antes de compilar.** Un solo comando, y el número no
   miente:

   ```bash
   ls -l src/arm_broker/arm_broker/broker.py
   ```

   Tiene que medir **16242 bytes**. Si mide 8003, estás en la carpeta
   equivocada.

3. **Usá `git clone` y no la copia manual.** Es la que resuelve esto de
   raíz: el repo que subiste es tu solución, así que no hay forma de clonar el
   andamiaje por accidente. Es la razón principal por la que el paso B1 del
   manual general empieza por Git.

### La tabla de hashes, para que la tengas a mano

Guardá esta tabla. Si alguna vez dudás de qué versión tenés en una máquina,
compará el tamaño del archivo y sabés:

| Archivo | Andamiaje (mal) | Tu solución (bien) |
|---|---|---|
| `broker.py` | 8003 | **16242** |
| `fk.py` | 2615 | **6350** |
| `politicas.py` | 1777 | **5142** |
| `cliente.py` | 3308 | 3308 (da igual cuál) |

Un chequeo rápido de que la máquina tiene tu versión:

```bash
cd ~/rb2_ws/src/arm_broker/arm_broker
wc -c broker.py fk.py politicas.py
```

Esperado: `16242`, `6350`, `5142`.

---

## 8 · El mínimo teórico y el recomendado

Esta sección es la que responde la objeción natural: *"si el cliente no importa
`fk` ni `politicas`, ¿para qué los pongo en la Raspberry?"*

### El mínimo teórico

Mirá los imports de `cliente.py`, líneas 10 a 14:

```python
import rclpy
from rclpy.action import ActionClient
from rclpy.node import Node

from arm_broker_interfaces.action import MoveArm
```

Son cuatro imports. **No aparece `fk`, ni `politicas`, ni `broker`.** El cliente
es autónomo: solo sabe construir un goal y esperar el resultado.

O sea que, en el papel, en la Raspberry bastaría con:

```
src/arm_broker/arm_broker/cliente.py
src/arm_broker/arm_broker/__init__.py
src/arm_broker_interfaces/  (completo, para generar MoveArm)
src/arm_broker/setup.py     (para el ejecutable)
```

Y funcionaría. Eso es un hecho, no una conjetura.

### Por qué no lo hagas

Cuatro razones, de la más fuerte a la más débil.

**1. El paquete es indivisible de todos modos.** Quitás `broker.py` de la
Raspberry, pero `setup.py` sigue declarando el ejecutable `broker`. Si alguien
corre `ros2 run arm_broker broker` en la Pi, revienta con
`ModuleNotFoundError`. Tenés un paquete a medio instalar, que es el peor estado
posible: ni funciona entero ni falla claro.

**2. `git pull` se rompe.** Es la razón práctica que más te va a doler. Si tenés
una versión recortada, cada vez que alguien haga un `git pull` en la Raspberry
van a aparecer archivos modificados o borrados que vos no tocaste. Git te va a
pedir que resuelvas conflictos de tu propia edición, en una máquina que no
editaste. Es un problema recurrente y tedioso.

**3. Las cinco máquinas tienen que ser auditables.** En la sustentación, si
preguntan en qué commit estabas cada máquina, la respuesta tiene que ser la
misma en las cinco. Con el `src/` completo, es `git rev-parse HEAD` en cada una y
los cinco dan el mismo hash. Con un `src/` recortado, la respuesta es
"depende".

**4. Es una optimización que no optimiza nada.** Copiar cuatro archivos de texto
por la red local toma milisegundos. Lo que cuesta tiempo de verdad es compilar, y
esa parte no es opcional: la Raspberry necesita compilar igual.

### La regla

> Compilá el `src/` **completo** en las cinco máquinas. No es que no puedas
> recortar: es que no hay nada que ganar y cuatro cosas que perder.

La distribución real del trabajo no está en el código, está en los datos y en
los resultados. El código va entero a todas partes; los bags, los CSV, la figura
y el log de rechazos se generan en el Jetson.

---

## 9 · Verificación y checklist

### En el Jetson

```bash
# 1. El driver está disponible
source <TU-CARPETA-DRIVER>/install/setup.bash
ros2 pkg list | grep jetcobot

# 2. Tu reto está compilado
source ~/rb2_ws/install/setup.bash
ros2 pkg list | grep -E 'arm_broker|jetcobot_driver'

# 3. La acción se generó
ros2 interface show arm_broker_interfaces/action/MoveArm

# 4. Tienes TU versión del código, no el andamiaje
wc -c ~/rb2_ws/src/arm_broker/arm_broker/{broker,fk,politicas}.py

# 5. La carga existe
cd ~/rb2 && wc -l carga.csv && md5sum carga.csv
```

Lo esperado en el paso 4: `16242`, `6350`, `5142`. Si salen `8003`, `2615`,
`1777`, estás con el andamiaje. Volvé a la sección 7.

### En cada Raspberry

```bash
# 1. Tu reto está compilado
source ~/rb2_ws/install/setup.bash
ros2 pkg list | grep arm_broker

# 2. La acción se generó AQUÍ, no vino del Jetson
ros2 interface show arm_broker_interfaces/action/MoveArm

# 3. Tenés la carga, y es la misma
wc -l ~/rb2/carga.csv
md5sum ~/rb2/carga.csv

# 4. La prueba de fuego: el cliente arranca
ros2 run arm_broker cliente --ros-args -p client_id:=prueba
```

El paso 4 es el que de verdad importa, y hay que saber interpretarlo:

| Lo que ves | Qué significa |
|---|---|
| `[prueba] esperando al broker...` y a los 15 s `El broker no aparece` | **Perfecto.** El cliente está bien instalado. La Pi funciona |
| `ModuleNotFoundError: No module named 'arm_broker_interfaces'` | No compilaste el paquete de interfaces en la Pi. Volvé a `colcon build` |
| `ModuleNotFoundError: No module named 'arm_broker'` | No compilaste, o no sourceaste el workspace |
| `ros2 run arm_broker cliente: command not found` | Falta el `source`, o `setup.py` no está en el workspace |
| Imprime `pose 0` y `pose 1`, y termina | **Le falta `carga.csv`.** Ver la sección 6 |

La primera fila es la buena, y suena a error pero no lo es: el cliente busca la
acción `/move_arm` durante 15 segundos, no la encuentra porque todavía no
lanzaste el broker, y se sale con un mensaje. **Eso demuestra que el paquete se
instaló bien en esa máquina.** Es el diagnóstico más útil que tenés, y funciona
sin necesidad de que el broker esté arriba ni de que la red ande.

### Checklist antes de la corrida

En cada máquina, tick:

- [ ] `git clone` hecho, repo completo
- [ ] `cp -r src/* ~/rb2_ws/src/` con los **dos** paquetes
- [ ] `colcon build` terminó con `Finished <<<` en los dos paquetes
- [ ] `ros2 interface show arm_broker_interfaces/action/MoveArm` imprime algo
- [ ] `echo $ROS_DOMAIN_ID` da **47** en las 5
- [ ] `echo $ROS_LOCALHOST_ONLY` da **0** en las 5
- [ ] Driver launchable en el Jetson: `ros2 pkg list | grep jetcobot`
- [ ] `wc -c broker.py` da **16242** (tu versión, no el andamiaje)
- [ ] `carga.csv` con 40 líneas en el Jetson y en las 4 Pi
- [ ] `md5sum carga.csv` **idéntico** en las 5
- [ ] `ros2 topic info /joint_states -v` dice `Publisher count: 1`
- [ ] Prueba de fuego del cliente: dice `esperando al broker...` y no revienta

### El resumen, en una frase

> El código va completo a las cinco máquinas porque `broker` y `cliente` son el
> mismo paquete y la acción se genera localmente. Los datos (`carga.csv`) van a
> las cuatro Raspberry. Los resultados (bags, CSV, figura, log) se quedan en el
> Jetson. Y el `src` que copiás siempre es el de
> `termin/solucionOO+/src/`, nunca el de `termin/src/`.
