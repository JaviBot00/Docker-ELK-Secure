# 📊 Kibana — Configuración y uso con Filebeat + Logstash

---

## ⚠️ Importante: Fleet/Integrations NO es para Filebeat

Si intentaste configurar algo en **Fleet** o **Integrations**, ignora todo lo que hiciste ahí.
Esos menús son para **Elastic Agent**, que es un sistema distinto e incompatible con Filebeat clásico.

Los errores como `beats_stats.beat.type was not found` son exactamente la señal de estar
mezclando los dos sistemas. **Filebeat no necesita Fleet ni ninguna integración.**

El flujo real es:

```cmd
Filebeat (cliente)
    │  campos: server_name, environment, log_category, app_name, tags
    ▼  puerto 5044
Logstash (Docker)
    │  lee log_category / app_name → decide el nombre del índice
    │  añade campo "severity" automático
    ▼
Elasticsearch
    │  índices: apps-{nombre}-*, sistema-*, docker-*, logstash-*
    ▼
Kibana → Data Views → Discover / Dashboard
```

---

## Paso 1 — Verificar índices en Elasticsearch

Antes de configurar nada en Kibana, confirma que hay datos. Ejecuta en el servidor Docker:

```bash
# Ver todos los índices creados por el pipeline
curl -s -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/_cat/indices?v&h=index,docs.count,store.size&s=index"
```

Deberías ver algo así:

```cmd
index                         docs.count   store.size
apps-apitest-2024.03.15           1823         890kb
apps-gitea-2024.03.15              412         210kb
docker-2024.03.15                 9201          4.1mb
sistema-2024.03.15                3847          1.8mb
```

Si no aparece ningún índice, el problema está antes de Kibana. Ve a la sección
de troubleshooting al final de este documento.

---

## Paso 2 — Crear los Data Views en Kibana

Un Data View le dice a Kibana qué índices de Elasticsearch mirar.
Crea los siguientes (los que necesites según lo que uses):

**Ruta:** Stack Management ⚙️ → Data Views → Create data view

| Nombre sugerido   | Index pattern                          | Timestamp field  | Para qué                       |
|---|---|---|---|
| Todas las apps    | `apps-*`                               | `@timestamp`     | Ver todas las aplicaciones     |
| Sistema           | `sistema-*`                            | `@timestamp`     | Auth, syslog, kernel, cron...  |
| Docker            | `docker-*`                             | `@timestamp`     | Contenedores de todos los hosts|
| Vista global      | `apps-*,sistema-*,docker-*,logstash-*` | `@timestamp`     | Todo a la vez                  |
| Solo apitest      | `apps-apitest-*`                       | `@timestamp`     | Una sola aplicación            |

> Para la vista global, escribe el patrón con comas directamente en el campo
> "Index pattern" — Kibana lo soporta de forma nativa.
> Si `@timestamp` no aparece en el desplegable, es que aún no hay índices.
> Vuelve al Paso 1.

---

## Paso 3 — Ver logs en Discover

1. Menú lateral → **Discover** 🧭
2. Selector de Data View (arriba a la izquierda) → elige el que quieras
3. Rango temporal (arriba a la derecha) → prueba **Last 24 hours** o **Last 7 days**
4. Los logs aparecen en la tabla central

### Campos disponibles en Discover

Estos son los campos que genera este stack. Aparecen en el panel izquierdo de Discover
y puedes añadirlos como columnas en la tabla.

| Campo                  | Origen       | Qué contiene                                      |
|---|---|---|
| `@timestamp`           | Filebeat     | Fecha y hora del log                              |
| `message`              | Filebeat     | Contenido del log en texto                        |
| `host.name`            | Filebeat     | Hostname del servidor cliente                     |
| `server_name`          | filebeat.yml | Nombre personalizado del servidor (`fields:`)     |
| `environment`          | filebeat.yml | production / staging / development                |
| `location`             | filebeat.yml | Zona, datacenter, proveedor cloud                 |
| `log_category`         | filebeat.yml | Tipo de log: auth, syslog, docker, apitest...     |
| `app_name`             | filebeat.yml | Nombre de la app (solo bloques de aplicación)     |
| `tags`                 | filebeat.yml | Array de etiquetas: ["app","sentry-project"]      |
| `severity`             | Logstash     | critical / error / warning / info (automático)    |
| `log.file.path`        | Filebeat     | Ruta del fichero de log en el cliente             |
| `input.type`           | Filebeat     | log / container                                   |
| `container.name`       | Filebeat     | Nombre del contenedor Docker (solo tipo container)|
| `container.image.name` | Filebeat     | Imagen del contenedor Docker                      |
| `src_ip`               | Logstash     | IP origen en logs de auth/SSH (si aplica grok)    |
| `ssh_user`             | Logstash     | Usuario en intentos SSH (si aplica grok)          |

---

## Paso 4 — Filtrar con KQL (Kibana Query Language)

La barra de búsqueda de Discover acepta KQL. Algunos ejemplos útiles:

```kql
# Todos los errores de todas las fuentes
severity : "error"

# Solo críticos
severity : "critical"

# Logs de un servidor concreto
server_name : "SENTRY"

# Una aplicación concreta
app_name : "apitest"

# Todas las apps de un entorno
environment : "test" and tags : "app"

# Intentos fallidos de SSH
log_category : "auth" and message : "Failed password"

# Logs de un contenedor Docker concreto
container.name : "nginx-proxy"

# Errores en producción, cualquier servidor
environment : "production" and severity : "error"

# Todo excepto logs informativos
not severity : "info"

# Combinar servidor + categoría + nivel
server_name : "SENTRY" and log_category : "auth" and severity : "error"
```

