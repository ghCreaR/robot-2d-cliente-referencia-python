# Plan de implementación · robot-2d-cliente-referencia-python

Este plan detalla cómo construir el cliente de control de referencia descrito en el [README del repositorio común](https://github.com/ojgarciab/carrera-robots-autonomos). Todavía no hay código: es una propuesta para revisar antes de empezar.

La API que usa el cliente está definida en [`contratos/api-cliente.md`](https://github.com/ojgarciab/carrera-robots-autonomos/blob/master/contratos/api-cliente.md) del repositorio común.

## 1. Objetivos

- Que un participante pueda **probar su primer algoritmo en minutos**, sin conocer los detalles de la API.
- Que el código sea **didáctico**: claro, comentado y con ejemplos fáciles de modificar.
- Usar **WebSocket** para los datos en tiempo real, que es el medio de menor latencia. El polling REST de la API no se usa en este cliente.
- Servir de **cliente de pruebas** del servidor en la integración de extremo a extremo.

## 2. Decisiones técnicas propuestas

| Tema | Propuesta | Motivo |
|------|-----------|--------|
| Lenguaje | **Python 3.10+** | Es la versión de Ubuntu 22.04 LTS, que aún tiene soporte, así que los participantes pueden usar el Python del sistema sin instalar otro. El código no usa nada posterior a 3.10 (por ejemplo, ni `tomllib` ni `TaskGroup`), y la CI lo comprueba. |
| WebSocket | **websockets** (`asyncio`) | Biblioteca madura y sencilla. |
| Consultas iniciales | **httpx** (`asyncio`) | Solo para `GET /mundos` y `GET /robots/<id>` al arrancar. |
| Línea de órdenes | **argparse** | Sin dependencias extra. |
| Pruebas | **pytest** y **pytest-asyncio**, con una pasarela falsa en proceso | |
| Calidad | **ruff** y **mypy** | |
| Publicación | Paquete instalable con `pip install git+https://…` y, más adelante, PyPI | |

Internamente es asíncrono, pero el participante puede escribir su algoritmo como una **función normal (síncrona)**: la biblioteca la llama en cada lectura. Así no hace falta saber `asyncio` para empezar.

## 3. Estructura del repositorio

```
robot-2d-cliente-referencia-python/
├── pyproject.toml
├── src/robot_cliente/
│   ├── __init__.py          # Cliente, Algoritmo, Lectura
│   ├── __main__.py          # línea de órdenes: robot-cliente
│   ├── config.py            # pasarela, token (entorno o fichero), mundo, modelo…
│   ├── cliente.py           # ciclo: entrar, bucle de control, salir
│   ├── conexion.py          # WebSocket: autenticar, entrar, lecturas, actuadores, reconexión
│   ├── consultas.py         # GET /mundos y GET /robots/<id>
│   ├── reloj.py             # ping: rtt, desfase de reloj y latencia de cada muestra
│   ├── modelo.py            # definición del robot (GET /robots/<id>)
│   └── algoritmos/
│       ├── base.py          # clase Algoritmo
│       ├── reglas_3ir.py    # seguidor por reglas para 3 sensores
│       ├── pid_5ir.py       # seguidor PID para 5 sensores
│       └── busqueda.py      # búsqueda inicial de la línea
├── ejemplos/
│   ├── minimo.py            # el algoritmo más corto posible, muy comentado
│   └── registrar.py         # guarda las lecturas en CSV para analizarlas
└── tests/
```

## 4. Diseño de la biblioteca

### 4.1. Interfaz para el algoritmo

```python
@dataclass
class Lectura:
    valores: dict[str, float]   # id del sensor -> valor (hoy 0 o 1; en el futuro, de 0 a 1)
    ts: float                   # marca del servidor (ms)
    dt: float | None            # s desde la lectura anterior (para PID)
    seq: int
    perdidas: int               # muestras perdidas desde la anterior

class Algoritmo:
    def inicio(self, modelo): ...                  # opcional: lee los ids de sensores y motores
    def paso(self, lectura: Lectura) -> dict[str, float]: ...   # devuelve los actuadores
    def fin(self): ...                             # opcional
```

- La biblioteca **limita los actuadores** al rango `[-1, 1]` y comprueba que los `id` existan en el modelo, con un error claro si no.
- Los `id` de sensores y motores se leen del modelo, no se escriben a mano: así el mismo algoritmo vale para robots con nombres distintos.

### 4.2. Ciclo de vida

1. Lee la configuración y comprueba que el token es de lectura-escritura (prefijo `crt_rw_`).
2. `GET /mundos`: comprueba que el mundo existe, está activo y permite el modelo elegido. Si no se indica mundo, lista los disponibles.
3. `GET /robots/<modelo>`: carga la definición.
4. Entra al mundo. Si está lleno, reintenta con espera creciente o termina, según una opción.
5. **Bucle de control:** recibe cada lectura, llama a `paso()` y manda los actuadores.
6. Con `Ctrl+C`, para los motores (manda `0` a todos), sale del mundo con `salir` si se pidió con `--salir-al-terminar` y cierra limpiamente.

### 4.3. Conexión WebSocket

- Abre `/ws` con la cabecera `Authorization: Bearer <token>` y manda `entrar`.
- **Lecturas:** llegan solas, empujadas por el servidor en cuanto se toman. Si el algoritmo tarda más que el intervalo entre lecturas, se procesa siempre la **más reciente** y se cuentan las descartadas, en lugar de acumular retraso.
- **Actuadores:** mensaje `actuadores` por la misma conexión, después de cada `paso()`.
- **Motores:** con WebSocket, el servidor mantiene los motores activos mientras la conexión siga abierta, aunque no lleguen consignas nuevas.
- **Desconexión:** reconecta con espera exponencial (de 1 s a 30 s) y vuelve a mandar `entrar`. Si vuelve antes de 5 minutos, recupera su robot donde esté. Mientras está desconectado, el robot frena solo hasta parar.
- **Ping:** el comando `ping` va por la misma conexión.

### 4.4. Reloj y latencia

- Al empezar, y después cada pocos segundos, se hace `ping` y se calculan `rtt = t1 − t0` y `desfase ≈ ts − (t0 + t1) / 2`, con la mediana de varias medidas para filtrar el ruido.
- Con eso se calcula la **latencia de cada muestra** (hora local de llegada − (marca del sensor − desfase)) y se muestra en el registro con `--verbose`.
- `dt` se calcula con las **marcas del servidor**, no con la hora local, para que los términos derivativo e integral del PID no dependan de la red.

## 5. Algoritmos de ejemplo

### 5.1. Búsqueda inicial (común)

El robot aparece en un punto aleatorio mirando al centro del mapa, así que:

1. Avanza recto a velocidad moderada hasta que algún sensor ve la línea.
2. Gira hacia el lado del sensor que la ha visto hasta centrarla y pasa al seguimiento.
3. Si pierde la línea durante el seguimiento, gira hacia el último lado donde la vio. Si tras un tiempo no la encuentra, vuelve al paso 1.
4. Si lleva demasiado tiempo sin ver la línea, por ejemplo porque está contra una pared o porque otro robot lo ha empujado, retrocede un poco, gira un ángulo aleatorio y vuelve al paso 1. El cliente no sabe dónde está (solo ve sus sensores), así que se guía solo por el tiempo.

### 5.2. Robot de 3 sensores: reglas

| Izq. | Centro | Der. | Acción |
|:---:|:---:|:---:|---|
| 0 | 1 | 0 | Recto. |
| 1 | 1 | 0 | Giro suave a la izquierda. |
| 1 | 0 | 0 | Giro fuerte a la izquierda. |
| 0 | 1 | 1 | Giro suave a la derecha. |
| 0 | 0 | 1 | Giro fuerte a la derecha. |
| 1 | 1 | 1 | Cruce (circuito en 8): seguir recto. |
| 1 | 0 | 1 | Situación ambigua (cruce visto de lado): mantener la última acción. |
| 0 | 0 | 0 | Línea perdida: búsqueda (5.1, paso 3). |

Las velocidades tienen en cuenta la **inercia** (0,5 m/s² de aceleración): se evita alternar entre extremos y se reduce la velocidad en las curvas.

### 5.3. Robot de 5 sensores: PID

- **Posición de la línea:** media ponderada por el valor de cada sensor de sus posiciones laterales `y`, leídas del modelo, normalizada a un error de `−1` a `1`. Con los sensores digitales actuales es la media de los sensores activos; con el sensor promediado futuro aprovechará automáticamente los valores intermedios.
- **Control:** `giro = Kp·e + Ki·∫e·dt + Kd·de/dt`, con `dt` de las marcas del servidor. Limita el término integral (*anti-windup*) y lo reinicia al perder la línea.
- **Motores:** `izq = base − giro` y `der = base + giro`, limitados a `[-1, 1]`. La velocidad base baja cuando el error es grande.
- **Cruces del 8:** si se activan casi todos los sensores a la vez, mantiene el último giro durante unas pocas lecturas, en vez de reaccionar a la línea transversal.
- Las constantes se ajustan desde la línea de órdenes (`--kp`, `--ki`, `--kd`, `--base`) para experimentar.

## 6. Fases de implementación

### Fase 0 · Esqueleto
- `pyproject.toml` con el comando `robot-cliente`, `ruff`, `mypy`, `pytest` y GitHub Actions (Python 3.10 a 3.13).

### Fase 1 · Pasarela falsa para pruebas
- Una pasarela mínima en proceso (WebSocket, más `GET /mundos` y `GET /robots/<id>`) que simula un robot muy simple. Permite desarrollar y probar el cliente sin el servidor real.

### Fase 2 · Conexión y ciclo de vida
- `config.py`, `consultas.py`, `conexion.py`, `cliente.py`, `modelo.py` y `reloj.py`.
- Pruebas: entrar, mundo lleno, modelo no permitido, token de solo lectura (error claro) y salida limpia.

### Fase 3 · Robustez de la conexión
- Reconexión con espera exponencial y recuperación del robot.
- Descarte de lecturas atrasadas si el algoritmo es lento.
- Pruebas: cortar la conexión de la pasarela falsa y comprobar que se reconecta; un algoritmo lento procesa siempre la última lectura.

### Fase 4 · Algoritmos
- Búsqueda, reglas para 3 sensores y PID para 5, con pruebas unitarias sobre lecturas sintéticas.
- `ejemplos/minimo.py` y `ejemplos/registrar.py`.

### Fase 5 · Integración con el servidor real
- Con el `compose.yaml` del repositorio común: los dos robots completan vueltas en el óvalo y en el ocho.
- Ajuste de los parámetros por defecto de los algoritmos.
- Prueba de recuperación: cortar la conexión unos segundos y comprobar que se recupera el robot donde estaba.

### Fase 6 · Documentación para participantes
- Guía paso a paso en el README: instalar, conseguir el token, primer algoritmo, cómo leer el registro y consejos de ajuste del PID.

## 7. Dependencias con otros repositorios

| Depende de | Qué necesita |
|------------|--------------|
| `robot-2d-pasarela` | WebSocket y las consultas `GET /mundos` y `GET /robots/<id>` de `contratos/api-cliente.md`. Hasta que exista, se usa la pasarela falsa de la fase 1, que implementa el mismo contrato. |
| `robot-2d-motor-fisicas` | Simulación real para la fase 5. |
| `robot-2d-interfaz-web` | Generar el token de lectura-escritura. |

## 8. Decisiones tomadas

- **Sensores IR:** digitales (`0`/`1`) para empezar. Los algoritmos tratan los valores como números, así que funcionarán también con el sensor promediado previsto (`0` a `1`).
- **Python 3.10 como mínimo**, por compatibilidad con las versiones de Ubuntu que aún tienen soporte.
- **Solo WebSocket:** por ahora los clientes de referencia usan solo WebSocket para los datos en tiempo real. El polling REST sigue en la API para quien quiera usarlo desde otros clientes.
- **Choques:** los robots pueden empujarse y chocar con las paredes. Los algoritmos de ejemplo no lo evitan a propósito, pero la búsqueda de la línea debe recuperarse si el robot queda contra una pared (por ejemplo, retrocediendo y girando si lleva un tiempo sin avanzar ni ver la línea).

## 9. Preguntas abiertas

Ninguna por ahora.
