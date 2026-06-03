# 🐳 Docker ELK Secure

Entorno ELK (Elasticsearch, Logstash, Kibana) dockerizado con **licencia Basic gratuita e ilimitada**, seguridad habilitada y listo para producción. Diseñado para centralizar logs de múltiples servidores mediante agentes **Filebeat instalados en los clientes**.

```cmd
┌──────────────────────────────────────────────────────────┐
│  Servidor Central (Docker)                               │
│  ┌─────────────┐   ┌──────────────┐   ┌───────────────┐  │
│  │Elasticsearch│◄──│   Logstash   │◄──│   :5044       │  │
│  │   :9200     │   │  :5044/:50000│   │  (Beats input)│  │
│  └──────┬──────┘   └──────────────┘   └───────────────┘  │
│         │                                                │
│  ┌──────▼──────┐                                         │
│  │   Kibana    │                                         │
│  │   :5601     │                                         │
│  └─────────────┘                                         │
└──────────────────────────────────────────────────────────┘
         ▲                ▲                ▲
         │                │                │
┌────────┴──┐   ┌─────────┴──┐   ┌─────────┴──┐
│ Servidor A│   │ Servidor B │   │ Servidor C │
│ Filebeat  │   │  Filebeat  │   │  Filebeat  │
│ (paquete) │   │  (paquete) │   │  (paquete) │
└───────────┘   └────────────┘   └────────────┘
```

---

## 📋 Requisitos previos

- Docker >= 20.x y Docker Compose >= 2.x
- Al menos **4 GB de RAM** disponible para el servidor ELK
- Clientes con Ubuntu/Debian 20.04+ (para instalación de Filebeat vía APT)

---

## ⚙️ Configuración inicial obligatoria

### 1. Variables de entorno (seguridad)

Copia el fichero de ejemplo y edita las contraseñas **antes** de lanzar los contenedores. El fichero `.env` está excluido del repositorio por `.gitignore`.

```bash
cp .env.example .env
nano .env   # o el editor que prefieras
```

Contenido de `.env` a rellenar:

```env
ELASTIC_PASSWORD=CambiaMeAhora!
KIBANA_SYSTEM_PASSWORD=OtraPasswordSegura!
LOGSTASH_SYSTEM_PASSWORD=YOtraPasswordMas!
```

> ⚠️ **Nunca** uses contraseñas por defecto en producción ni hagas commit del `.env` con valores reales.

### 2. Parámetro del kernel (Linux/macOS)

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

Deberías ver `"status": "green"` o `"yellow"` (yellow es normal en single-node).

---

## 🔑 Configurar la contraseña del usuario `kibana_system`

Este paso es necesario **una sola vez** tras el primer arranque, antes de que Kibana pueda conectarse a Elasticsearch.

```bash
# Espera a que Elasticsearch esté listo (puede tardar 30-60 segundos)
curl -u elastic:${ELASTIC_PASSWORD} \
  -X POST http://localhost:9200/_security/user/kibana_system/_password \
  -H "Content-Type: application/json" \
  -d '{"password": "'"${KIBANA_SYSTEM_PASSWORD}"'"}'
```

Si ves `{}` como respuesta, el usuario se ha actualizado correctamente. Después puedes reiniciar Kibana:

```bash
docker compose restart kibana
```

---

## 🌐 Acceso a los servicios

| Servicio       | URL                        | Credenciales                        |
|----------------|----------------------------|-------------------------------------|
| Kibana         | http://localhost:5601      | `elastic` / `${ELASTIC_PASSWORD}`   |
| Elasticsearch  | http://localhost:9200      | `elastic` / `${ELASTIC_PASSWORD}`   |
| Logstash Beats | http://localhost:5044      | (recibe de Filebeat, no es web)     |
| Logstash TCP   | tcp://localhost:50000      | (input TCP genérico)                |

---

## 📡 Instalación de Filebeat en clientes (Linux/Debian/Ubuntu)

Ejecuta los siguientes pasos **en cada servidor cliente** que quieras monitorizar.

### 1. Instalar Filebeat

```bash
# Importar la clave GPG de Elastic
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch \
  | sudo gpg --dearmor -o /etc/apt/keyrings/elastic-keyring.gpg

# Añadir el repositorio (versión 8.x, igual que el stack Docker)
echo "deb [signed-by=/etc/apt/keyrings/elastic-keyring.gpg] \
  https://artifacts.elastic.co/packages/8.x/apt stable main" \
  | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list

# Instalar
sudo apt-get update && sudo apt-get install -y filebeat
```

### 2. Configurar Filebeat

Copia la configuración de ejemplo del repositorio y edita la IP del servidor ELK:

```bash
# Desde el servidor donde clonaste el repo
sudo cp ./filebeat/filebeat.yml /etc/filebeat/filebeat.yml

# Editar la IP del servidor central
sudo nano /etc/filebeat/filebeat.yml
```

Busca la línea `hosts: ["IP_DEL_SERVIDOR_ELK:5044"]` y sustitúyela por la IP real de tu servidor ELK.

**Identificar cada cliente con un nombre único** (muy recomendado cuando hay múltiples servidores):

```yaml
# Añadir al final del filebeat.yml en cada cliente
fields:
  server_name: "web-produccion-01"   # cambia por el nombre de este servidor
  environment: "production"           # production / staging / development
fields_under_root: true
```

### 3. Arrancar y habilitar el servicio

```bash
# Iniciar Filebeat
sudo systemctl start filebeat
sudo systemctl enable filebeat

# Comprobar que está corriendo y sin errores
sudo systemctl status filebeat
sudo journalctl -u filebeat -f
```

