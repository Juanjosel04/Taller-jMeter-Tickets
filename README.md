# Taller de rendimiento con JMeter · Tiquetmania

Monorepo del taller de pruebas de rendimiento. Contiene la API simulada de compra de boletas de Tiquetmania, el plan de JMeter que se ejecuta en el pipeline y el plan de experimentación local.

## Estructura

```
api/
  ticketmania_simple.py        API bajo prueba (solo biblioteca estándar de Python)
pruebas/
  jmeter/
    Ticketmania.jmx            plan que ejecuta el pipeline
    datos.csv                  datos de compra (tarifa, cantidad)
  experimentacion/
    Experimentacion.jmx        escenarios "¿Qué pasaría si...?" (solo local)
    datos.csv                  mismos datos de compra del plan principal
    datos_total.csv            datos con el total esperado (pregunta 9)
    datos_total_alterado.csv   igual, con una fila de total incorrecto (pregunta 9)
.github/workflows/
  rendimiento.yml              pipeline de GitHub Actions
```

## API

Requiere Python 3.8 o superior.

```bash
python api/ticketmania_simple.py
```

Queda en `http://127.0.0.1:8080`.

| Método | Ruta | Respuesta |
|---|---|---|
| GET | `/api/salud` | `{"estado": "OK"}` |
| GET | `/api/tarifas` | lista de tarifas y precios |
| POST | `/api/seleccion` | crea una reserva (201) con `tarifa`, `cantidad` y `total` |
| POST | `/api/pago` | paga la reserva (200, `APROBADO`); pagarla otra vez responde 404 |

## Plan de JMeter

`pruebas/jmeter/Ticketmania.jmx` simula 200 usuarios que entran en 300 segundos. Cada uno consulta las tarifas, selecciona una tarifa y una cantidad leídas de `datos.csv`, y paga la reserva. Entre peticiones hay una pausa aleatoria de 500 a 1500 ms. El pago se valida con una aserción JSON sobre `estado = APROBADO`.

Ejecución local en modo no-GUI, con la API arriba y desde la carpeta del plan:

```bash
cd pruebas/jmeter
jmeter -n -t Ticketmania.jmx -l resultados.jtl -e -o reporte
```

El reporte queda en `pruebas/jmeter/reporte/index.html`.

## Pipeline

`.github/workflows/rendimiento.yml` se ejecuta con cada push a `main` o a mano desde Actions:

1. Descarga el repositorio en una máquina `ubuntu-latest`.
2. Instala Java 17 y JMeter 5.6.3, y agrega `bin` de JMeter al `PATH`.
3. Levanta la API en segundo plano y espera a que `/api/salud` responda.
4. Ejecuta `Ticketmania.jmx` en modo no-GUI y genera el reporte HTML.
5. Revisa `resultados.jtl` y deja la ejecución en rojo si alguna petición falló.
6. Sube la carpeta del reporte como artefacto `reporte-rendimiento`, aunque un paso anterior haya fallado.

En el pipeline la API y JMeter comparten la misma máquina virtual, así que la ejecución comprueba que la prueba corre de principio a fin, no la capacidad de la API.

## Experimentación

`pruebas/experimentacion/Experimentacion.jmx` reúne un grupo de hilos por escenario, todos deshabilitados menos uno. Para correr un escenario se habilita solo ese grupo. Algunos escenarios usan elementos de JMeter Plugins (Ultimate Thread Group, Transactions per Second y PerfMon), por lo que este plan se ejecuta solo en local y el pipeline no lo usa.
