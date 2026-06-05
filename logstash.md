# ⚙️ Logstash — Pipeline, enrutamiento y parseo

Logstash es el intermediario entre Filebeat y Elasticsearch. Su función en este stack es:

1. **Recibir** los logs que mandan los agentes Filebeat (puerto 5044)
2. **Enrutar** cada log al índice de Elasticsearch correcto según su tipo
3. **Parsear** ciertos formatos conocidos (auth, Apache) para extraer campos estructurados
4. **Enriquecer** todos los logs con el campo `severity` automático

---

## Arquitectura del pipeline

```cmd
                     ┌─────────────────────────────────────────┐
                     │           logstash.conf                 │
                     │                                         │
  Filebeat :5044 ──► │  INPUT         FILTER          OUTPUT   │
  TCP JSON :50000 ──►│  ───────►  ───────────►  ──────────►    │──► Elasticsearch
                     │            enrutamiento    índice       │
                     │            parseo grok     dinámico     │
                     │            severity                     │
                     └─────────────────────────────────────────┘
```

---

## Inputs

### Puerto 5044 — Beats (Filebeat)

Recibe eventos de todos los agentes Filebeat de la red.
No requiere autenticación adicional — la seguridad se gestiona a nivel de red/firewall.

```ruby
input {
  beats {
    port => 5044
  }
}
```

### Puerto 50000 — TCP con JSON

Input genérico para aplicaciones que quieran mandar logs directamente
sin pasar por Filebeat, usando JSON sobre TCP.

```ruby
input {
  tcp {
    port => 50000
    codec => json
  }
}
```

Ejemplo de uso desde una aplicación:

```bash
echo '{"message": "deploy completado", "app": "miapp", "severity": "info"}' \
  | nc IP_SERVIDOR_ELK 50000
```

---

## Filter — Enrutamiento de índices

### Cómo funciona `[@metadata][target_index]`

`@metadata` es un campo especial de Logstash que **no se guarda en Elasticsearch** —
existe solo durante el procesamiento interno del pipeline. Se usa para tomar decisiones
sobre el output sin contaminar el documento final.

El campo `log_category` y `app_name` los manda Filebeat desde cada cliente
(definidos en `filebeat.yml` bajo `fields:`). Logstash los lee y decide el índice:

```cmd
¿Tiene app_name?                  →  apps-{app_name}-YYYY.MM.dd
¿log_category es sistema/auth/…?  →  sistema-YYYY.MM.dd
¿tiene tag "docker"?              →  docker-YYYY.MM.dd
Cualquier otra cosa               →  logstash-YYYY.MM.dd   (fallback)
```

### Tabla de enrutamiento completa

| Condición en el log                        | Índice resultante              |
|---|---|
| `app_name` presente (cualquier valor)      | `apps-{app_name}-YYYY.MM.dd`   |
| `log_category` = syslog, kernel, auth,     | `sistema-YYYY.MM.dd`           |
| cron, paquetes, vmware, apache             |                                |
| tag `docker` presente                      | `docker-YYYY.MM.dd`            |
| Resto / sin clasificar                     | `logstash-YYYY.MM.dd`          |

### Añadir un nuevo tipo de índice

Si en el futuro quieres un índice separado para, por ejemplo, logs de bases de datos:

```ruby
# Añadir en el bloque filter, antes del else final
else if [log_category] in ["mysql", "postgresql", "mongodb"] {
  mutate {
    add_field => { "[@metadata][target_index]" => "bbdd" }
  }
}
```

Y en Filebeat, asegurarte de que el bloque correspondiente tiene:

```yaml
fields:
  log_category: "mysql"
```

---

## Filter — Parseo con Grok

### auth.log — SSH y sudo

Extrae campos estructurados de los logs de autenticación de Linux.
Solo se aplica cuando `log_category == "auth"`.

Patrones que reconoce:

```cmd
# Login SSH fallido → extrae: ssh_user, src_ip
Mar 15 10:23:41 servidor sshd[1234]: Failed password for root from 192.168.1.50

# Login SSH exitoso → extrae: ssh_user, src_ip
Mar 15 10:23:41 servidor sshd[1234]: Accepted password for deploy from 10.20.1.5

# Ejecución con sudo → extrae: sudo_user, sudo_command
Mar 15 10:23:41 servidor sudo: deploy : TTY=pts/0 ; PWD=/home ; USER=root ; COMMAND=/bin/systemctl restart nginx
```

Campos resultantes en Elasticsearch:

| Campo          | Ejemplo                  | Descripción                            |
|---|---|---|
| `ssh_user`     | `root`                   | Usuario con el que se intentó el login |
| `src_ip`       | `192.168.1.50`           | IP origen del intento                  |
| `sudo_user`    | `deploy`                 | Usuario que ejecutó sudo               |
| `sudo_command` | `/bin/systemctl restart` | Comando ejecutado con sudo             |

> Si el mensaje no encaja con ningún patrón, Logstash lo deja pasar sin
> añadir campos extra (`tag_on_failure => []` suprime el tag `_grokparsefailure`).

### Apache — Combined Log Format

Parsea el formato estándar de access.log de Apache/Nginx.
Solo se aplica cuando `log_category == "apache"`.

```cmd
192.168.1.1 - frank [10/Oct/2024:13:55:36 +0000] "GET /index.html HTTP/1.1" 200 2326
```

