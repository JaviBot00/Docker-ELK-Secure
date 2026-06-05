# 03 — Logstash

Intermediario entre Filebeat y Elasticsearch. Recibe los eventos, decide
a qué índice van, los parsea y los enriquece antes de escribirlos.

---

## Fichero de configuración

```cmd
~/docker-elk-secure/logstash/pipeline/logstash.conf
```

Después de cualquier cambio en este fichero:

```bash
docker compose restart logstash
docker compose logs -f logstash   # verificar que arranca sin errores
```

---

## Estructura del pipeline

```cmd
INPUT           FILTER                          OUTPUT
──────    ──────────────────────────────    ──────────────────
:5044  ──► normalización de campos      ──► Elasticsearch
:50000 ──► enrutamiento de índice           índice dinámico:
           parseo grok (auth, apache)        %{target_index}-%{fecha}
           parseo JSON (docker)
           campo severity automático
```

---

## Inputs

**Puerto 5044** — recibe eventos de todos los agentes Filebeat.

**Puerto 50000** — input TCP genérico con codec JSON. Para aplicaciones
que quieran enviar logs directamente sin pasar por Filebeat:

```bash
echo '{"message": "deploy completado", "app_name": "miapp", "log_category": "miapp"}' \
  | nc 10.20.173.250 50000
```

---

## Enrutamiento de índices

Logstash lee los campos `app_name` y `log_category` que manda Filebeat
y decide el nombre del índice usando `[@metadata][target_index]`.
Los campos `@metadata` son internos — no se guardan en Elasticsearch.

### Lógica de enrutamiento (por orden de prioridad)

```cmd
1. input.type == "container"           →  docker-YYYY.MM.dd
2. app_name presente (cualquier valor) →  apps-{app_name}-YYYY.MM.dd
3. log_category en lista de sistema    →  sistema-YYYY.MM.dd
4. cualquier otra cosa                 →  logstash-YYYY.MM.dd  (fallback)
```

### Por qué Docker va primero

El processor `add_docker_metadata` de Filebeat añade el tag `docker` a **todos**
los eventos del host — no solo a los de contenedores. Si se usara `"docker" in [tags]`
para detectar Docker, todos los logs del host irían al índice `docker-*`.
Usando `[input][type] == "container"` se identifica solo los eventos
que realmente vienen de un input de tipo container.

### Añadir un nuevo tipo de índice

Por ejemplo, separar las bases de datos en su propio índice:

```ruby
# En logstash.conf, dentro del bloque filter, antes del else final:
else if [log_category] in ["mysql", "postgresql", "mongodb"] {
  mutate {
    add_field => { "[@metadata][target_index]" => "bbdd" }
  }
}
# Resultado: bbdd-2026.06.05
```

En Filebeat, asegurarse de que ese input tiene `log_category` con el valor correcto:

```yaml
fields:
  log_category: "mysql"
  app_name: "mysql"
fields_under_root: true
```

### Separar por servidor cuando el volumen lo justifica

Si un servidor genera muchos más logs que los demás y ralentiza las búsquedas,
se puede incluir `server_name` en el nombre del índice:

```ruby
# Sistema separado por servidor
else if [log_category] in ["syslog", "kernel", "auth", ...] {
  mutate {
    add_field => { "[@metadata][target_index]" => "sistema-%{server_name}" }
  }
}
# Resultado: sistema-SENTRY-2026.06.05, sistema-WEBPROD-2026.06.05
```

Esto también permite políticas ILM distintas por servidor
(`sistema-SENTRY-*` retención 90 días, `sistema-staging-*` retención 15 días).

---

## Parseo con Grok

Grok extrae campos estructurados del texto libre del log.
Solo se aplica a ciertos tipos de log — el resto pasa sin parsear.

### auth.log — SSH y sudo

Detecta patrones de autenticación y extrae campos útiles para alertas de seguridad:

```cmd
# Login SSH fallido → src_ip, ssh_user
Mar 15 10:23:41 sentry sshd[1234]: Failed password for root from 45.33.32.156

# Login SSH exitoso → src_ip, ssh_user
Mar 15 10:23:41 sentry sshd[1234]: Accepted password for deploy from 10.20.1.5

# Ejecución con sudo → sudo_user, sudo_command
Mar 15 10:23:41 sentry sudo: deploy : TTY=pts/0 ; COMMAND=/bin/systemctl restart nginx
```

Campos resultantes en Kibana: `src_ip`, `ssh_user`, `sudo_user`, `sudo_command`.

Si el mensaje no encaja con ningún patrón, el log pasa igualmente — no se descarta.
El tag `_grokparsefailure` está desactivado para no ensuciar los datos.

### Apache — Combined Log Format

```cmd
192.168.1.1 - frank [10/Oct/2026:13:55:36 +0000] "GET /index.html HTTP/1.1" 200 2326
```

Campos resultantes: `clientip`, `request`, `response` (integer), `bytes`, `agent`.

---

## Campo severity automático

