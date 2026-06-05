# 🐳 Docker ELK Secure

Entorno ELK (Elasticsearch, Logstash, Kibana) dockerizado con **licencia Basic gratuita
e ilimitada**, seguridad habilitada y listo para producción. Centraliza logs de múltiples
servidores mediante agentes **Filebeat instalados en los clientes**.

```
┌─────────────────────────────────────────────────────────────────┐
│  Servidor Central (Docker)                                      │
│                                                                 │
│  ┌───────────────┐     ┌──────────────────┐                     │
│  │ Elasticsearch │◄────│    Logstash      │◄── :5044 (Filebeat) │
│  │    :9200      │     │  :5044 / :50000  │◄── :50000 (TCP JSON)│
│  └───────┬───────┘     └──────────────────┘                     │
│          │              Enruta a índices:                       │
│  ┌───────▼───────┐       apps-{nombre}-*                        │
│  │    Kibana     │       sistema-*                              │
│  │    :5601      │       docker-*                               │
│  └───────────────┘                                              │
└─────────────────────────────────────────────────────────────────┘
          ▲                  ▲                   ▲
┌─────────┴───┐   ┌──────────┴───┐   ┌──────────┴───┐
│  Servidor A │   │  Servidor B  │   │  Servidor C  │
│  Filebeat   │   │   Filebeat   │   │   Filebeat   │
│  (paquete)  │   │   (paquete)  │   │   (paquete)  │
└─────────────┘   └──────────────┘   └──────────────┘
```

> 📘 **¿Ya tienes el stack montado?** La guía de operaciones para el día a día
> está en [`docs/`](./docs/README.md).

---

## Requisitos previos

- Docker >= 20.x y Docker Compose >= 2.x
- Al menos **4 GB de RAM** en el servidor central
- Clientes Linux: Ubuntu/Debian 20.04+ para instalación de Filebeat vía APT

---

## Estructura del proyecto

```
docker-elk-secure/
│
├── docker-compose.yml              # Stack ELK principal
├── docker-compose-opensource.yml   # Alternativa con OpenSearch
├── docker-compose-filebeat.yml     # Filebeat como contenedor (para el propio host)
├── .env.example                    # Plantilla de variables de entorno
├── .gitignore
│
├── logstash/
│   ├── config/logstash.yml         # Configuración general de Logstash
│   └── pipeline/logstash.conf      # Pipeline: recibe → enruta → escribe índices
│
├── filebeat/
│   └── filebeat.yml.example        # Plantilla para instalar en clientes
│
└── docs/                           # Guía de operaciones para el técnico
    ├── README.md                   # Índice y mapa de la documentación
    ├── 01-ARQUITECTURA.md
    ├── 02-FILEBEAT.md
    ├── 03-LOGSTASH.md
    ├── 04-KIBANA.md
    ├── 05-OPERACIONES.md
    └── 06-TROUBLESHOOTING.md
```

---

## Despliegue inicial — paso a paso

### 1. Variables de entorno

Copia la plantilla y edita las contraseñas **antes** de arrancar nada.
El fichero `.env` está en `.gitignore` — nunca se sube al repositorio.

```bash
cp .env.example .env
nano .env
```

```env
ELASTIC_PASSWORD=CambiaMeAhora!
KIBANA_SYSTEM_PASSWORD=OtraPasswordSegura!
LOGSTASH_SYSTEM_PASSWORD=YOtraPasswordMas!
```

Las tres contraseñas tienen roles distintos — ver detalle en
[¿Cuáles son las 3 contraseñas?](#cuáles-son-las-3-contraseñas).

### 2. Parámetro del kernel

Elasticsearch necesita un límite alto de memoria virtual. Sin esto crashea al arrancar.

```bash
# Temporal (se pierde al reiniciar)
sudo sysctl -w vm.max_map_count=262144

# Permanente
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 3. Arrancar el stack

```bash
docker compose up -d
docker compose ps          # verificar que los 3 contenedores están Up
docker compose logs -f     # seguir los logs de arranque
```

Elasticsearch tarda 30-60 segundos en estar listo. Kibana y Logstash
pueden mostrar errores de conexión durante ese tiempo — es normal.

### 4. Configurar la contraseña de kibana_system

Este paso es obligatorio **una sola vez** tras el primer arranque.
Kibana no puede conectarse a Elasticsearch hasta que se ejecute.

```bash
curl -u elastic:${ELASTIC_PASSWORD} \
  -X POST http://localhost:9200/_security/user/kibana_system/_password \
  -H "Content-Type: application/json" \
  -d '{"password": "'"${KIBANA_SYSTEM_PASSWORD}"'"}'
```

Respuesta esperada: `{}`. Luego:

```bash
docker compose restart kibana
```

### 5. Verificar que todo responde

```bash
# Elasticsearch
curl -u elastic:${ELASTIC_PASSWORD} http://localhost:9200/_cluster/health?pretty

# Kibana — abrir en el navegador
http://localhost:5601   # usuario: elastic
```

Estado `"yellow"` en Elasticsearch es normal en instalación single-node.

---

## Acceso a los servicios

| Servicio | URL | Credenciales |
|---|---|---|
| Kibana | http://localhost:5601 | `elastic` / `${ELASTIC_PASSWORD}` |
| Elasticsearch | http://localhost:9200 | `elastic` / `${ELASTIC_PASSWORD}` |
| Logstash Beats | tcp://localhost:5044 | — recibe de Filebeat |
| Logstash TCP | tcp://localhost:50000 | — input JSON genérico |

---

## Instalar Filebeat en un servidor cliente

### Linux (Debian/Ubuntu)

```bash
# Clave GPG
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch \
  | sudo gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg

# Repositorio 8.x — debe coincidir con la versión del stack
echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] \
  https://artifacts.elastic.co/packages/8.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-8.x.list

