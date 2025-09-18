# Informe de Implementación del Proyecto TheHive + Cortex + MISP + Wazuh

**Tarea #997**  
**Fecha:** 18 de Septiembre de 2025  
**Estudiante:** Emmanuel García Peñaloza 

---

## 1. Introducción

En este informe se documenta el proceso completo de implementación del proyecto de integración de herramientas de ciberseguridad que incluye TheHive, Cortex, MISP y Wazuh utilizando Docker Compose en un entorno Windows 11.

### 1.1 Objetivos del Proyecto

- Implementar una plataforma integrada de gestión de incidentes de seguridad
- Configurar TheHive como plataforma central de gestión de casos
- Integrar Cortex para análisis automatizado de observables
- Preparar la arquitectura para integración futura con MISP y Wazuh
- Documentar todo el proceso de instalación y configuración

### 1.2 Alcance

La implementación se realizó en un entorno de testing utilizando contenedores Docker, incluyendo:
- TheHive 5.5.2-1 (Gestión de incidentes)
- Cortex 3.1.8-1 (Análisis de observables)
- Elasticsearch 7.17.28 (Motor de búsqueda e indexación)
- Cassandra 4.1.8 (Base de datos principal)
- Nginx 1.27.5 (Proxy reverso con SSL)

---

## 2. Descripción de los Componentes

### 2.1 TheHive

**Descripción:** Plataforma de gestión de respuesta a incidentes de seguridad (SIRP) de código abierto.

**Funcionalidades principales:**
- Gestión de casos e incidentes de seguridad
- Colaboración en equipo para investigaciones
- Seguimiento de tareas y observables
- Integración con herramientas de análisis externas

**Versión implementada:** 5.5.2-1
**Puerto de acceso:** 9000
**URL de acceso:** http://localhost:9000/thehive

### 2.2 Cortex

**Descripción:** Motor de análisis de observables que permite automatizar el análisis de indicadores de compromiso.

**Funcionalidades principales:**
- Análisis automatizado de IPs, dominios, hashes, URLs
- Integración con múltiples fuentes de threat intelligence
- Ejecución de analyzers y responders
- API REST para integración con TheHive

**Versión implementada:** 3.1.8-1
**Puerto de acceso:** 9001
**URL de acceso:** http://localhost:9001/cortex

### 2.3 Elasticsearch

**Descripción:** Motor de búsqueda y análisis distribuido utilizado como backend de indexación.

**Funcionalidades principales:**
- Indexación de datos de TheHive y Cortex
- Búsquedas rápidas y análisis de datos
- Almacenamiento de logs y métricas

**Versión implementada:** 7.17.28
**Configuración:** Modo single-node con autenticación habilitada

### 2.4 Cassandra

**Descripción:** Base de datos NoSQL distribuida utilizada como almacén principal de TheHive.

**Funcionalidades principales:**
- Almacenamiento de casos, tareas y observables
- Alta disponibilidad y escalabilidad
- Consistencia eventual

**Versión implementada:** 4.1.8
**Configuración:** Cluster de un solo nodo con autenticación

### 2.5 Nginx

**Descripción:** Servidor web y proxy reverso para acceso seguro a las aplicaciones.

**Funcionalidades principales:**
- Proxy reverso para TheHive y Cortex
- Terminación SSL/TLS
- Balanceador de carga (preparado para múltiples instancias)

**Versión implementada:** 1.27.5
**Configuración:** SSL con certificado autofirmado para testing

---

## 3. Arquitectura e Integración

### 3.1 Diagrama de Arquitectura

```
[Internet] 
    |
    v
[Nginx Proxy (443)] 
    |
    +-- TheHive (9000)
    |       |
    |       +-- Cassandra (9042)
    |       +-- Elasticsearch (9200)
    |
    +-- Cortex (9001)
            |
            +-- Elasticsearch (9200)
```

### 3.2 Flujo de Datos

