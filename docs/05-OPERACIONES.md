# 05 — Operaciones

Gestión del día a día del stack ELK: arranque, mantenimiento, retención y monitorización.

---

## Arranque y parada

El stack corre en Docker Compose en el servidor central.

```bash
cd ~/docker-elk-secure

# Arrancar todo
docker compose up -d

# Ver estado de los contenedores
docker compose ps

# Parar — conserva todos los datos
docker compose down

# Parar y BORRAR todos los datos (volúmenes) — usar con mucho cuidado
docker compose down -v

# Reiniciar un servicio concreto
docker compose restart logstash
docker compose restart kibana
docker compose restart elasticsearch

# Ver logs en tiempo real
docker compose logs -f
docker compose logs -f logstash    # solo un servicio
```

### Orden de arranque y tiempos

Elasticsearch tarda 30-60 segundos en estar completamente listo.
Kibana y Logstash arrancan antes y pueden dar error de conexión los primeros minutos.
Si ocurre, basta esperar o reiniciarlos:

```bash
docker compose restart kibana logstash
```

### Verificar que todo está bien

```bash
# Cluster Elasticsearch
curl -s -u elastic:${ELASTIC_PASSWORD} \
  http://localhost:9200/_cluster/health?pretty | grep '"status"'
# green = todo bien
# yellow = réplicas sin asignar (normal y esperado en instalación single-node)
# red = hay shards perdidos, problema serio

# Ver índices y tamaño
curl -s -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/_cat/indices?v&s=index&h=index,docs.count,store.size"

# Eventos procesados por Logstash
curl -s http://localhost:9600/_node/stats/pipeline | python3 -m json.tool | grep -A3 '"events"'
```

---

## Retención de datos — ILM

Sin política de retención, Elasticsearch llenará el disco.
Hay que definir cuánto tiempo conservar cada tipo de log.

### Ver políticas existentes

```bash
curl -s -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/_ilm/policy?pretty" | grep -E '"policy"|"min_age"'
```

### Crear políticas de retención

```bash
# Política de 30 días
curl -u elastic:${ELASTIC_PASSWORD} \
  -X PUT http://localhost:9200/_ilm/policy/elk-30dias \
  -H "Content-Type: application/json" \
  -d '{"policy":{"phases":{"delete":{"min_age":"30d","actions":{"delete":{}}}}}}'

# Política de 60 días
curl -u elastic:${ELASTIC_PASSWORD} \
  -X PUT http://localhost:9200/_ilm/policy/elk-60dias \
  -H "Content-Type: application/json" \
  -d '{"policy":{"phases":{"delete":{"min_age":"60d","actions":{"delete":{}}}}}}'

# Política de 15 días (para índices que crecen rápido)
curl -u elastic:${ELASTIC_PASSWORD} \
  -X PUT http://localhost:9200/_ilm/policy/elk-15dias \
  -H "Content-Type: application/json" \
  -d '{"policy":{"phases":{"delete":{"min_age":"15d","actions":{"delete":{}}}}}}'
```

### Aplicar políticas a los índices

```bash
# Apps — 30 días
curl -u elastic:${ELASTIC_PASSWORD} \
  -X PUT "http://localhost:9200/apps-*/_settings" \
  -H "Content-Type: application/json" \
  -d '{"index.lifecycle.name": "elk-30dias"}'

# Sistema — 60 días (suele interesar más historial)
curl -u elastic:${ELASTIC_PASSWORD} \
  -X PUT "http://localhost:9200/sistema-*/_settings" \
  -H "Content-Type: application/json" \
  -d '{"index.lifecycle.name": "elk-60dias"}'

# Docker — 15 días (crecen rápido)
curl -u elastic:${ELASTIC_PASSWORD} \
  -X PUT "http://localhost:9200/docker-*/_settings" \
  -H "Content-Type: application/json" \
  -d '{"index.lifecycle.name": "elk-15dias"}'
```

También se puede gestionar desde Kibana:
**Stack Management → Index Lifecycle Policies → Create Policy**

### Ver cuánto ocupa cada índice

```bash
# Ordenar por tamaño descendente
curl -s -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/_cat/indices?v&s=store.size:desc&h=index,docs.count,store.size" \
  | head -20
```

