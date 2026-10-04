# robot-2d-cliente-referencia-python

Cliente de control de referencia, en Python, para los robots sigue líneas de la [carrera de robots autónomos](https://github.com/ojgarciab/carrera-robots-autonomos).

> **Estado:** en diseño. Todavía no hay código; el plan de implementación está en [`Plan.md`](Plan.md).

## Qué es

En la carrera, cada participante escribe solo la "inteligencia" de su robot: lee los sensores, decide y mueve los motores. El servidor pone todo lo demás (física, sensores, circuito y visualización).

Este repositorio es el **punto de partida** para escribir esa inteligencia en Python, y tiene dos partes:

1. **Una biblioteca de conexión** que oculta los detalles de la API: autenticación con el token, elección del mundo y del robot, WebSocket o polling REST, marcas de tiempo, latencia y reconexión. Así el participante solo escribe su algoritmo.
2. **Algoritmos de ejemplo** que siguen la línea con los dos robots de prácticas:
   - **3 sensores IR:** control por reglas sencillas (izquierda, centro, derecha).
   - **5 sensores IR:** control **PID** sobre la posición estimada de la línea.

   Los dos incluyen la **búsqueda inicial de la línea**: el robot aparece orientado hacia el centro del mapa, así que avanza recto hasta encontrarla.

Sirve también como **ejemplo vivo de la API**: quien quiera escribir un cliente en otro lenguaje puede leerlo para ver cómo se usa cada parte.

```
  ┌───────────────────────────┐   WebSocket o REST   ┌──────────┐
  │ tu algoritmo              │  ◄── sensores (10 Hz)│ pasarela │
  │   └─ biblioteca de cliente│  ── actuadores ─────►│          │
  └───────────────────────────┘                      └──────────┘
```

## Uso previsto

Se instalará como paquete de Python y se podrá usar desde la línea de órdenes:

```sh
export CRT_TOKEN=crt_rw_…          # token de lectura-escritura de la interfaz de gestión
robot-cliente --pasarela http://localhost:8080 \
              --mundo <uuid> --modelo sigue-lineas-5ir \
              --algoritmo pid --transporte ws
```

O desde código, escribiendo solo la función de control:

```python
from robot_cliente import Cliente, Algoritmo

class MiAlgoritmo(Algoritmo):
    def paso(self, lectura):
        # lectura.valores: {"ir_izquierdo": 0, "ir_central": 1, …}
        # lectura.ts y lectura.dt: marca de tiempo e intervalo desde la anterior
        return {"motor_izquierdo": 0.5, "motor_derecho": 0.5}
```

El token se lee de una variable de entorno o de un fichero, **nunca** de un argumento de la línea de órdenes, para que no quede en el historial.

## Documentación relacionada

- [API de cliente](https://github.com/ojgarciab/carrera-robots-autonomos#api-de-cliente) y [marcas de tiempo y latencia](https://github.com/ojgarciab/carrera-robots-autonomos#marcas-de-tiempo-y-latencia)
- [Modelos de robot](https://github.com/ojgarciab/carrera-robots-autonomos#modelos-de-robot) y [ciclo de vida del robot en el mundo](https://github.com/ojgarciab/carrera-robots-autonomos#ciclo-de-vida-del-robot-en-el-mundo)
- [Plan de implementación](Plan.md)

## Licencia

[GPL-3.0](LICENSE).