1. **Acceso externo:** Los usuarios acceden a través de Nginx en el puerto 443 (HTTPS)
2. **Enrutamiento:** Nginx direcciona las peticiones a TheHive o Cortex según la URL
3. **Almacenamiento:** TheHive almacena datos estructurados en Cassandra
4. **Indexación:** Tanto TheHive como Cortex utilizan Elasticsearch para búsquedas
5. **Análisis:** Cortex procesa observables y devuelve resultados a TheHive

### 3.3 Red Docker

- **Red:** `testing_thehive-cortex-network`
- **Tipo:** Bridge network
- **Resolución DNS:** Automática entre contenedores

---

## 4. Proceso de Implementación

### 4.1 Requisitos Previos

#### 4.1.1 Hardware
- **CPU:** Mínimo 4 vCPUs (recomendado para testing)
- **Memoria RAM:** Mínimo 8 GB (2 GB por contenedor)
- **Almacenamiento:** 20 GB libres mínimo
- **Sistema Operativo:** Windows 11 Pro

#### 4.1.2 Software
- Docker Desktop para Windows (versión utilizada: 28.0.1)
- Docker Compose plugin (versión utilizada: v2.33.1-desktop.1)
- PowerShell 5.1 o superior
- Git (para clonar el repositorio)

### 4.2 Pasos de Instalación

#### 4.2.1 Preparación del Entorno

1. **Clonación del repositorio:**
   ```bash
   git clone https://github.com/StrangeBeeCorp/docker.git
   cd docker/testing
   ```

2. **Verificación de requisitos:**
   ```powershell
   docker --version
   docker compose version
   ```

#### 4.2.2 Configuración de Variables de Entorno

Se creó el archivo `.env` con las siguientes configuraciones:

```env
# System variables (Windows values)
UID=1000
GID=1000

# ElasticSearch password
elasticsearch_password=ElasticSearch2024!SecurePass

# Cortex specific configuration
cortex_docker_job_directory=C:/Users/20030/Documents/GitHub/docker/testing/cortex/cortex-jobs

# Containers versions
cassandra_image_version=4.1.8
elasticsearch_image_version=7.17.28
thehive_image_version=5.5.2-1
cortex_image_version=3.1.8-1
nginx_image_version=1.27.5

# Nginx configuration
nginx_server_name=localhost
nginx_ssl_trusted_certificate=""
```

#### 4.2.3 Configuración de TheHive

Se crearon los archivos de configuración necesarios:

**index.conf:**
```hocon
db.janusgraph.index.search {
  backend = elasticsearch
  hostname = ["elasticsearch"]
  index-name = thehive
  elasticsearch.http.auth {
    type = "basic"
    basic {
      username = "elastic"
      password = "ElasticSearch2024!SecurePass"
    }
  }
}
```

**secret.conf:**
```hocon
play.http.secret.key="TheHiveSecretKey2024ForProductionChangeThis64BytesLong"
```

#### 4.2.4 Configuración de Cortex

**index.conf:**
```hocon
search {
  index = cortex
  user = "elastic"
  password = "ElasticSearch2024!SecurePass"
}
```

**secret.conf:**
```hocon
play.http.secret.key="CortexSecretKey2024ForProductionChangeThis64BytesLongKey"
```

#### 4.2.5 Configuración de Certificados SSL

Se generaron certificados autofirmados para nginx:

```powershell
docker run --rm -v "${PWD}/nginx/certs:/certs" alpine/openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /certs/server.key -out /certs/server.crt -subj "/C=ES/ST=Madrid/L=Madrid/O=Testing/OU=IT Department/CN=localhost"
```

#### 4.2.6 Configuración de Permisos

Se configuraron los permisos necesarios para Elasticsearch:

```powershell
icacls .\elasticsearch\data /grant Everyone:F /T
icacls .\elasticsearch\logs /grant Everyone:F /T
```

### 4.3 Despliegue de Servicios

#### 4.3.1 Levantamiento de Contenedores

```powershell
docker compose up -d
```