# Instalar
sudo apt-get update && sudo apt-get install -y filebeat

# Copiar la plantilla del repositorio
sudo cp ./filebeat/filebeat.yml.example /etc/filebeat/filebeat.yml
sudo nano /etc/filebeat/filebeat.yml
```

### Windows (PowerShell)

```powershell
Invoke-WebRequest -Uri "https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.12.2-windows-x86_64.zip" `
  -OutFile "filebeat.zip"
Expand-Archive -Path "filebeat.zip" -DestinationPath "C:\Program Files\"
Rename-Item "C:\Program Files\filebeat-8.12.2-windows-x86_64" "C:\Program Files\Filebeat"
Copy-Item "filebeat.yml.example" "C:\Program Files\Filebeat\filebeat.yml"
cd "C:\Program Files\Filebeat"
.\install-service-filebeat.ps1
Start-Service filebeat
```

### Qué editar en cada cliente

```yaml
# Identidad del servidor — obligatorio cambiar
fields:
  server_name: "nombre-unico-del-servidor"
  environment: "production"        # production | staging | development
  location: "datacenter-mad"

# IP del servidor ELK
output.logstash:
  hosts: ["IP_DEL_SERVIDOR_ELK:5044"]
```

Luego activar con `enabled: true` solo los bloques de logs que existan
en ese servidor. Arrancar:

```bash
sudo filebeat test config -e && sudo filebeat test output -e
sudo systemctl start filebeat && sudo systemctl enable filebeat
```

---

## ¿Cuáles son las 3 contraseñas?

| Variable | Usuario | Dónde se usa |
|---|---|---|
| `ELASTIC_PASSWORD` | `elastic` (superusuario) | `docker-compose.yml` ES + Logstash, `logstash.conf`, curl de administración |
| `KIBANA_SYSTEM_PASSWORD` | `kibana_system` (interno) | `docker-compose.yml` Kibana, comando curl del paso 4 |
| `LOGSTASH_SYSTEM_PASSWORD` | `logstash_system` (monitorización) | `logstash/config/logstash.yml` |

---

## Comandos habituales

```bash
# Estado
docker compose ps

# Logs en tiempo real
docker compose logs -f
docker compose logs -f logstash

# Reiniciar un servicio
docker compose restart kibana

# Parar (conserva datos)
docker compose down

# Parar y BORRAR datos — irreversible
docker compose down -v

# Actualizar imágenes
docker compose pull && docker compose up -d
```

---

## Seguridad para producción

- Contraseñas fuertes en `.env` — nunca en el repositorio.
- Firewall: restringir puertos 9200, 5601 y 5044 solo a IPs conocidas.
- Considerar TLS entre Filebeat y Logstash (puerto 5044 con certificados).
- No usar el superusuario `elastic` para todo — crear usuarios con roles mínimos.
- Configurar ILM para evitar que el disco se llene con índices antiguos.

---

## Alternativa OpenSearch (sin licencia Elastic)

```bash
docker compose -f docker-compose-opensource.yml up -d
```

Incluye OpenSearch + OpenSearch Dashboards. Compatible con el mismo Filebeat.

---

## Referencias

- [Elasticsearch 8.x](https://www.elastic.co/guide/en/elasticsearch/reference/8.12/index.html)
- [Filebeat inputs](https://www.elastic.co/guide/en/beats/filebeat/8.12/filebeat-input-filestream.html)
- [Logstash](https://www.elastic.co/guide/en/logstash/8.12/index.html)
- [Index Lifecycle Management](https://www.elastic.co/guide/en/elasticsearch/reference/8.12/index-lifecycle-management.html)
- [Elastic Basic License](https://www.elastic.co/subscriptions)
