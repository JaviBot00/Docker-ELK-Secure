# 🐳 Docker ELK Secure

Entorno ELK (Elasticsearch, Logstash, Kibana) dockerizado con **licencia Basic gratuita e ilimitada**, seguridad habilitada y listo para producción. Diseñado para centralizar logs de múltiples servidores mediante agentes **Filebeat instalados en los clientes**.

```cmd
┌─────────────────────────────────────────────────────────────────┐
│  Servidor Central (Docker)                                      │
│                                                                 │
│  ┌───────────────┐     ┌──────────────────┐   ┌─────────────┐   │
│  │ Elasticsearch │◄────│    Logstash      │◄──│  :5044      │   │
│  │    :9200      │     │  :5044 / :50000  │   │  (Beats)    │   │
│  └───────┬───────┘     └──────────────────┘   └─────────────┘   │
│          │              Enruta a índices:                       │
│  ┌───────▼───────┐       apps-{nombre}-*                        │
│  │    Kibana     │       sistema-*                              │
│  │    :5601      │       docker-*                               │
│  └───────────────┘                                              │
└─────────────────────────────────────────────────────────────────┘
          ▲                  ▲                  ▲
          │                  │                  │
┌─────────┴───┐   ┌──────────┴───┐   ┌──────────┴───┐
│  Servidor A │   │  Servidor B  │   │  Servidor C  │
│  Filebeat   │   │   Filebeat   │   │   Filebeat   │
│  (paquete)  │   │   (paquete)  │   │   (paquete)  │
└─────────────┘   └──────────────┘   └──────────────┘
```

---

## 📋 Requisitos previos

- Docker >= 20.x y Docker Compose >= 2.x
- Al menos **4 GB de RAM** disponible para el servidor ELK
- Clientes con Ubuntu/Debian 20.04+ (para instalación de Filebeat vía APT)

---

## ⚙️ Configuración inicial obligatoria

### 1. Variables de entorno (seguridad)

Copia el fichero de ejemplo y edita las contraseñas **antes** de lanzar los contenedores.
El fichero `.env` está excluido del repositorio por `.gitignore`.

```bash
cp .env.example .env
nano .env
```

Contenido de `.env` a rellenar:

```env
ELASTIC_PASSWORD=CambiaMeAhora!
KIBANA_SYSTEM_PASSWORD=OtraPasswordSegura!
LOGSTASH_SYSTEM_PASSWORD=YOtraPasswordMas!
```

> ⚠️ **Nunca** uses contraseñas por defecto en producción ni hagas commit del `.env` con valores reales.

### 2. Parámetro del kernel (Linux)

Elasticsearch requiere un límite alto de mapas de memoria virtual. Sin esto, el contenedor crashea al arrancar.

```bash
# Aplicar de forma temporal (se pierde al reiniciar)
sudo sysctl -w vm.max_map_count=262144

# Aplicar de forma permanente
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

## 🚀 Despliegue del servidor ELK

```bash
# Clonar el repositorio
git clone <url-del-repo>
cd docker-elk-secure

# Configurar variables de entorno
cp .env.example .env && nano .env

# Levantar el stack
docker compose up -d

# Verificar que todo está corriendo
docker compose ps
docker compose logs -f
```

### Verificar que Elasticsearch responde

```bash
curl -u elastic:${ELASTIC_PASSWORD} http://localhost:9200/_cluster/health?pretty
```

Deberías ver `"status": "green"` o `"yellow"` — yellow es normal en instalaciones single-node.

---

## 🔑 Configurar la contraseña del usuario `kibana_system`

Este paso es necesario **una sola vez** tras el primer arranque. Ni Kibana ni Logstash pueden conectarse
a Elasticsearch hasta que se ejecute.

```bash
# Espera 30-60 segundos a que Elasticsearch esté listo, luego:
curl -u elastic:${ELASTIC_PASSWORD} \
  -X POST http://localhost:9200/_security/user/kibana_system/_password \
  -H "Content-Type: application/json" \
  -d '{"password": "'"${KIBANA_SYSTEM_PASSWORD}"'"}'
```

```bash
# Espera 30-60 segundos a que Elasticsearch esté listo, luego:
curl -u elastic:${ELASTIC_PASSWORD} \
  -X POST http://localhost:9200/_security/user/logstash_system/_password \
  -H "Content-Type: application/json" \
  -d '{"password": "'"${LOGSTASH_SYSTEM_PASSWORD}"'"}'