**Resultado del despliegue:**
- ✅ elasticsearch: Iniciado y saludable en 16.5s
- ✅ cassandra: Iniciado y saludable en 81.5s
- ✅ cortex: Iniciado y saludable en 16.5s
- ✅ thehive: Iniciado correctamente en 81.4s
- ✅ nginx: Iniciado tras configuración de certificados

#### 4.3.2 Verificación de Servicios

```powershell
# Verificación de estado
docker compose ps

# Verificación de conectividad
Invoke-WebRequest -Uri "http://localhost:9000/thehive/api/status" -UseBasicParsing
Invoke-WebRequest -Uri "http://localhost:9001/cortex/api/status" -UseBasicParsing
```

**Resultados:**
- TheHive: HTTP 200 - Operativo
- Cortex: HTTP 200 - Operativo
- Nginx: HTTPS funcional con certificado autofirmado

---

## 5. Pruebas y Validación

### 5.1 Pruebas de Conectividad

| Servicio | URL | Puerto | Estado | Tiempo de Respuesta |
|----------|-----|---------|---------|---------------------|
| TheHive | http://localhost:9000/thehive | 9000 | ✅ Operativo | <1s |
| Cortex | http://localhost:9001/cortex | 9001 | ✅ Operativo | <1s |
| Nginx HTTPS | https://localhost | 443 | ✅ Operativo | <1s |

### 5.2 Pruebas de Bases de Datos

| Base de Datos | Estado | Detalles |
|---------------|---------|----------|
| Elasticsearch | ✅ Healthy | Autenticación configurada, índices listos |
| Cassandra | ✅ Healthy | Keyspace TheHive configurado |

### 5.3 Verificación de Logs

Todos los servicios generan logs correctamente en sus respectivos directorios:
- `./elasticsearch/logs/`: Logs de Elasticsearch
- `./cassandra/logs/`: Logs de Cassandra
- `./thehive/logs/`: Logs de TheHive
- `./cortex/logs/`: Logs de Cortex

---

## 6. Configuración Post-Instalación

### 6.1 Acceso a TheHive

1. Acceder a http://localhost:9000/thehive
2. Configurar el primer usuario administrador
3. Configurar la organización
4. Configurar conectores y analizadores

### 6.2 Configuración de Cortex

1. Acceder a http://localhost:9001/cortex
2. Crear la primera organización
3. Configurar usuarios y roles
4. Instalar y configurar analizadores requeridos

### 6.3 Integración TheHive-Cortex

1. En TheHive, configurar la conexión a Cortex:
   - URL: http://cortex:9001
   - Configurar autenticación API
2. Probar la conectividad desde TheHive

---

## 7. Gestión y Mantenimiento

### 7.1 Comandos de Gestión

```powershell
# Iniciar servicios
docker compose up -d

# Detener servicios
docker compose down

# Ver logs
docker compose logs [servicio]

# Reiniciar un servicio específico
docker compose restart [servicio]

# Ver estado de servicios
docker compose ps
```

### 7.2 Backup y Restore

El repositorio incluye scripts de backup y restore ubicados en `./scripts/`:
- `backup.sh`: Backup de datos de Cassandra y Elasticsearch
- `restore.sh`: Restauración de backups
- `reset.sh`: Reset completo del entorno

### 7.3 Monitoreo

Se pueden consultar los health checks de los servicios:
```powershell
docker compose ps
```

Todos los servicios incluyen health checks automáticos que verifican su estado cada pocos segundos.

---

## 8. Seguridad

### 8.1 Medidas Implementadas

1. **Autenticación en Elasticsearch:** Usuario/contraseña configurados
2. **Autenticación en Cassandra:** Configuración de credenciales
3. **SSL/TLS:** Certificados configurados para nginx
4. **Claves secretas:** Generadas para TheHive y Cortex
5. **Red aislada:** Contenedores en red dedicada

### 8.2 Recomendaciones de Seguridad