Logstash analiza el texto de **todos** los mensajes y añade `severity`:

| Valor | Palabras clave detectadas (sin distinguir mayúsculas) |
|---|---|
| `critical` | critical, fatal, emergency |
| `error` | error, exception, traceback, failed, failure |
| `warning` | warn, warning, deprecated |
| `info` | cualquier mensaje que no encaje con los anteriores |

Permite filtrar en Kibana con `severity : "error"` sin importar el origen ni
el formato que use cada aplicación para indicar el nivel.

Si una app ya manda su propio campo `level` o `log_level`, se puede adaptar
el pipeline para usarlo cuando exista y caer al regex solo cuando no exista.

---

## Variable ELASTIC_PASSWORD

Logstash lee la contraseña de Elasticsearch del entorno del contenedor Docker.
Para que funcione, el servicio `logstash` en `docker-compose.yml` debe tener:

```yaml
services:
  logstash:
    environment:
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
```

Sin esta línea, Logstash intentará conectarse con la cadena literal
`${ELASTIC_PASSWORD}` y fallará con error 401.

Verificar que la variable llega correctamente:

```bash
docker compose exec logstash env | grep ELASTIC_PASSWORD
```

---

## Ampliar el pipeline

### Parsear logs JSON de una aplicación

Si una app manda logs en formato JSON dentro del campo `message`:

```ruby
filter {
  if [app_name] == "miapp" {
    json {
      source => "message"
      target => "app"          # los campos JSON quedan bajo app.campo
    }
  }
}
```

### Descartar logs que generan ruido

```ruby
filter {
  if [message] =~ /health.?check|GET \/ping|GET \/status/ {
    drop {}
  }
}
```

### Añadir un campo calculado

```ruby
filter {
  if [src_ip] =~ /^(10\.|192\.168\.|172\.(1[6-9]|2\d|3[01])\.)/ {
    mutate { add_field => { "ip_type" => "privada" } }
  } else if [src_ip] {
    mutate { add_field => { "ip_type" => "publica" } }
  }
}
```

### Debug en tiempo real

Para ver exactamente qué campos tiene cada evento que procesa Logstash,
añadir temporalmente al output (solo en desarrollo, nunca en producción):

```ruby
output {
  stdout { codec => rubydebug }   # ← ver en docker compose logs logstash
  elasticsearch { ... }           # mantener el output normal
}
```

Acordarse de quitarlo después y reiniciar Logstash.

---

## Verificar el estado del pipeline

```bash
# Eventos procesados (in = recibidos, out = escritos en ES)
curl -s http://localhost:9600/_node/stats/pipeline | python3 -m json.tool | grep -A5 '"events"'

# Logs del contenedor
docker compose logs -f logstash

# Solo errores
docker compose logs logstash | grep -i "error\|warn\|exception" | tail -20

# Ver índices creados hoy
curl -s -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/_cat/indices?v&s=index" | grep "$(date +%Y.%m.%d)"
```

---

## Tabla de enrutamiento completa

| Condición evaluada en orden | Índice resultante |
|---|---|
| `input.type == "container"` | `docker-YYYY.MM.dd` |
| `app_name` presente (cualquier valor) | `apps-{app_name}-YYYY.MM.dd` |
| `log_category` = syslog, kernel, auth, cron, paquetes, vmware, apache, php-fpm | `sistema-YYYY.MM.dd` |
| Cualquier otra cosa | `logstash-YYYY.MM.dd` |

El orden importa — la primera condición que se cumple gana y las demás no se evalúan.

---

## Troubleshooting de Logstash

### Índices creados como logstash-* en lugar de apps-* o sistema-*

Los campos `app_name` o `log_category` no llegan al nivel raíz.
Diagnóstico:

```bash
curl -s -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/logstash-*/_search?pretty&size=1" \
  | grep -E '"app_name"|"log_category"|"fields"'
```

Si ves `"fields": { "app_name": "..." }` en lugar de `"app_name": "..."` directamente,
falta `fields_under_root: true` en el bloque del input en `filebeat.yml`.

### Error de autenticación con Elasticsearch (401)

```bash
# Ver si la variable llega al contenedor
docker compose exec logstash env | grep ELASTIC_PASSWORD
```

Si no aparece, falta en `docker-compose.yml`:

```yaml
services:
  logstash:
    environment:
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
```

Aplicar con `docker compose up -d --force-recreate logstash`.

### Logstash arranca antes que Elasticsearch

```bash
docker compose restart logstash
```

Reintentará la conexión automáticamente en cuanto Elasticsearch esté listo.

### Ver exactamente qué eventos procesa Logstash

Añadir temporalmente al bloque `output` en `logstash.conf` — solo para debug,
nunca dejar en producción:

```ruby
output {
  stdout { codec => rubydebug }
  elasticsearch { ... }
}
```

```bash
docker compose restart logstash
docker compose logs -f logstash
```

Quitar el `stdout` y reiniciar de nuevo cuando termines.