```

Si la respuesta es `{}` el usuario se ha actualizado correctamente. Reinicia Kibana:

```bash
docker compose restart kibana
```

---

## 🌐 Acceso a los servicios

| Servicio       | URL                        | Credenciales                      |
|---|---|---|
| Kibana         | http://localhost:5601      | `elastic` / `${ELASTIC_PASSWORD}` |
| Elasticsearch  | http://localhost:9200      | `elastic` / `${ELASTIC_PASSWORD}` |
| Logstash Beats | tcp://localhost:5044       | (recibe de Filebeat, no es web)   |
| Logstash TCP   | tcp://localhost:50000      | (input TCP genérico con JSON)     |

---

## 🗂️ Estructura del proyecto

```cmd
docker-elk-secure/
│
├── docker-compose.yml              # Stack ELK principal
├── docker-compose-opensource.yml   # Alternativa con OpenSearch
├── docker-compose-filebeat.yml     # Filebeat como contenedor (para el propio host)
├── .env.example                    # Plantilla de variables de entorno
├── .gitignore                      # Excluye .env y datos sensibles
│
├── logstash/
│   ├── config/
│   │   └── logstash.yml            # Configuración general de Logstash
│   └── pipeline/
│       └── logstash.conf           # Pipeline: enruta logs a índices por tipo/app
│
├── filebeat/
│   └── filebeat.yml                # Config base de Filebeat para clientes
│
└── docs/
    ├── KIBANA-SETUP.md             # Configuración de Kibana, Data Views y Dashboards
    └── LOGSTASH.md                 # Pipeline, enrutamiento de índices y parseo
```

---

## 📡 Instalación de Filebeat en clientes (Linux/Debian/Ubuntu)

### 1. Instalar Filebeat

```bash
# Importar la clave GPG de Elastic
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch \
  | sudo gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg

# Añadir el repositorio (versión 8.x — debe coincidir con el stack Docker)
echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] \
  https://artifacts.elastic.co/packages/8.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-8.x.list

# Instalar
sudo apt-get update && sudo apt-get install -y filebeat
```

### 2. Configurar Filebeat

```bash
# Copiar la configuración del repositorio
sudo cp ./filebeat/filebeat.yml /etc/filebeat/filebeat.yml
sudo nano /etc/filebeat/filebeat.yml
```

Los campos obligatorios a editar en cada cliente:

```yaml
fields:
  server_name: "nombre-de-este-servidor"   # identificador único del cliente
  environment: "production"                # production | staging | development
  location: "datacenter-mad"              # opcional: zona, rack, proveedor

output.logstash:
  hosts: ["IP_DEL_SERVIDOR_ELK:5044"]     # IP real del servidor ELK
```

### 3. Activar los bloques de logs que apliquen

Cada tipo de log tiene su propio bloque en `filebeat.yml` con `enabled: true/false`.
Activa solo los que existan en ese servidor:

```yaml
# Ejemplo: activar nginx y mysql, dejar apache desactivado
- type: log
  enabled: true    # ← nginx activo
  paths:
    - /var/log/nginx/*.log
  tags: ["nginx", "web"]
  fields:
    log_category: "nginx"

- type: log
  enabled: false   # ← apache inactivo en este servidor
  ...
```

### 4. Validar y arrancar

```bash
# Validar la configuración antes de arrancar
sudo filebeat test config -e
sudo filebeat test output -e

# Arrancar y habilitar en el inicio del sistema
sudo systemctl start filebeat
sudo systemctl enable filebeat

# Verificar que funciona
sudo systemctl status filebeat
sudo journalctl -u filebeat -f
```

---

## 🪟 Instalación de Filebeat en clientes Windows (PowerShell)

```powershell
# Descargar Filebeat
Invoke-WebRequest -Uri "https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.12.2-windows-x86_64.zip" `
  -OutFile "filebeat.zip"

# Descomprimir e instalar
Expand-Archive -Path "filebeat.zip" -DestinationPath "C:\Program Files\"
Rename-Item "C:\Program Files\filebeat-8.12.2-windows-x86_64" "C:\Program Files\Filebeat"

# Copiar configuración
Copy-Item "filebeat.yml" "C:\Program Files\Filebeat\filebeat.yml"

# Instalar como servicio de Windows y arrancar
cd "C:\Program Files\Filebeat"
.\install-service-filebeat.ps1
Start-Service filebeat
```

---

## 🗃️ Estrategia de índices y pipeline de Logstash

El pipeline de Logstash enruta cada log al índice correcto según los campos
`app_name` y `log_category` que manda Filebeat:

```cmd
app_name presente    →  apps-{app_name}-YYYY.MM.dd
log_category=auth    →  sistema-YYYY.MM.dd
tag docker           →  docker-YYYY.MM.dd
resto                →  logstash-YYYY.MM.dd
```

Logstash también añade un campo `severity` automático (`critical` / `error` / `warning` / `info`)
analizando el texto de cada mensaje.

> Para la documentación completa del pipeline, parseo grok, enrutamiento y troubleshooting
> de Logstash consulta **[LOGSTASH.md](./docs/LOGSTASH.md)**.

---

## 🔄 Comandos de gestión habituales

```bash
# Ver estado de los contenedores
docker compose ps

# Ver logs en tiempo real (todos o uno solo)
docker compose logs -f
docker compose logs -f logstash

# Reiniciar un servicio concreto
docker compose restart kibana

# Parar el stack manteniendo los datos
docker compose down

# Parar el stack y BORRAR todos los datos
docker compose down -v

# Actualizar imágenes a la última versión
docker compose pull && docker compose up -d
```

---

## 🗃️ Gestión del ciclo de vida de los índices (ILM)

Sin política de retención el disco se llenará con el tiempo. Aplica una política ILM
que borre índices antiguos automáticamente.

Desde Kibana: **Stack Management → Index Lifecycle Policies → Create Policy**

O vía API (borra índices con más de 30 días):

```bash
curl -u elastic:${ELASTIC_PASSWORD} \
  -X PUT http://localhost:9200/_ilm/policy/elk-cleanup \
  -H "Content-Type: application/json" \
  -d '{
    "policy": {
      "phases": {
        "delete": {
          "min_age": "30d",
          "actions": { "delete": {} }
        }
      }
    }
  }'
```

Aplica la política a todos los índices del stack:

```bash
# Para apps-*
curl -u elastic:${ELASTIC_PASSWORD} \
  -X PUT http://localhost:9200/apps-*/_settings \
  -H "Content-Type: application/json" \
  -d '{"index.lifecycle.name": "elk-cleanup"}'

