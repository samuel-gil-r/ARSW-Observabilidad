# Laboratorio de Observabilidad — ARSW

Observabilidad de Microservicios con **Grafana, Prometheus, Loki y OpenTelemetry**.

- **Asignatura:** Arquitecturas de Software (ARSW)
- **Tecnologías:** Spring Boot 3, Java 17, Docker Compose, Prometheus, Grafana, Loki, Promtail
- **Aplicación:** `arsw-observability-lab/` — servicio de pedidos (`observability-demo`) instrumentado con Actuator + Micrometer

---

## 1. Requisitos previos

**Qué pedía el laboratorio:** tener instalados Docker, Docker Compose, Java 17+, Maven 3.8+ y un cliente HTTP (curl/Postman).

**Objetivo:** garantizar que el entorno local puede compilar la aplicación y levantar los contenedores de observabilidad.

**Cómo se verifica:**

```powershell
docker --version
docker compose version
java --version
mvn --version
```

---

## 2. Aplicación Spring Boot instrumentada

**Qué pedía el laboratorio:** crear un proyecto Spring Boot (`edu.eci.arsw / observability-demo`) con las dependencias Spring Web, Actuator, Micrometer Registry Prometheus y Validation; configurar Actuator para exponer `health`, `info`, `metrics` y `prometheus`; crear un `OrderController` con métricas de negocio y un `GlobalExceptionHandler`.

**Objetivo:** que la aplicación produzca señales observables: métricas técnicas (HTTP, JVM, CPU) vía Actuator/Micrometer, métricas de negocio propias (`orders_created_total`, `orders_failed_total`) y logs estructurados, incluyendo endpoints que simulan incidentes (latencia y errores).

**Endpoints de la aplicación (puerto 8081):**

| Endpoint | Método | Qué hace |
|---|---|---|
| `/orders` | POST | Crea un pedido e incrementa `orders_created_total` |
| `/orders/{id}` | GET | Consulta un pedido (log nivel DEBUG) |
| `/orders/simulate-latency` | GET | Responde con demora aleatoria de 500–3000 ms (log WARN) |
| `/orders/simulate-error` | GET | Lanza un error 500 e incrementa `orders_failed_total` (log ERROR) |
| `/actuator/health` | GET | Estado del servicio |
| `/actuator/prometheus` | GET | Métricas en formato Prometheus |

**Cómo se ejecuta:**

```powershell
cd C:\Users\timon\Observabilidad\arsw-observability-lab
mvn spring-boot:run
```

> Nota: la aplicación además escribe sus logs en `logs/observability-demo.log`, archivo que Promtail lee para enviarlos a Loki.

---

## 3. Prueba inicial de la aplicación

**Qué pedía el laboratorio:** verificar que el servicio responde y que expone métricas en formato Prometheus.

**Objetivo:** confirmar que la instrumentación funciona antes de conectar Prometheus: el health check responde `UP` y el endpoint `/actuator/prometheus` entrega métricas en texto plano (`http_server_requests_seconds_count`, `jvm_memory_used_bytes`, `process_cpu_usage`, `orders_created_total`, `orders_failed_total`).

**Cómo se ejecuta:**

```powershell
curl.exe http://localhost:8081/actuator/health
curl.exe http://localhost:8081/actuator/prometheus
```

**Evidencia:**

![Health check UP](docs/img/01-health.png)

![Métricas en /actuator/prometheus](docs/img/02-actuator-prometheus.png)

---

## 4. Generación de tráfico

**Qué pedía el laboratorio:** ejecutar varias solicitudes contra los endpoints para generar datos observables.

**Objetivo:** producir métricas y logs reales (pedidos creados, consultas, latencias y errores) que luego se analizan en Prometheus, Grafana y Loki.

**Cómo se ejecuta (PowerShell):**

```powershell
Invoke-RestMethod -Uri http://localhost:8081/orders -Method Post -ContentType "application/json" -Body '{"customerId":"CUS-01","total":120000}'
Invoke-RestMethod http://localhost:8081/orders/ORD-1001
Invoke-RestMethod http://localhost:8081/orders/simulate-latency
curl.exe http://localhost:8081/orders/simulate-error
```

Repita cada comando varias veces para acumular datos.

**Evidencia:**

![Tráfico generado](docs/img/03-trafico.png)

---

## 5. Entorno de observabilidad con Docker Compose

**Qué pedía el laboratorio:** definir un `docker-compose.yml` que levante Prometheus, Grafana, Loki y Promtail, y verificar que los cuatro contenedores quedan corriendo.

