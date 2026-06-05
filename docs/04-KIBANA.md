# 04 — Kibana

Interfaz web para buscar, visualizar y crear alertas sobre los logs.
Acceso: `http://IP_SERVIDOR:5601` — usuario `elastic`.

---

## ⚠️ Fleet e Integrations NO son para Filebeat

Si ves los menús **Fleet** o **Integrations** en Kibana, son para **Elastic Agent**,
que es un sistema distinto e incompatible con Filebeat clásico.

Errores como `beats_stats.beat.type was not found` o `beats_stats.beat.uuid was not found`
son la señal de haber mezclado los dos sistemas. No tocar nada de Fleet ni Integrations.

---

## Primer paso — Crear Data Views

Un Data View le dice a Kibana qué índices de Elasticsearch mirar.
Hay que crearlos una vez y ya están disponibles para siempre.

**Ruta:** Stack Management ⚙️ → Data Views → Create data view

### Data Views recomendados

| Nombre | Index pattern | Para qué |
|---|---|---|
| Todas las apps | `apps-*` | Ver todas las aplicaciones juntas |
| Sistema | `sistema-*` | syslog, auth, kernel, cron, nginx... |
| Docker | `docker-*` | Contenedores de todos los hosts |
| Vista global | `apps-*,sistema-*,docker-*,logstash-*` | Todo a la vez |
| Una app concreta | `apps-apitest-*` | Solo una aplicación |

En todos: **Timestamp field → `@timestamp`**

> El patrón con comas (`apps-*,sistema-*,...`) es multi-índice nativo de Kibana.
> No hace falta ningún plugin — se escribe directamente en el campo Index pattern.

Si `@timestamp` no aparece en el desplegable al crear el Data View,
es que todavía no hay índices con ese patrón. Verificar con:

```bash
curl -s -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/_cat/indices?v&s=index&h=index,docs.count"
```

---

## Discover — buscar logs

**Ruta:** menú lateral → Discover 🧭

1. Selector de Data View (arriba izquierda) → elegir el que corresponda
2. Rango temporal (arriba derecha) → ajustar la ventana de tiempo
3. Barra de búsqueda → escribir filtros KQL
4. Tabla central → los eventos que coinciden

### Columnas útiles para añadir a la tabla

Click en el nombre del campo en el panel izquierdo → **+** para añadirlo como columna:
`server_name`, `log_category`, `severity`, `app_name`, `message`

### KQL — referencia rápida

KQL (Kibana Query Language) es el lenguaje de filtrado de Discover.

```kql
# Por servidor
server_name : "SENTRY"
server_name : "SENTRY" or server_name : "WEBPROD"

# Por entorno
environment : "production"
not environment : "test"

# Por severidad
severity : "error"
severity : "critical" or severity : "error"
not severity : "info"

# Por tipo de log
log_category : "auth"
log_category : "nginx" or log_category : "apache"
tags : "seguridad"

# Buscar texto en el mensaje
message : "error"
message : "Failed password"
message : "Out of memory"
message : "Connection refused"

# Campos extraídos por Logstash (auth.log)
src_ip : "45.33.32.156"
ssh_user : "root"

# Contenedores Docker
container.name : "nginx-proxy"

# Combinaciones
server_name : "SENTRY" and severity : "error"
environment : "production" and log_category : "auth" and message : "Failed password"
server_name : "SENTRY" and severity : "error" and not log_category : "docker"

# Rango de fechas dentro de la query
@timestamp >= "2026-06-01" and @timestamp <= "2026-06-05"
```

### Casos de uso habituales

**Ver todos los errores de producción de hoy:**
- Data View: `apps-*,sistema-*`
- KQL: `environment : "production" and severity : "error"`
- Rango: Last 24 hours

**Investigar un incidente a una hora concreta:**
- Rango temporal → personalizado, ventana del incidente ± 30 minutos
- KQL: `server_name : "servidor-afectado"`
- Ordenar por `@timestamp` ascendente para ver la secuencia de eventos

**Ver intentos de login SSH fallidos:**
- Data View: `sistema-*`
- KQL: `log_category : "auth" and message : "Failed password"`
- Columnas: `src_ip`, `ssh_user`, `host.name`, `@timestamp`

**Monitorizar una app durante un deploy:**
- Data View: `apps-nombreapp-*`
- KQL: `severity : "error" or severity : "critical"`
- Activar **auto-refresh** (arriba derecha) cada 10 segundos

### Guardar búsquedas frecuentes

Discover → **Save** → nombre descriptivo.
Las búsquedas guardadas se pueden añadir a dashboards y acceder desde
Stack Management → Saved Objects.

---