### 4. Verificar que llegan datos a Kibana

En Kibana ve a **Management → Stack Management → Index Management** y busca índices `logstash-*`. Si aparecen, los logs están llegando correctamente.

---

## 🪟 Instalación de Filebeat en clientes Windows (PowerShell)

```powershell
# Descargar Filebeat para Windows
Invoke-WebRequest -Uri "https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.12.2-windows-x86_64.zip" `
  -OutFile "filebeat.zip"

# Descomprimir
Expand-Archive -Path "filebeat.zip" -DestinationPath "C:\Program Files\"
Rename-Item "C:\Program Files\filebeat-8.12.2-windows-x86_64" "C:\Program Files\Filebeat"

# Copiar configuración
Copy-Item "filebeat.yml" "C:\Program Files\Filebeat\filebeat.yml"

# Instalar como servicio de Windows
cd "C:\Program Files\Filebeat"
.\install-service-filebeat.ps1

# Arrancar el servicio
Start-Service filebeat
```

---

## 🗂️ Estructura del proyecto

```cmd
docker-elk-secure/
│
├── docker-compose.yml              # Stack ELK principal (Elastic + Kibana + Logstash)
├── docker-compose-opensource.yml   # Alternativa con OpenSearch
├── docker-compose-filebeat.yml     # Filebeat como contenedor (opcional, para el propio host)
├── .env.example                    # Plantilla de variables de entorno
├── .gitignore                      # Excluye .env y otros ficheros sensibles
│
├── logstash/
│   ├── config/
│   │   └── logstash.yml            # Configuración general de Logstash
│   └── pipeline/
│       └── logstash.conf           # Pipeline: recibe de Beats → envía a Elasticsearch
│
├── filebeat/
│   └── filebeat.yml                # Config de Filebeat para instalar en clientes
│
└── README.md
```

---

## 🔄 Comandos de gestión habituales

```bash
# Ver estado de los contenedores
docker compose ps

# Ver logs en tiempo real
docker compose logs -f
docker compose logs -f elasticsearch   # solo un servicio

# Reiniciar un servicio concreto
docker compose restart kibana

# Parar el stack (mantiene los datos)
docker compose down

# Parar el stack y BORRAR los datos (volúmenes)
docker compose down -v

# Actualizar imágenes
docker compose pull
docker compose up -d
```

---

## 🗃️ Gestión del ciclo de vida de los índices (ILM)

Sin una política de retención, Elasticsearch llenará el disco con el tiempo. Se recomienda configurar una política ILM (Index Lifecycle Management) desde Kibana:

1. Ve a **Stack Management → Index Lifecycle Policies → Create Policy**
2. Configura una fase de borrado automático, por ejemplo:
   - **Hot**: 7 días (índices activos)
   - **Delete**: borrar después de 30 días

O vía API:

```bash
curl -u elastic:${ELASTIC_PASSWORD} \
  -X PUT http://localhost:9200/_ilm/policy/logstash-cleanup \
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

---

## 🛠️ Solución de problemas frecuentes

### Elasticsearch no arranca / exit code 137

Síntoma: el contenedor muere nada más arrancar.

```bash
# Ver la causa exacta
docker compose logs elasticsearch | tail -30
```

Causa más común: `vm.max_map_count` demasiado bajo. Aplica el `sysctl` del paso inicial.

### Kibana muestra "Kibana server is not ready yet"

Espera 60-90 segundos después del `docker compose up`. Si persiste:

```bash
docker compose logs kibana | grep -i error
```

Lo más habitual es que la contraseña de `kibana_system` no se haya actualizado. Repite el paso de configuración de contraseñas.

### Filebeat conecta pero no llegan datos

```bash
# En el cliente, ver logs de Filebeat en detalle
sudo filebeat -e -d "*"

# Comprobar conectividad al puerto Logstash
nc -zv IP_DEL_SERVIDOR_ELK 5044
```

Causas comunes: firewall bloqueando el puerto 5044, IP incorrecta en la configuración, o ruta de logs inexistente.

### Logstash no se conecta a Elasticsearch

Si Logstash levanta antes de que Elasticsearch esté completamente listo, puede fallar. Solucion:

```bash
docker compose restart logstash
```

---

## 🔒 Consideraciones de seguridad para producción

- Usa siempre contraseñas fuertes en `.env` y nunca las commitees al repositorio.
- Considera habilitar TLS entre Filebeat y Logstash (puerto 5044 con certificados).
- Restringe el acceso a los puertos 9200, 5601 y 5044 mediante firewall (`ufw` o reglas de red) solo a IPs conocidas.
- Crea usuarios de Elasticsearch con roles mínimos para Logstash y Kibana (no usar el usuario `elastic` para todo).
- Revisa periódicamente los índices y aplica ILM para evitar que el disco se llene.

---

## 🔀 Alternativa: OpenSearch (sin licencia Elastic)

Si prefieres una alternativa 100% open source sin dependencia de Elastic:

```bash
docker compose -f docker-compose-opensource.yml up -d
```

Incluye OpenSearch + OpenSearch Dashboards. Compatibilidad con Filebeat mediante el output de Logstash/Elasticsearch estándar.

---

## 📚 Referencias

- [Documentación oficial de Elasticsearch 8.x](https://www.elastic.co/guide/en/elasticsearch/reference/8.12/index.html)
- [Filebeat: inputs y configuración](https://www.elastic.co/guide/en/beats/filebeat/8.12/filebeat-input-log.html)
- [Index Lifecycle Management](https://www.elastic.co/guide/en/elasticsearch/reference/8.12/index-lifecycle-management.html)
- [Elastic Basic License](https://www.elastic.co/subscriptions)
