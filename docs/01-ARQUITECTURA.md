# 01 — Arquitectura y flujo de datos

---

## El sistema en una imagen

```
SERVIDORES CLIENTE                      SERVIDOR CENTRAL (Docker)
──────────────────────                  ─────────────────────────────────────

 /var/log/syslog                        ┌─────────────────────────────────┐
 /var/log/auth.log      ┌─────────┐     │  Logstash :5044                 │
 /var/log/app*.log  ───►│Filebeat │────►│  · recibe eventos de Filebeat   │
 /var/log/nginx/    ───►│ (8.x)   │     │  · decide el índice destino     │──► Elasticsearch
 contenedores Docker     └─────────┘     │  · parsea auth, apache          │     · apps-{nombre}-*
                                         │  · añade campo severity         │     · sistema-*
 /var/log/syslog                        └─────────────────────────────────┘     · docker-*
 /var/log/auth.log      ┌─────────┐                                             · logstash-*
 /var/log/app*.log  ───►│Filebeat │────►  (mismo Logstash)
  ...otro servidor       └─────────┘                                        Kibana :5601
                                                                             · Data Views
                                                                             · Discover
                                                                             · Dashboards
                                                                             · Alertas
```

---

## Qué hace cada pieza

### Filebeat
Agente ligero instalado como paquete del sistema en cada servidor cliente.
Su único trabajo es leer ficheros de log y enviarlos a Logstash.

- Lee los ficheros definidos en `/etc/filebeat/filebeat.yml`
- Guarda en `/var/lib/filebeat/registry` hasta dónde ha leído cada fichero,
  para no duplicar eventos si se reinicia
- Añade metadatos al evento: `host.name`, `@timestamp`, `input.type`
- Envía los eventos al servidor ELK central por el puerto 5044

### Logstash
Corre en Docker en el servidor central. Es el cerebro del pipeline.

- **Recibe** eventos de todos los agentes Filebeat de la red
- **Enruta** cada evento al índice correcto leyendo `app_name` y `log_category`
- **Parsea** formatos conocidos (auth.log con Grok, JSON de Docker)
- **Enriquece** todos los eventos con el campo `severity` automático
- **Escribe** el resultado en Elasticsearch

### Elasticsearch
Base de datos de búsqueda donde se almacenan los logs.

- Organiza los datos en **índices** por tipo y fecha
- Permite búsquedas en texto completo y por campos estructurados
- Gestiona la retención mediante políticas ILM

### Kibana
Interfaz web para trabajar con los logs.

- **Discover**: buscar y explorar logs en tiempo real
- **Dashboards**: visualizaciones y métricas
- **Alertas**: notificaciones cuando ocurre algo anómalo
- **Stack Management**: gestión de índices, Data Views, políticas ILM

---

## Campos que viajan con cada evento

Estos campos están disponibles en todos los logs en Kibana:

| Campo | Lo añade | Contenido |
|---|---|---|
| `@timestamp` | Filebeat | Fecha y hora del log |
| `message` | Filebeat | Texto del log original |
| `host.name` | Filebeat | Hostname del servidor cliente |
| `server_name` | filebeat.yml | Nombre personalizado del servidor |
| `environment` | filebeat.yml | production / staging / development |
| `location` | filebeat.yml | Zona, datacenter, proveedor |
| `log_category` | filebeat.yml | Tipo: auth, syslog, nginx, apitest... |
| `app_name` | filebeat.yml | Nombre de la app (define el índice destino) |
| `tags` | filebeat.yml | Array de etiquetas: ["app","sentry-project"] |
| `input.type` | Filebeat | filestream / container |
| `severity` | Logstash | critical / error / warning / info |
| `log.file.path` | Filebeat | Ruta del fichero de log en el cliente |
| `container.name` | Filebeat | Nombre del contenedor (solo inputs Docker) |
| `src_ip` | Logstash | IP origen en logs SSH (si grok aplica) |
| `ssh_user` | Logstash | Usuario en intentos SSH (si grok aplica) |
| `sudo_command` | Logstash | Comando ejecutado con sudo (si grok aplica) |

---

## Índices en Elasticsearch

Logstash crea los índices automáticamente según el tipo de log:

| Índice | Qué contiene | Criterio |
|---|---|---|
| `apps-{app_name}-YYYY.MM.dd` | Logs de una app concreta | Tiene campo `app_name` |
| `sistema-YYYY.MM.dd` | syslog, auth, kernel, cron, nginx, vmware... | `log_category` de sistema |
| `docker-YYYY.MM.dd` | Logs de contenedores Docker | `input.type == container` |
| `logstash-YYYY.MM.dd` | Lo que no encaja en ninguna categoría | Fallback |

Un índice nuevo se crea automáticamente cada día que llegan logs.
No hay que hacer nada manual para que aparezcan.

---

## Decisiones de diseño

Estas decisiones se tomaron con criterio — si las cambias, entiende primero por qué están así.

### ¿Por qué Filebeat → Logstash → Elasticsearch y no Filebeat directo a Elasticsearch?

Filebeat puede enviar directo a Elasticsearch. Se eligió pasar por Logstash porque:

- Filebeat no puede decidir dinámicamente a qué índice va cada log. Logstash sí,
  usando `[@metadata][target_index]`.
- El parseo con Grok (extraer `src_ip`, `ssh_user` de auth.log) solo es posible en Logstash.
- El campo `severity` automático requiere analizar el texto del mensaje — eso es Logstash.
- Si en el futuro hay que filtrar, transformar o enrutar a múltiples destinos,
  solo hay que tocar `logstash.conf`, sin tocar los agentes.

### ¿Por qué type: filestream en lugar de type: log?

`type: log` está deprecado en Filebeat 8.x. En versiones recientes se ignora
silenciosamente. `type: filestream` es el reemplazo oficial con mejor gestión
de ficheros rotados y un `id` único por input que evita duplicados al reiniciar.

### ¿Por qué fields_under_root: true en cada input y no solo en el global?

En Filebeat 8.x `fields_under_root: true` a nivel global **no se hereda** a los inputs.
Sin él en cada bloque, los campos `app_name` y `log_category` llegan a Logstash
anidados bajo `fields.app_name` y el enrutamiento de índices falla — todo acaba
en el índice fallback `logstash-*`.

### ¿Por qué [input][type] == "container" para detectar Docker y no el tag "docker"?

El processor `add_docker_metadata` añade el tag `docker` a **todos** los eventos
del host, no solo a los de contenedores. Usando `[input][type] == "container"`
se identifica con precisión solo los eventos que vienen del input de tipo container.

### ¿Por qué un índice por app y no un índice global para todas?

- Permite **retención distinta** por app — una app crítica guarda 90 días,
  una de desarrollo 15 días. Las políticas ILM van por patrón de índice.
- Las búsquedas en un índice pequeño son más rápidas.
- Permite **acceso granular** — un equipo puede tener permisos solo sobre
  `apps-suapp-*` sin ver el resto.
- En Kibana es más limpio tener un Data View `apps-miapp-*` que filtrar
  siempre manualmente.

### ¿Por qué severity lo añade Logstash y no viene de la app?

Las aplicaciones no tienen formato uniforme: unas usan `ERROR`, otras `[error]`,
otras `CRITICAL`, otras nada. Logstash normaliza todo esto con regex sobre
el campo `message` y produce un `severity` consistente para todos los logs.
Si una app manda su propio campo `level`, el pipeline puede adaptarse para
usarlo cuando exista, sin tocar Filebeat.
