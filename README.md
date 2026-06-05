# 📘 Guía de Operaciones — ELK Stack

Documentación para el técnico que gestiona el sistema con el stack ya desplegado.
Si necesitas montar el sistema desde cero, empieza por el
[README principal](../README.md).

---

## Documentos

| # | Documento | Para qué |
|---|---|---|
| 01 | [ARQUITECTURA](./01-ARQUITECTURA.md) | Cómo funciona el sistema, qué hace cada pieza y por qué está hecho así |
| 02 | [FILEBEAT](./02-FILEBEAT.md) | Instalar el agente, añadir servidores y configurar nuevos logs |
| 03 | [LOGSTASH](./03-LOGSTASH.md) | Pipeline, enrutamiento de índices, parseo grok y enriquecimiento |
| 04 | [KIBANA](./04-KIBANA.md) | Data Views, Discover, KQL, Dashboards y Alertas |
| 05 | [OPERACIONES](./05-OPERACIONES.md) | Arranque, retención ILM, actualizaciones y backups |
| 06 | [TROUBLESHOOTING](./06-TROUBLESHOOTING.md) | Problemas frecuentes y cómo resolverlos paso a paso |

---

## Por dónde empezar según la tarea

| Situación | Documento |
|---|---|
| Soy nuevo y no conozco el sistema | [01-ARQUITECTURA](./01-ARQUITECTURA.md) → [05-OPERACIONES](./05-OPERACIONES.md) |
| Añadir un servidor cliente nuevo | [02-FILEBEAT](./02-FILEBEAT.md) — "Añadir un servidor cliente" |
| Añadir logs de una app nueva | [02-FILEBEAT](./02-FILEBEAT.md) — "Añadir logs de una app nueva" |
| No llegan logs de un servidor | [06-TROUBLESHOOTING](./06-TROUBLESHOOTING.md) — "No veo logs de un servidor" |
| Los logs van al índice equivocado | [06-TROUBLESHOOTING](./06-TROUBLESHOOTING.md) — "Logs en índice equivocado" |
| Buscar algo concreto en Kibana | [04-KIBANA](./04-KIBANA.md) — "KQL referencia rápida" |
| Crear un dashboard o alerta | [04-KIBANA](./04-KIBANA.md) — "Dashboards" / "Alertas" |
| El disco está casi lleno | [05-OPERACIONES](./05-OPERACIONES.md) — "Retención de datos ILM" |
| Entender cómo funciona Logstash | [03-LOGSTASH](./03-LOGSTASH.md) |
| Cambiar cómo se nombran los índices | [03-LOGSTASH](./03-LOGSTASH.md) — "Enrutamiento de índices" |

---

## Ficheros de configuración

| Fichero | Servidor | Ruta |
|---|---|---|
| `docker-compose.yml` | ELK central | `~/docker-elk-secure/` |
| `.env` | ELK central | `~/docker-elk-secure/` |
| `logstash.conf` | ELK central | `~/docker-elk-secure/logstash/pipeline/` |
| `logstash.yml` | ELK central | `~/docker-elk-secure/logstash/config/` |
| `filebeat.yml` | cada cliente | `/etc/filebeat/filebeat.yml` |
| `filebeat.yml.example` | repo | `filebeat/` |
| Registry de Filebeat | cada cliente | `/var/lib/filebeat/registry/` |
| Logs del agente | cada cliente | `/var/log/filebeat/filebeat.log` |

---

## Versiones

| Componente | Versión | Notas |
|---|---|---|
| Elasticsearch | 8.x | Licencia Basic — gratuita e ilimitada |
| Logstash | 8.x | Misma versión major que ES |
| Kibana | 8.x | Misma versión major que ES |
| Filebeat | 8.x | Instalado como paquete en cada cliente |
| Docker Compose | 2.x | Stack central |