**Objetivo:** disponer de toda la pila de observabilidad con un solo comando: Prometheus recolecta métricas, Loki almacena logs, Promtail los envía y Grafana los visualiza.

**Cómo se ejecuta:**

```powershell
cd C:\Users\timon\Observabilidad\arsw-observability-lab
docker compose up -d
docker ps
```

Deben aparecer: `arsw-prometheus`, `arsw-grafana`, `arsw-loki`, `arsw-promtail`.

**Evidencia:**

![Contenedores corriendo](docs/img/04-docker-ps.png)

---

## 6. Prometheus: recolección y verificación

**Qué pedía el laboratorio:** configurar `prometheus/prometheus.yml` para hacer scrape de `/actuator/prometheus` cada 5 segundos, verificar el target en **Status → Targets** y probar consultas básicas.

**Objetivo:** validar que Prometheus está recolectando las métricas de la aplicación: el target `observability-demo` debe aparecer en estado **UP**, y las consultas PromQL deben devolver datos.

**Cómo se verifica:**

1. Abrir <http://localhost:9090>
2. Ir a **Status → Targets** → el target `observability-demo` debe estar **UP**
3. En la pestaña **Graph**, probar consultas:

```text
up
http_server_requests_seconds_count
jvm_memory_used_bytes
orders_created_total
orders_failed_total
```

> Si el target aparece DOWN: revisar que la aplicación esté encendida en el puerto 8081 y que `host.docker.internal` funcione en su sistema.

**Evidencia:**

![Prometheus Targets UP](docs/img/05-prometheus-targets.png)

![Consulta PromQL](docs/img/06-prometheus-query.png)

---

## 7. Loki + Promtail: centralización de logs

**Qué pedía el laboratorio:** configurar Loki (`loki/loki-config.yml`) como almacén de logs y Promtail (`promtail/promtail-config.yml`) como agente que los envía.

**Objetivo:** centralizar los logs de la aplicación para poder consultarlos desde Grafana y correlacionarlos con las métricas al analizar incidentes.

**Cómo funciona en este proyecto:** la aplicación escribe en `logs/observability-demo.log`; esa carpeta se monta en el contenedor de Promtail (`./logs:/var/log/app`), que lee `/var/log/app/*.log` y lo envía a Loki con la etiqueta `job=observability-demo`. No requiere comandos adicionales: se levanta junto con el resto del entorno (`docker compose up -d`).

---

## 8. Grafana y fuentes de datos

**Qué pedía el laboratorio:** ingresar a Grafana y configurar Prometheus y Loki como fuentes de datos.

**Objetivo:** conectar Grafana con las dos fuentes de señales (métricas en Prometheus y logs en Loki) para poder construir el dashboard.

**Cómo se ejecuta:**

1. Abrir <http://localhost:3300> *(en este proyecto Grafana usa el puerto 3300, no el 3000 de la guía)*
2. Credenciales: usuario `admin`, contraseña `admin`

> En este proyecto las fuentes de datos **ya quedan provisionadas automáticamente** (`grafana/provisioning/datasources/datasources.yml`): Prometheus en `http://prometheus:9090` y Loki en `http://loki:3100`. Puede verificarlas en **Connections → Data sources**.

**Evidencia:**

![Data sources en Grafana](docs/img/07-datasources.png)

---

## 9. Dashboard: ARSW - Observabilidad de Microservicios

**Qué pedía el laboratorio:** construir un dashboard con 8 paneles para analizar disponibilidad, tráfico, latencia, errores, métricas de negocio y recursos.

**Objetivo:** tener una vista única del estado del microservicio que permita detectar y diagnosticar problemas. En este proyecto el dashboard **se provisiona automáticamente** (`grafana/dashboards/arsw-observabilidad.json`) e incluye un noveno panel con los logs de error desde Loki.

| # | Panel | Tipo | Consulta | Qué permite analizar |
|---|---|---|---|---|
| 1 | Estado del servicio | Stat | `up{job="observability-demo"}` | Disponibilidad (1 = arriba, 0 = caído) |
| 2 | Solicitudes HTTP por endpoint | Time series | `sum by (uri, method, status) (rate(http_server_requests_seconds_count[1m]))` | Solicitudes/seg por endpoint |
| 3 | Latencia promedio | Time series | `sum(rate(http_server_requests_seconds_sum[1m])) / sum(rate(http_server_requests_seconds_count[1m]))` | Si el tiempo de respuesta aumenta |
| 4 | Errores HTTP 500 | Time series | `sum(rate(http_server_requests_seconds_count{status="500"}[1m]))` | Tasa de errores internos |
| 5 | Pedidos creados | Stat | `orders_created_total` | Actividad de negocio exitosa |
| 6 | Pedidos fallidos | Stat | `orders_failed_total` | Errores en el flujo de pedidos |
| 7 | Memoria usada por JVM | Time series | `sum(jvm_memory_used_bytes{application="observability-demo"})` | Consumo de memoria |
| 8 | Uso de CPU del proceso | Time series | `process_cpu_usage{application="observability-demo"}` | Carga del proceso Java |
| 9 | Logs de error (Loki) | Logs | `{job="observability-demo"} \|= "ERROR"` | Logs de error correlacionados |