Campos resultantes:

| Campo        | Ejemplo                          |
|---|---|
| `clientip`   | `192.168.1.1`                    |
| `ident`      | `frank`                          |
| `request`    | `GET /index.html HTTP/1.1`       |
| `response`   | `200` (convertido a integer)     |
| `bytes`      | `2326`                           |
| `referrer`   | URL de referencia                |
| `agent`      | User-Agent del cliente           |

### Docker — JSON nativo

Los contenedores Docker escriben sus logs en formato JSON. Logstash los parsea
y deposita el contenido en el campo `docker_payload` para no machacar el `message` original.

---

## Filter — Campo `severity` automático

Logstash analiza el texto de **todos** los mensajes (independientemente del origen)
y añade un campo `severity` según las palabras clave que encuentre:

| Valor `severity` | Palabras clave detectadas (case-insensitive)          |
|---|---|
| `critical`       | critical, fatal, emergency                            |
| `error`          | error, exception, traceback, failed, failure          |
| `warning`        | warn, warning, deprecated                             |
| `info`           | cualquier mensaje que no encaje con los anteriores    |

Este campo es especialmente útil en Kibana para filtrar con:

```kql
severity : "error"
severity : "critical" and server_name : "SENTRY"
not severity : "info"
```

O para crear visualizaciones de distribución de errores en dashboards.

> El campo `severity` lo añade Logstash — no viene de Filebeat ni de la aplicación.
> Si la aplicación ya manda un campo `level` o `log_level`, puedes adaptarlo en el filter.

---

## Output — Índice dinámico

```ruby
output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    user => "elastic"
    password => "${ELASTIC_PASSWORD}"
    index => "%{[@metadata][target_index]}-%{+YYYY.MM.dd}"
  }
}
```

El nombre final del índice se compone de dos partes:

```cmd
apps-apitest  -  2024.03.15
    ▲                ▲
    │                └── fecha del log (no del día de procesamiento)
    └── valor de [@metadata][target_index]
```

### Variable de entorno `ELASTIC_PASSWORD`

Logstash lee `${ELASTIC_PASSWORD}` del entorno del contenedor Docker.
Para que funcione, el servicio `logstash` en `docker-compose.yml` debe tener:

```yaml
services:
  logstash:
    environment:
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}   # ← obligatorio
```

Sin esta línea, Logstash intentará conectarse con la cadena literal
`${ELASTIC_PASSWORD}` y fallará con error de autenticación.

---

## Verificar el estado del pipeline

```bash
# Ver si Logstash está procesando eventos (busca "events.in" y "events.out")
curl -s http://localhost:9600/_node/stats/pipeline | python3 -m json.tool

# Ver logs del contenedor en tiempo real
docker compose logs -f logstash

# Filtrar solo errores en los logs de Logstash
docker compose logs logstash | grep -i "error\|warn\|exception"

# Ver los índices creados en Elasticsearch
curl -s -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/_cat/indices?v&s=index&h=index,docs.count,store.size"
```

---

## Ampliar el pipeline — Casos de uso frecuentes

### Parsear logs JSON de una aplicación

Si tu app manda logs en formato JSON dentro del campo `message`:

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

### Descartar logs que no interesan

Para no almacenar en Elasticsearch ciertos mensajes que generan ruido:

```ruby
filter {
  if [message] =~ /health.?check|ping|GET \/status/ {
    drop {}
  }
}
```

### Añadir un campo calculado

Por ejemplo, marcar logs de IPs privadas vs públicas:

```ruby
filter {
  if [src_ip] =~ /^(10\.|192\.168\.|172\.(1[6-9]|2\d|3[01])\.)/ {
    mutate { add_field => { "ip_type" => "privada" } }
  } else if [src_ip] {
    mutate { add_field => { "ip_type" => "publica" } }
  }
}
```

---

## Troubleshooting de Logstash

### Los índices se crean como `logstash-*` en lugar de `apps-*`

El campo `app_name` o `log_category` no está llegando correctamente.
Causas más comunes:

1. Falta `fields_under_root: true` en `filebeat.yml` — sin esto los campos
   van dentro de `fields.app_name` en lugar de al raíz, y Logstash no los encuentra.
2. El bloque de input en `filebeat.yml` no tiene el campo `fields:` correctamente indentado.

Diagnóstico:

```bash
# Activar debug temporal en Logstash para ver qué campos llegan
docker compose logs logstash | grep -A5 "app_name\|log_category"
```

### Error de autenticación con Elasticsearch

```cmd
Elasticsearch error: response code 401 (Unauthorized)
```

Verifica que la variable de entorno está pasada al contenedor:

```bash
docker compose exec logstash env | grep ELASTIC_PASSWORD
```

Si no aparece, falta la línea `environment` en el `docker-compose.yml`.

### Logstash arranca antes que Elasticsearch y falla

```bash
docker compose restart logstash
```

Logstash reintentará la conexión y debería conectar en cuanto ES esté listo.

### Ver eventos que procesa Logstash en tiempo real (debug)

```bash
# Añadir temporalmente un output stdout al logstash.conf
output {
  stdout { codec => rubydebug }   # ← solo para debug, nunca en producción
  elasticsearch { ... }
}
```

Luego `docker compose restart logstash` y `docker compose logs -f logstash`.
Recuerda quitarlo después.
