# 02 — Filebeat

Agente de recolección de logs instalado como paquete en cada servidor cliente.
Lee ficheros de log y los envía al Logstash central.

---

## Instalación (Debian/Ubuntu)

```bash
# 1. Clave GPG de Elastic
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch \
  | sudo gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg

# 2. Repositorio versión 8.x — debe coincidir con el stack Docker
echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] \
  https://artifacts.elastic.co/packages/8.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-8.x.list

# 3. Instalar
sudo apt-get update && sudo apt-get install -y filebeat

# 4. Copiar la plantilla del repositorio
sudo cp filebeat.yml.example /etc/filebeat/filebeat.yml
sudo nano /etc/filebeat/filebeat.yml
```

## Instalación (Windows — PowerShell)

```powershell
Invoke-WebRequest -Uri "https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.12.2-windows-x86_64.zip" `
  -OutFile "filebeat.zip"
Expand-Archive -Path "filebeat.zip" -DestinationPath "C:\Program Files\"
Rename-Item "C:\Program Files\filebeat-8.12.2-windows-x86_64" "C:\Program Files\Filebeat"
Copy-Item "filebeat.yml" "C:\Program Files\Filebeat\filebeat.yml"
cd "C:\Program Files\Filebeat"
.\install-service-filebeat.ps1
Start-Service filebeat
```

---

## Añadir un servidor cliente nuevo

### 1. Editar la identidad del servidor

Lo primero que hay que cambiar en `/etc/filebeat/filebeat.yml`:

```yaml
fields:
  server_name: "nombre-unico-del-servidor"   # identificador en Kibana
  environment: "production"                  # production | staging | development
  location: "donde-esta"                     # datacenter, zona, proveedor
fields_under_root: true
```

### 2. Configurar el output

```yaml
output.logstash:
  hosts: ["10.20.173.250:5044"]   # IP del servidor ELK — no cambiar
```

### 3. Activar los bloques de logs que apliquen

Cada tipo de log tiene su bloque con `enabled: true/false`.
Activa solo los que existan en ese servidor:

```yaml
- type: filestream
  id: sistema-nginx
  enabled: true      # ← cambiar a true si hay Nginx
  ...