# Para sistema-* y docker-*
curl -u elastic:${ELASTIC_PASSWORD} \
  -X PUT http://localhost:9200/sistema-*,docker-*/_settings \
  -H "Content-Type: application/json" \
  -d '{"index.lifecycle.name": "elk-cleanup"}'
```

---

## 🛠️ Solución de problemas frecuentes

### Elasticsearch no arranca / exit code 137

```bash
docker compose logs elasticsearch | tail -30
```

Causa más común: `vm.max_map_count` demasiado bajo. Aplica el `sysctl` del paso inicial.

### Kibana muestra "Kibana server is not ready yet"

Espera 60-90 segundos. Si persiste, lo más habitual es que la contraseña de `kibana_system`
no se haya actualizado:

```bash
docker compose logs kibana | grep -i error
```

Repite el paso de configuración de contraseñas y reinicia Kibana.

### Logstash no conecta a Elasticsearch

Logstash puede arrancar antes de que ES esté listo. Solución:

```bash
docker compose restart logstash
```

### Filebeat conecta pero no llegan datos / no se crean índices

```bash
# En el servidor cliente
sudo filebeat test config -e        # verifica sintaxis del fichero
sudo filebeat test output -e        # verifica conectividad con Logstash
sudo journalctl -u filebeat -f      # ver logs del agente en tiempo real

# Comprobar que el puerto es accesible
nc -zv IP_DEL_SERVIDOR_ELK 5044
```

### Los índices no se crean con el nombre esperado (apps-*, sistema-*)

Verifica que Logstash tiene acceso a la variable `ELASTIC_PASSWORD`:

```bash
docker compose logs logstash | grep -i "error\|password\|connection"
```

Si ves errores de autenticación, asegúrate de que en `docker-compose.yml` el servicio
`logstash` tiene `ELASTIC_PASSWORD=${ELASTIC_PASSWORD}` en su sección `environment`.

---

## 🔒 Consideraciones de seguridad para producción

- Usa contraseñas fuertes en `.env` y nunca las commitees al repositorio.
- Habilita TLS entre Filebeat y Logstash (puerto 5044 con certificados mutuos).
- Restringe los puertos 9200, 5601 y 5044 con firewall — solo IPs conocidas.
- Crea usuarios de Elasticsearch con roles mínimos para Logstash y Kibana
  en lugar de usar el superusuario `elastic` para todo.
- Aplica ILM para evitar que el disco se llene con índices antiguos.

---

## 🔀 Alternativa: OpenSearch (sin licencia Elastic)

Si prefieres una alternativa 100% open source sin dependencia de Elastic:

```bash
docker compose -f docker-compose-opensource.yml up -d
```

Incluye OpenSearch + OpenSearch Dashboards. Compatible con Filebeat mediante
el output de Logstash/Elasticsearch estándar.

---

## 📚 Referencias

- [Documentación oficial de Elasticsearch 8.x](https://www.elastic.co/guide/en/elasticsearch/reference/8.12/index.html)
- [Filebeat: referencia de inputs](https://www.elastic.co/guide/en/beats/filebeat/8.12/filebeat-input-log.html)
- [Logstash: filtros y outputs](https://www.elastic.co/guide/en/logstash/8.12/index.html)
- [Index Lifecycle Management](https://www.elastic.co/guide/en/elasticsearch/reference/8.12/index-lifecycle-management.html)
- [Elastic Basic License](https://www.elastic.co/subscriptions)