### Borrar un índice manualmente (disco lleno)

```bash
# Identificar los más antiguos
curl -s -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/_cat/indices?v&s=index&h=index,docs.count,store.size" \
  | grep "2026.01"

# Borrar — irreversible
curl -u elastic:${ELASTIC_PASSWORD} \
  -X DELETE "http://localhost:9200/sistema-2026.01.15"

# Borrar un patrón completo — mucho cuidado con esto
curl -u elastic:${ELASTIC_PASSWORD} \
  -X DELETE "http://localhost:9200/apps-testapp-*"
```

### Reducir espacio en instalación single-node

En single-node las réplicas son innecesarias y doblan el uso de disco:

```bash
curl -u elastic:${ELASTIC_PASSWORD} \
  -X PUT "http://localhost:9200/_settings" \
  -H "Content-Type: application/json" \
  -d '{"index.number_of_replicas": 0}'
```

---

## Monitorización del propio stack

### Verificaciones recomendadas

```bash
# 1. Espacio en disco del servidor
df -h /var/lib/docker/volumes/

# 2. Estado del cluster
curl -s -u elastic:${ELASTIC_PASSWORD} \
  http://localhost:9200/_cluster/health?pretty | grep '"status"'

# 3. Contenedores corriendo
docker compose ps

# 4. Filebeat corriendo en cada cliente (ejecutar en el cliente)
sudo systemctl status filebeat

# 5. Último log procesado (ver que la fecha es reciente)
curl -s -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/_cat/indices?v&s=index:desc&h=index,docs.count" | head -5
```

### Alertas recomendadas en Kibana

Configúralas en Stack Management → Rules:

- **Servidor sin logs en 15 minutos** — indica Filebeat caído o problema de red
- **Más de 5 errores críticos en producción en 10 minutos**
- **Más de 20 intentos SSH fallidos en 5 minutos** — posible fuerza bruta

Ver ejemplos completos en [04-KIBANA.md](./04-KIBANA.md) — sección Alertas.

---

## Actualizar el stack

```bash
cd ~/docker-elk-secure

# Descargar nuevas imágenes
docker compose pull

# Aplicar con tiempo de parada mínimo
docker compose up -d

# Verificar que todo arrancó bien
docker compose ps
docker compose logs --tail=50
```

> Mantener siempre la misma versión major (8.x) entre Elasticsearch, Kibana,
> Logstash y Filebeat. Mezclar versiones puede causar errores de protocolo.

---

## Cambiar contraseñas

Las contraseñas están en `~/docker-elk-secure/.env`.

Después de cambiar el fichero `.env`:

```bash
# 1. Actualizar la contraseña en Elasticsearch
curl -u elastic:CONTRASEÑA_ANTIGUA \
  -X POST http://localhost:9200/_security/user/elastic/_password \
  -H "Content-Type: application/json" \
  -d '{"password": "CONTRASEÑA_NUEVA"}'

# 2. Actualizar kibana_system
curl -u elastic:CONTRASEÑA_NUEVA \
  -X POST http://localhost:9200/_security/user/kibana_system/_password \
  -H "Content-Type: application/json" \
  -d '{"password": "KIBANA_SYSTEM_PASSWORD_NUEVA"}'

# 3. Reiniciar los servicios para que cojan el nuevo .env
docker compose down
docker compose up -d
```

---

## Backup y recuperación

Elasticsearch no tiene backup automático en esta instalación.
Los datos están en volúmenes de Docker:

```bash
# Ver dónde están los volúmenes
docker volume ls | grep elk

# Backup manual del volumen de Elasticsearch
docker run --rm \
  -v docker-elk-secure_elasticsearch-data:/data \
  -v $(pwd)/backup:/backup \
  alpine tar czf /backup/elasticsearch-$(date +%Y%m%d).tar.gz /data
```

Para recuperación ante desastre, lo más práctico es asumir que los logs
en Elasticsearch son efímeros y que la fuente de verdad son los ficheros
en los servidores cliente. Filebeat puede reenviar el historial borrando
el registry (ver [02-FILEBEAT.md](./02-FILEBEAT.md)).