```

### 4. Verificar y arrancar

```bash
sudo filebeat test config -e     # valida la sintaxis del fichero
sudo filebeat test output -e     # verifica conectividad con Logstash
sudo systemctl start filebeat
sudo systemctl enable filebeat   # arrancar automáticamente al reiniciar
sudo journalctl -u filebeat -f   # ver que no hay errores
```

---

## Añadir logs de una app nueva

Copia el bloque de ejemplo de la plantilla y personalízalo:

```yaml
- type: filestream
  id: app-miapp                  # único en todo el fichero — no repetir
  enabled: true
  paths:
    - /var/log/miapp/*.log       # ruta real de los logs
    - /ruta/custom/logs/*.log    # se pueden poner varios paths
  tags: ["app"]
  fields:
    log_category: "miapp"        # igual que el id sin "app-"
    app_name: "miapp"            # este valor define el índice: apps-miapp-*
  fields_under_root: true        # obligatorio en cada bloque
```

Después de editar:

```bash
sudo filebeat test config -e
sudo systemctl restart filebeat
```

---

## Estructura del fichero filebeat.yml

### Reglas que no se pueden saltarse en Filebeat 8.x

**`type: filestream`** — `type: log` está deprecado. En 8.x puede ignorarse
silenciosamente sin `allow_deprecated_use: true`. Usar siempre `filestream`.

**`id` único por input** — obligatorio en `filestream`. Si dos inputs tienen
el mismo `id`, Filebeat se comporta de forma imprevisible.

**`fields_under_root: true` en cada input** — no se hereda del nivel global.
Sin él, los campos `app_name` y `log_category` llegan a Logstash anidados
bajo `fields.app_name` y el enrutamiento de índices falla.

**`multiline` dentro de `parsers:`** — en `filestream` la multiline no va
directamente en el input sino bajo la clave `parsers:`:

```yaml
# ✗ Incorrecto (sintaxis de type: log)
- type: filestream
  multiline:
    pattern: '^\d{4}'
    negate: true
    match: after

# ✓ Correcto (sintaxis de type: filestream)
- type: filestream
  parsers:
    - multiline:
        type: pattern
        pattern: '^\d{4}'
        negate: true
        match: after
```

**`type: container` para Docker** — sigue siendo el tipo correcto para contenedores.
No tiene equivalente `filestream` todavía. Requiere `allow_deprecated_use: true`:

```yaml
- type: container
  allow_deprecated_use: true
  id: docker-containers
  ...
```

---

## Logs comprimidos — qué lee y qué no

**Filebeat NO lee ficheros `.gz`.** Solo lee texto plano.

Los ficheros rotados y comprimidos por logrotate (`syslog.2.gz`, `auth.log.3.gz`)
se ignoran completamente. Esto en la práctica no es problema porque:

- Filebeat lee los ficheros activos en tiempo real
- Cuando logrotate rota `syslog` a `syslog.1`, Filebeat continúa leyendo `syslog.1`
  hasta terminar, antes de que se comprima a `syslog.2.gz`
- El registry guarda la posición de cada fichero — no hay duplicados al reiniciar

**El único caso donde se pierden logs** es si Filebeat estuvo parado durante
una rotación. En ese caso los logs comprimidos mientras estaba caído se pierden.

Para minimizar este riesgo en logs críticos, editar `/etc/logrotate.d/rsyslog`
y añadir `delaycompress` para que no comprima hasta la siguiente rotación,
dando más margen a Filebeat:

```
/var/log/syslog {
    rotate 7
    daily
    delaycompress    # ← añadir esta línea
    compress
    ...
}
```

---

## El registry — cómo Filebeat recuerda lo que ha leído

Filebeat guarda en `/var/lib/filebeat/registry/` un registro de hasta dónde
ha leído cada fichero. Esto evita duplicados al reiniciar el agente.

```bash
# Ver qué ficheros está siguiendo y su posición
sudo cat /var/lib/filebeat/registry/filebeat/log.json | python3 -m json.tool | head -60
```

**Forzar que Filebeat relea todos los ficheros desde el principio:**

```bash
# CUIDADO: duplicará todos los logs históricos en Elasticsearch
sudo systemctl stop filebeat
sudo rm -rf /var/lib/filebeat/registry
sudo systemctl start filebeat
```

Hacer esto solo si se asume la duplicación de eventos — por ejemplo al
migrar a un Elasticsearch nuevo o al cambiar radicalmente la configuración.

---

## Comandos de gestión diaria

```bash
# Estado del servicio
sudo systemctl status filebeat

# Ver logs del agente en tiempo real
sudo journalctl -u filebeat -f

# Ver solo errores y warnings
sudo journalctl -u filebeat | grep -i "error\|warn\|ERR"

# Ver logs del agente guardados en fichero
sudo tail -f /var/log/filebeat/filebeat.log

# Validar configuración
sudo filebeat test config -e

# Probar conectividad con Logstash
sudo filebeat test output -e

# Debug completo (mucho output — usar con cabeza)
sudo filebeat -e -d "*" 2>&1 | head -100

# Debug solo del input
sudo filebeat -e -d "input" 2>&1 | grep -i "harvester\|reader\|error"

# Reiniciar
sudo systemctl restart filebeat

# Ver métricas internas de Filebeat
curl -s http://localhost:5066/stats | python3 -m json.tool
```

---

## Estrategia de etiquetado — tags y fields

Los `tags` y `fields` son los que permiten filtrar en Kibana. Hay que usarlos
con criterio desde el principio porque cambiarlos después implica reindexar.

**`tags`** — para categorías rápidas, van como array. Útil para agrupar:
```yaml
tags: ["app", "sentry-project"]
# En Kibana: tags : "app"  o  tags : "sentry-project"
```

**`fields`** — para metadatos estructurados con valor concreto:
```yaml
fields:
  log_category: "apitest"    # define el índice en Logstash
  app_name: "apitest"        # define el índice en Logstash
  stack: "nodejs"            # información extra para filtros
```

**Regla general:** `app_name` y `log_category` son los dos campos que Logstash
usa para el enrutamiento — sin ellos todo va al índice fallback `logstash-*`.
El resto de fields son opcionales pero enriquecen las búsquedas en Kibana.