### Guardar búsquedas frecuentes

1. Aplica el filtro en Discover
2. Pulsa **Save** (arriba) → dale un nombre descriptivo
3. Queda disponible en **Saved Searches** para acceso rápido
4. También puedes añadir búsquedas guardadas a un Dashboard

---

## Paso 5 — Crear un Dashboard

### Crear el dashboard base

1. Menú lateral → **Dashboard** → **Create dashboard**
2. Pulsa **Create visualization** para añadir cada panel

### Panel 1 — Volumen de logs por tiempo

Detecta picos de actividad o caídas de logs.

- Tipo: **Bar vertical stacked** o **Line**
- Eje X: `@timestamp` (date histogram, intervalo Auto)
- Eje Y: Count of records
- Break down by: `log_category` (Top 10)

### Panel 2 — Distribución por severidad

Proporción de errores vs info en un vistazo.

- Tipo: **Pie** o **Donut**
- Slice by: `severity` (Top 5 values)

### Panel 3 — Top aplicaciones por volumen de logs

Ver qué app genera más logs.

- Tipo: **Bar horizontal**
- Eje Y: `app_name` (Top 15 values)
- Eje X: Count

### Panel 4 — Errores recientes (tabla)

Lista de los últimos errores con detalle.

- Tipo: **Data table**
- Filtro KQL en el panel: `severity : "error" or severity : "critical"`
- Columnas: `@timestamp`, `server_name`, `app_name`, `message`
- Ordenar: `@timestamp` descendente

### Panel 5 — Intentos de acceso SSH fallidos

Seguridad — detectar ataques de fuerza bruta.

- Tipo: **Data table** o **Bar**
- Filtro KQL: `log_category : "auth" and message : "Failed password"`
- Columnas: `@timestamp`, `src_ip`, `ssh_user`, `host.name`

### Panel 6 — Contador de logs por servidor

Métrica rápida de actividad por máquina.

- Tipo: **Metric**
- Break down by: `server_name`

---

## Paso 6 — Alertas (opcional pero recomendado)

Kibana puede notificarte cuando detecta algo anómalo.

**Stack Management → Rules → Create rule → Elasticsearch query**

### Alerta: pico de errores

```json
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "severity": "error" } }
      ]
    }
  }
}
```

- Threshold: más de 50 en 5 minutos
- Acción: email o Slack webhook

### Alerta: intentos SSH fallidos

```json
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "log_category": "auth" } },
        { "match": { "message": "Failed password" } }
      ]
    }
  }
}
```

- Threshold: más de 10 en 1 minuto
- Indica posible ataque de fuerza bruta

### Alerta: un servidor deja de enviar logs

Usa el tipo de regla **Metrics threshold** con `host.name` — si el contador
de documentos de un servidor cae a 0 durante X minutos, algo va mal
(el servidor está caído o Filebeat ha fallado).

---

## Referencia rápida de rutas en Kibana

| Qué hacer                     | Ruta en Kibana                                           |
|---|---|
| Ver logs en tiempo real       | Discover → selecciona Data View → ajusta tiempo          |
| Crear/gestionar Data Views    | Stack Management → Data Views                            |
| Ver y crear dashboards        | Dashboard                                                |
| Crear alertas                 | Stack Management → Rules                                 |
| Estado de índices             | Stack Management → Index Management                      |
| Políticas ILM (retención)     | Stack Management → Index Lifecycle Policies              |
| Ver mappings de campos        | Stack Management → Index Management → índice → Mappings  |
| Búsquedas guardadas           | Stack Management → Saved Objects                         |

---

## 🛠️ Troubleshooting

### No aparecen índices en Elasticsearch

```bash
# Ver todos los índices existentes
curl -s -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/_cat/indices?v&s=index"

# Ver si Logstash está procesando eventos
docker compose logs logstash | grep -i "events\|error\|pipeline"

# Ver el estado del pipeline de Logstash en tiempo real
curl -s http://localhost:9600/_node/stats/pipeline | python3 -m json.tool
```

### Los índices se crean como `logstash-*` en lugar de `apps-*` / `sistema-*`

El campo `app_name` o `log_category` no está llegando correctamente desde Filebeat.
Verifica que `fields_under_root: true` está en el `filebeat.yml` del cliente y
que los bloques de input tienen el campo `fields:` correctamente indentado.

```bash
# Ver qué campos están llegando realmente a Logstash
docker compose logs logstash | grep -i "app_name\|log_category"
```

### Discover no muestra logs aunque el índice existe

- Revisa el **rango temporal** — puede que los datos sean de otro día.
- Verifica que el **Data View** tiene el patrón correcto (con asterisco).
- Confirma que el **Timestamp field** es `@timestamp`.

### En el cliente: Filebeat no conecta o no manda datos

```bash
# Test completo de configuración y conectividad
sudo filebeat test config -e
sudo filebeat test output -e

# Ver logs del agente
sudo journalctl -u filebeat -f

# Probar conectividad manual al puerto de Logstash
nc -zv IP_DEL_SERVIDOR_ELK 5044
```

| Error en Filebeat                      | Causa probable              | Solución                                        |
|----------------------------------------|-----------------------------|-------------------------------------------------|
| `connection refused :5044`             | Firewall o Logstash caído   | Abre el puerto 5044; reinicia Logstash          |
| `No such file or directory`            | Ruta de log no existe       | Verifica el path en el cliente                  |
| `permission denied`                    | Sin permisos de lectura     | `sudo chmod o+r /var/log/fichero.log`           |
| `version mismatch`                     | Versiones distintas         | Usa la misma versión 8.x en Filebeat y el stack |