**Cómo se abre:** Grafana → **Dashboards** → `ARSW - Observabilidad de Microservicios`.

**Evidencia:**

![Dashboard completo](docs/img/08-dashboard.png)

---

## 10. Exploración de logs con Loki

**Qué pedía el laboratorio:** consultar los logs centralizados desde Grafana (Explore) usando LogQL.

**Objetivo:** buscar y filtrar logs para explicar incidentes: errores, actividad de pedidos y latencias artificiales.

**Cómo se ejecuta:** en Grafana ir a **Explore**, seleccionar la fuente **Loki** y probar:

```text
{job="observability-demo"}
{job="observability-demo"} |= "ERROR"
{job="observability-demo"} |= "Pedido"
{job="observability-demo"} |= "latencia"
```

> Nota: la guía usa `{job="docker"}`; en este proyecto la etiqueta es `job="observability-demo"` porque Promtail lee directamente el archivo de log de la aplicación.

**Evidencia:**

![Logs en Loki Explore](docs/img/09-loki-explore.png)

---

## 11. Simulación de incidentes

**Qué pedía el laboratorio:** generar tres incidentes controlados y observar su efecto en métricas, dashboard y logs.

### Incidente 1: aumento de errores

**Objetivo:** ver cómo un error interno se refleja en la tasa de HTTP 500, el contador `orders_failed_total` y los logs de nivel ERROR.

```powershell
curl.exe http://localhost:8081/orders/simulate-error
```

Observar: panel *Errores HTTP 500*, panel *Pedidos fallidos*, logs con `|= "ERROR"`.

![Incidente de errores](docs/img/10-incidente-errores.png)

### Incidente 2: aumento de latencia

**Objetivo:** ver cómo se degrada la latencia promedio y confirmarlo con los logs WARN de latencia artificial.

```powershell
Invoke-RestMethod http://localhost:8081/orders/simulate-latency
```

Observar: panel *Latencia promedio*, logs con `|= "latencia"`, tiempo de respuesta del cliente.

![Incidente de latencia](docs/img/11-incidente-latencia.png)

### Incidente 3: creación de pedidos

**Objetivo:** ver la actividad normal de negocio: aumento de solicitudes HTTP, del contador de pedidos creados y de los logs de creación.

```powershell
Invoke-RestMethod -Uri http://localhost:8081/orders -Method Post -ContentType "application/json" -Body '{"customerId":"CUS-01","total":120000}'
```

Observar: panel *Solicitudes HTTP*, panel *Pedidos creados*, logs con `|= "Pedido"`.

![Actividad de pedidos](docs/img/12-incidente-pedidos.png)

---

## 12. Alertas propuestas

**Qué pedía el laboratorio:** proponer tres alertas básicas sobre las consultas del dashboard.

**Objetivo:** pasar de observar a reaccionar: definir condiciones que avisen automáticamente cuando el servicio se degrada.

| Alerta | Condición (PromQL) | Interpretación |
|---|---|---|
| Servicio caído | `up{job="observability-demo"} == 0` | Prometheus no puede consultar la aplicación |
| Errores HTTP 500 | `sum(rate(http_server_requests_seconds_count{status="500"}[1m])) > 0` | La aplicación está generando errores internos |
| Latencia elevada | `(sum(rate(http_server_requests_seconds_sum[1m])) / sum(rate(http_server_requests_seconds_count[1m]))) > 1` | La latencia promedio supera un segundo |

**Evidencia (opcional, si se configuran en Grafana):**

![Alertas en Grafana](docs/img/13-alertas.png)

---


## Resumen de ejecución completa

```powershell
cd C:\Users\timon\Observabilidad\arsw-observability-lab
docker compose up -d
docker ps
mvn spring-boot:run
```

En otra terminal, generar tráfico (sección 4) y abrir:

| Herramienta | URL |
|---|---|
| Aplicación | <http://localhost:8081/actuator/health> |
| Prometheus | <http://localhost:9090> |
| Grafana | <http://localhost:3300> (admin / admin) |
| Loki (API) | <http://localhost:3100> |

Para apagar el entorno:

```powershell
docker compose down
```