## Índice vs filtro — cuándo usar cada uno

Esta es una decisión importante. No hay una respuesta universal.

### Usar filtros KQL (lo que tenemos ahora)

Mejor cuando:
- Pocos servidores (menos de 10-15)
- Quieres comparar servidores entre sí en el mismo dashboard
- Los servidores tienen volúmenes similares
- No necesitas retención distinta por servidor

```kql
server_name : "SENTRY" and log_category : "auth"
environment : "production" and severity : "error"
```

### Separar por índice

Mejor cuando:
- **Volúmenes muy distintos** entre servidores — un servidor con 10M logs/día
  no debería compartir índice con uno de 10K. Las búsquedas del pequeño se penalizan.
- **Retención distinta** — producción 90 días, staging 15 días. Las políticas ILM
  van por patrón de índice, no por campo.
- **Control de acceso** — los permisos en Elasticsearch van por índice. Si el equipo A
  solo puede ver los logs del servidor A, necesitan índices separados.

Para separar por servidor, cambiar en `logstash.conf`:

```ruby
# De:
add_field => { "[@metadata][target_index]" => "sistema" }
# A:
add_field => { "[@metadata][target_index]" => "sistema-%{server_name}" }
# Resultado: sistema-SENTRY-2026.06.05, sistema-WEBPROD-2026.06.05
```

Luego crear los Data Views correspondientes en Kibana.

---

## Dashboards

**Ruta:** menú lateral → Dashboard → Create dashboard → Create visualization

### Panel 1 — Volumen de logs por tiempo

Para detectar picos de actividad o caídas de logs.

- Tipo: **Bar vertical stacked**
- Eje X: `@timestamp` (date histogram, intervalo Auto)
- Eje Y: Count
- Break down by: `log_category` o `server_name` (Top 10)

### Panel 2 — Distribución por severidad

Para ver la proporción de errores de un vistazo.

- Tipo: **Donut**
- Slice by: `severity` (Top 5)

### Panel 3 — Top apps por volumen

Para ver qué aplicación genera más logs.

- Tipo: **Bar horizontal**
- Eje Y: `app_name` (Top 15 values)
- Eje X: Count

### Panel 4 — Tabla de últimos errores

Para tener siempre a la vista los últimos errores.

- Tipo: **Data table**
- Filtro KQL del panel: `severity : "error" or severity : "critical"`
- Columnas: `@timestamp`, `server_name`, `app_name`, `message`
- Ordenar: `@timestamp` descendente

### Panel 5 — Intentos SSH fallidos

Para monitorización de seguridad.

- Tipo: **Data table**
- Filtro KQL del panel: `log_category : "auth" and message : "Failed password"`
- Columnas: `@timestamp`, `src_ip`, `ssh_user`, `host.name`

### Panel 6 — Contador de logs por servidor

Para ver actividad relativa entre servidores.

- Tipo: **Metric** o **Bar horizontal**
- Break down by: `server_name`

---

## Alertas

**Ruta:** Stack Management → Rules → Create rule

### Alerta — errores críticos en producción

- Tipo: **Elasticsearch query**
- Query:
```json
{
  "bool": {
    "filter": [
      { "term": { "severity": "critical" } },
      { "term": { "environment": "production" } }
    ]
  }
}
```
- Threshold: más de 5 en 10 minutos
- Acción: email o Slack webhook

### Alerta — servidor deja de enviar logs

Si un servidor está caído o Filebeat falló, el flujo de logs se corta.

- Tipo: **Elasticsearch query count**
- Query: `{ "term": { "server_name": "SENTRY" } }`
- Condición: count = 0 durante los últimos 15 minutos
- Esto indica que el servidor no responde o Filebeat está caído

### Alerta — posible ataque de fuerza bruta SSH

- Tipo: **Elasticsearch query**
- Query:
```json
{
  "bool": {
    "filter": [
      { "term": { "log_category": "auth" } },
      { "match": { "message": "Failed password" } }
    ]
  }
}
```
- Threshold: más de 20 en 5 minutos
- Acción: email urgente o PagerDuty

---

## Referencia de rutas en Kibana

| Tarea | Ruta |
|---|---|
| Ver logs en tiempo real | Discover → selecciona Data View |
| Crear / gestionar Data Views | Stack Management → Data Views |
| Ver y crear dashboards | Dashboard |
| Crear alertas | Stack Management → Rules |
| Estado de índices y tamaño | Stack Management → Index Management |
| Políticas de retención ILM | Stack Management → Index Lifecycle Policies |
| Búsquedas guardadas | Stack Management → Saved Objects |
| Mappings y campos de un índice | Stack Management → Index Management → índice → Mappings |