Para entornos de producción:

1. **Cambiar todas las contraseñas por defecto**
2. **Utilizar certificados SSL válidos**
3. **Configurar firewall para limitar acceso**
4. **Implementar backup automatizado**
5. **Configurar logging centralizado**
6. **Actualizar regularmente las imágenes Docker**

---

## 9. Integración Futura con MISP y Wazuh

### 9.1 Preparación para MISP

El entorno está preparado para integrar MISP mediante:
- Configuración de conectores en TheHive
- API REST disponible para intercambio de IOCs
- Elasticsearch común para búsquedas correlacionadas

### 9.2 Preparación para Wazuh

La integración con Wazuh se realizaría mediante:
- APIs de TheHive para crear casos automáticamente
- Forwarding de logs de Wazuh a Elasticsearch
- Configuración de reglas para escalado automático

---

## 10. Troubleshooting

### 10.1 Problemas Comunes

#### Error 404 "NotFound" en localhost:9000
**Síntoma:** Acceder a http://localhost:9000 devuelve `{"type": "NotFound", "message": ""}`
**Causa:** TheHive requiere la ruta completa `/thehive` en la URL
**Solución:** Usar la URL completa http://localhost:9000/thehive

#### Error de certificados en nginx
**Síntoma:** nginx no inicia por falta de certificados
**Solución:** Generar certificados autofirmados como se documentó

#### Permisos de Elasticsearch
**Síntoma:** Elasticsearch no puede escribir en directorios
**Solución:** Configurar permisos con icacls en Windows

#### Conectividad entre contenedores
**Síntoma:** Servicios no pueden comunicarse
**Solución:** Verificar que todos estén en la misma red Docker

### 10.2 Logs de Diagnóstico

```powershell
# Ver logs de todos los servicios
docker compose logs

# Ver logs de un servicio específico
docker compose logs elasticsearch
docker compose logs thehive
docker compose logs cortex
```

---

## 11. Conclusiones

### 11.1 Resultados Obtenidos

✅ **Implementación exitosa** del stack TheHive + Cortex + Elasticsearch + Cassandra  
✅ **Todos los servicios operativos** y respondiendo correctamente  
✅ **Configuración de seguridad básica** implementada  
✅ **Acceso web funcional** a través de HTTP y HTTPS  
✅ **Arquitectura escalable** preparada para producción  

### 11.2 Lecciones Aprendidas

1. **Adaptación a Windows:** Los scripts bash originales requirieron adaptación para PowerShell
2. **Certificados SSL:** Necesarios para nginx, generación automatizada recomendada
3. **Permisos de archivos:** Críticos para el correcto funcionamiento de Elasticsearch
4. **Healthchecks:** Fundamentales para verificar el estado de los servicios

### 11.3 Próximos Pasos

1. **Configuración de usuarios y organizaciones** en TheHive y Cortex
2. **Instalación de analizadores** en Cortex para análisis de observables
3. **Configuración de conectores** para integración con fuentes externas
4. **Implementación de MISP** como fuente de threat intelligence
5. **Integración de Wazuh** para detección y respuesta automatizada
6. **Migración a entorno de producción** con alta disponibilidad

### 11.4 Recomendaciones Finales

- Utilizar este entorno de testing para familiarizarse con las herramientas
- Planificar la arquitectura de producción considerando alta disponibilidad
- Establecer procedimientos de backup y recovery
- Configurar monitoreo y alertas para entornos críticos
- Mantener actualizada la documentación de configuración

---

## 12. Referencias

- [Documentación oficial de TheHive](https://docs.thehive-project.org/)
- [Documentación oficial de Cortex](https://docs.thehive-project.org/cortex/)
- [Repositorio Docker StrangeBeeCorp](https://github.com/StrangeBeeCorp/docker)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Elasticsearch Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/)

---

**Fin del Informe**

*Documento generado automáticamente durante el proceso de implementación*  
*Fecha de generación: 17 de septiembre de 2025*