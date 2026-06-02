# Docker ELK Secure

🐳 Entorno ágil de ELK (Elasticsearch, Logstash y Kibana) corriendo en Docker. Configurado específicamente con la licencia basic gratuita e ilimitada y con la seguridad optimizada para desarrollo local y análisis rápido de logs, libre de periodos de prueba o suscripciones.

## Requisitos previos (Linux/macOS)

Antes de levantar el entorno, recuerda asignar suficiente memoria virtual en el host:

```bash
sudo sysctl -w vm.max_map_count=262144
```

## Despliegue rápido

```bash
docker-compose up -d
```
