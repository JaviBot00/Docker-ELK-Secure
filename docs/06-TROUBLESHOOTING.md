# 06 — Troubleshooting

Guía de diagnóstico para los problemas más frecuentes.
Cada sección empieza por los síntomas y va de lo más simple a lo más complejo.

---

## Índice de problemas

- [No veo logs de un servidor en Kibana](#no-veo-logs-de-un-servidor-en-kibana)
- [Los logs van al índice equivocado](#los-logs-van-al-índice-equivocado)
- [Kibana no carga o dice "not ready"](#kibana-no-carga-o-dice-not-ready)
- [Logstash no arranca o error de autenticación](#logstash-no-arranca-o-error-de-autenticación)
- [Elasticsearch no arranca](#elasticsearch-no-arranca)
- [Un tipo de log concreto no llega](#un-tipo-de-log-concreto-no-llega)
- [Los logs se duplican](#los-logs-se-duplican)
- [El disco está casi lleno](#el-disco-está-casi-lleno)
- [Discover no muestra resultados aunque hay índices](#discover-no-muestra-resultados-aunque-hay-índices)

---

## No veo logs de un servidor en Kibana

### Paso 1 — Verificar que hay índices recientes

```bash
# En el servidor ELK
curl -s -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/_cat/indices?v&s=index:desc&h=index,docs.count" | head -10
```

Si no hay índices de hoy, el problema está en Filebeat o en la red.
Si hay índices pero no del servidor en cuestión, el problema es específico de ese agente.

### Paso 2 — Verificar Filebeat en el servidor cliente

```bash
# Estado del servicio
sudo systemctl status filebeat

# Ver los últimos logs del agente
sudo journalctl -u filebeat --since "10 minutes ago"

# Probar conectividad con Logstash
sudo filebeat test output -e

# Probar conectividad manual al puerto
nc -zv 10.20.173.250 5044
```

### Paso 3 — Verificar que el fichero de configuración es correcto

```bash
sudo filebeat test config -e
```

Si da errores, corregirlos antes de continuar.

### Paso 4 — Ver si Logstash recibe conexiones

```bash
# En el servidor ELK
docker compose logs logstash | grep -i "beats\|connection\|error" | tail -20
```

### Tabla de errores comunes en Filebeat

| Error en los logs | Causa | Solución |
|---|---|---|
| `connection refused :5044` | Logstash caído o firewall | Reiniciar Logstash; abrir puerto 5044 |
| `i/o timeout` | Problema de red | Verificar conectividad con `nc` |
| `No such file or directory` | Path de log incorrecto | Verificar que el fichero existe |
| `permission denied` | Sin permisos de lectura | Verificar que Filebeat corre como root |
| `Error loading config file` | Error de sintaxis en yml | Ejecutar `filebeat test config -e` |

---

## Los logs van al índice equivocado

**Síntoma:** todo aparece en `docker-*` o en `logstash-*` en lugar de en `apps-*` o `sistema-*`.

### Diagnóstico — ver qué campos llegan realmente

```bash
# Ver los campos de un documento del índice incorrecto
curl -s -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/logstash-*/_search?pretty&size=1" \
  | grep -E '"app_name"|"log_category"|"fields"|"input"'
```

**Si ves esto** — los campos están anidados, `fields_under_root` no funciona:
```json
"fields": {
  "log_category": "apitest",
  "app_name": "apitest"
}
```

**Si ves esto** — los campos están al nivel raíz, correcto:
```json
"log_category": "apitest",
"app_name": "apitest"
```

### Solución — fields_under_root en cada input

En `/etc/filebeat/filebeat.yml`, verificar que **cada bloque de input** tiene
`fields_under_root: true` directamente dentro del bloque (no solo en el global):

```yaml
- type: filestream
  id: app-apitest
  enabled: true
  paths:
    - /var/log/apitest*.log
  fields:
    log_category: "apitest"
    app_name: "apitest"
  fields_under_root: true    # ← debe estar en CADA bloque
```

Después:

```bash
sudo systemctl restart filebeat
```

### Si el problema es que todo va a docker-*

El processor `add_docker_metadata` añade el tag `docker` a todos los eventos
del host. Si la condición de Docker en Logstash usa `"docker" in [tags]` en lugar
de `[input][type] == "container"`, todo caerá en el índice Docker.

Verificar en `logstash.conf` que la condición de Docker es:
```ruby
if [input][type] == "container" {
```
Y no:
```ruby
if "docker" in [tags] {    # ← incorrecto
```

---

## Kibana no carga o dice "not ready"

### Esperar el tiempo de arranque

Kibana tarda 60-90 segundos en estar listo tras `docker compose up`.
Esperar antes de diagnosticar.

### Verificar el motivo

```bash
docker compose logs kibana | grep -i "error\|warn\|fatal" | tail -20
```

### Error de autenticación con Elasticsearch

Causa más común: la contraseña del usuario `kibana_system` no se configuró
o no coincide con el `.env`.

```bash
# Actualizar la contraseña
curl -u elastic:${ELASTIC_PASSWORD} \
  -X POST http://localhost:9200/_security/user/kibana_system/_password \
  -H "Content-Type: application/json" \
  -d '{"password": "'"${KIBANA_SYSTEM_PASSWORD}"'"}'

docker compose restart kibana
```

### Elasticsearch no está listo todavía

```bash
# Verificar que ES responde
curl -s -u elastic:${ELASTIC_PASSWORD} http://localhost:9200/_cluster/health

# Si no responde, esperar o reiniciar
docker compose restart elasticsearch
# Esperar 60 segundos
docker compose restart kibana
```

---

## Logstash no arranca o error de autenticación

```bash
docker compose logs logstash | tail -40
```

### Error 401 — contraseña incorrecta

La variable `ELASTIC_PASSWORD` no llega al contenedor.

```bash
# Verificar que la variable existe en el contenedor
docker compose exec logstash env | grep ELASTIC_PASSWORD
```

Si no aparece, falta en `docker-compose.yml`:

```yaml
services:
  logstash:
    environment:
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}   # ← añadir esto
```

Después: `docker compose up -d --force-recreate logstash`

### Logstash arranca antes que Elasticsearch

```bash
docker compose restart logstash
```

Logstash reintentará la conexión automáticamente.

### Error de sintaxis en logstash.conf

```bash
# Ver el error exacto
docker compose logs logstash | grep -i "error\|syntax\|unexpected" | head -20
```

Logstash es estricto con la sintaxis Ruby. Los errores más comunes son
llaves `{}` sin cerrar o comillas mal emparejadas.

---

## Elasticsearch no arranca

```bash
docker compose logs elasticsearch | tail -30
```

### Exit code 137 — memoria insuficiente

El contenedor fue matado por el OOM killer. Necesita al menos 2GB de RAM.

```bash
# Ver memoria disponible
free -h

# Reducir el heap de ES si hay poca RAM (en docker-compose.yml)
environment:
  - ES_JAVA_OPTS=-Xms512m -Xmx512m   # reducir de 1g a 512m
```

### vm.max_map_count demasiado bajo

```bash
# Síntoma en los logs:
# max virtual memory areas vm.max_map_count [65530] is too low

sudo sysctl -w vm.max_map_count=262144

# Para que persista tras reiniciar
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

### Datos corruptos en el volumen

```bash
# Último recurso — borra todos los datos
docker compose down -v
docker compose up -d
```

---

## Un tipo de log concreto no llega

Por ejemplo: VMware, Nginx, una app concreta.

### Paso 1 — Verificar que el fichero existe y tiene contenido

```bash
ls -la /ruta/del/log.log
tail -5 /ruta/del/log.log
```

Si el fichero no existe, el input está definido para una ruta incorrecta.

### Paso 2 — Verificar el path en filebeat.yml

```bash
sudo grep -A5 "id: sistema-vmware" /etc/filebeat/filebeat.yml
```

Comparar el path del fichero real con el path en la configuración.
El error más común es que los ficheros están en `/var/log/` directamente
pero el path en la config apunta a `/var/log/vmware/` (subdirectorio inexistente).

### Paso 3 — Verificar permisos

```bash
# Ver permisos del fichero
ls -la /ruta/del/log.log

# Filebeat corre normalmente como root
ps aux | grep filebeat | grep -v grep | awk '{print $1}'
```

Si el fichero es `rw-------` y Filebeat corre como root, puede leerlo.
Si corre como otro usuario, habría que añadirlo al grupo propietario del fichero.

### Paso 4 — Debug específico del input

```bash
sudo filebeat -e -d "input" 2>&1 | grep -i "vmware\|harvester\|error" | head -30
```

### Paso 5 — Ver en Kibana si llega algo

```bash
curl -s -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/sistema-*/_search?pretty&size=3&q=log_category:vmware" \
  | grep -E '"message"|"log_category"|"hits"'
```

Si aparecen resultados aquí pero no en Kibana, el problema es el Data View
o el rango de tiempo en Discover — ver sección siguiente.

---

## Los logs se duplican

Causa más común: el registry de Filebeat se borró o está dañado, o se cambió
el `id` de un input ya existente (Filebeat lo trata como input nuevo y relee desde el principio).

**No cambiar el `id` de un input** una vez que está en producción — si hay que hacerlo,
asumir que habrá duplicados hasta que el fichero se rote.

Si los duplicados ya existen y hay que limpiarlos, la forma más práctica es
borrar los índices afectados en Elasticsearch y dejar que Filebeat los regenere
con los datos actuales (perderás el historial).

---

## El disco está casi lleno

```bash
# Ver qué ocupa más
df -h /var/lib/docker/volumes/
docker system df

# Ver índices por tamaño
curl -s -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/_cat/indices?v&s=store.size:desc&h=index,store.size,docs.count" \
  | head -15
```

Acciones por orden de impacto:

1. **Aplicar ILM** para borrar índices antiguos automáticamente — ver [05-OPERACIONES.md](./05-OPERACIONES.md)
2. **Borrar índices antiguos manualmente** si la situación es urgente
3. **Reducir réplicas a 0** en single-node (las réplicas doblan el espacio sin beneficio):
```bash
curl -u elastic:${ELASTIC_PASSWORD} \
  -X PUT "http://localhost:9200/_settings" \
  -H "Content-Type: application/json" \
  -d '{"index.number_of_replicas": 0}'
```
4. **Reducir retención** — acortar los plazos en las políticas ILM

---

## Discover no muestra resultados aunque hay índices

### Verificar el rango temporal

El selector de tiempo está arriba a la derecha. Si está en "Last 15 minutes"
y los datos son de ayer, no aparecerá nada. Probar con "Last 7 days" o "Last 30 days".

### Verificar el Data View

El Data View debe tener el patrón correcto (con asterisco) y el campo
`@timestamp` como timestamp field.

```bash
# Verificar que el índice existe
curl -s -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/_cat/indices?v&h=index" | grep "sistema"

# Verificar que tiene documentos
curl -s -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/sistema-*/_count"
```

### Refrescar el Data View

Si se crearon índices nuevos después de crear el Data View, puede que no reconozca
los nuevos campos. En Kibana: Stack Management → Data Views → seleccionar el Data View → Refresh.
