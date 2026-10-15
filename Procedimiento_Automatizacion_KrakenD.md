# Procedimiento de Automatización — Registro de Endpoints en KrakenD

---

## Control de Versiones

| Fecha | Versión | Descripción | Autor |
|:---|:---|:---|:---|
| 15-oct-2026 | 1.0 | Versión inicial | simon, jhoel |

---

## Tabla de Contenido

1. [Alcance](#1-alcance)
2. [Pre-requisitos](#2-pre-requisitos)
3. [Variables](#3-variables)
4. [Procedimiento](#4-procedimiento)
   - 4.1 [Paso 1: Creación del archivo de endpoint por el desarrollador](#41-paso-1-creación-del-archivo-de-endpoint-por-el-desarrollador)
     - 4.1.1 [Estructura del archivo](#411-estructura-del-archivo)
     - 4.1.2 [Ejemplo de endpoint](#412-ejemplo-de-endpoint)
   - 4.2 [Paso 2: Push al repositorio del microservicio](#42-paso-2-push-al-repositorio-del-microservicio)
   - 4.3 [Paso 3: Ejecución del pipeline en Jenkins](#43-paso-3-ejecución-del-pipeline-en-jenkins)
     - 4.3.1 [Etapas existentes del pipeline](#431-etapas-existentes-del-pipeline)
     - 4.3.2 [Nueva etapa: Registro en KrakenD](#432-nueva-etapa-registro-en-krakend)
   - 4.4 [Paso 4: Validación del despliegue](#44-paso-4-validación-del-despliegue)
5. [Script de automatización (merge-endpoint.py)](#5-script-de-automatización-merge-endpointpy)
6. [Etapa en Jenkinsfile](#6-etapa-en-jenkinsfile)
7. [Rollback](#7-rollback)
   - 7.1 [Restaurar krakend.json](#71-restaurar-krakenjson)
   - 7.2 [Reiniciar contenedor KrakenD](#72-reiniciar-contenedor-krakend)

---

## 1. Alcance

Automatizar el registro de endpoints en el API Gateway KrakenD mediante un script integrado como nueva etapa en el pipeline de Jenkins. El script leerá el archivo JSON del endpoint enviado por el desarrollador al repositorio del microservicio, lo insertará en el archivo maestro `krakend.json` del servidor, validará la configuración resultante y reiniciará el contenedor de KrakenD. Esto elimina la intervención manual actual, reduce errores humanos y garantiza trazabilidad de cada cambio a través del historial de Git y los logs de Jenkins.

---

## 2. Pre-requisitos

| # | Requisito | Detalle |
|:---|:---|:---|
| 1 | Acceso a Jenkins | Validar acceso a la herramienta: [https://jenkins1.tigo.net.bo/](https://jenkins1.tigo.net.bo/) |
| 2 | Docker | Instalado en el servidor de despliegue, con el contenedor `krakend-gateway` corriendo la imagen oficial `krakend/krakend-ce`. |
| 3 | Python 3.6+ | Disponible en el agente de Jenkins para ejecutar el script de merge (`merge-endpoint.py`). |
| 4 | Archivo maestro `krakend.json` | Centralizado en una ruta accesible por el pipeline (ej: `/etc/krakend/krakend.json` o en un repositorio Git dedicado). |
| 5 | Credenciales Git | Configuradas en Jenkins (SSH key o token) con permisos de lectura/escritura sobre el repositorio de configuración de KrakenD. |
| 6 | Convención de nombre | Cada microservicio debe incluir en su repositorio un archivo llamado `krakend-endpoints.json` con la definición de su(s) endpoint(s) en el formato estándar de KrakenD. |
| 7 | Permisos Docker | El usuario de Jenkins debe poder ejecutar `docker restart krakend-gateway` en el servidor destino. |

---

## 3. Variables

| Variable | Valor |
|:---|:---|
| **ARCHIVO DEL DESARROLLADOR** | `krakend-endpoints.json` (en la raíz del repo del microservicio) |
| **ARCHIVO MAESTRO** | `krakend.json` (configuración central del API Gateway) |
| **CONTENEDOR** | `krakend-gateway` |
| **IMAGEN DOCKER** | `krakend/krakend-ce:latest` |
| **SCRIPT DE MERGE** | `scripts/merge-endpoint.py` |
| **NAMESPACE** | `qa`, `prod` |

---

## 4. Procedimiento

### 4.1 Paso 1: Creación del archivo de endpoint por el desarrollador

> [!IMPORTANT]
> El desarrollador es responsable de crear un archivo `krakend-endpoints.json` en la raíz de su repositorio del microservicio. Este archivo debe contener **únicamente** la definición del endpoint que su servicio expone, **NO** la configuración completa de KrakenD.

#### 4.1.1 Estructura del archivo

El archivo `krakend-endpoints.json` debe contener un objeto JSON con la siguiente estructura mínima:

| Campo | Obligatorio | Descripción |
|:---|:---|:---|
| `endpoint` | ✅ | Ruta pública expuesta por KrakenD (ej: `/api/v1/products`) |
| `method` | ✅ | Método HTTP (`GET`, `POST`, `PUT`, `DELETE`, `PATCH`) |
| `output_encoding` | ✅ | Formato de salida (`json`) |
| `extra_config` | Opcional | Configuración de seguridad (JWT), rate limiting, etc. |
| `backend` | ✅ | Array con la definición del servicio interno destino |
| `backend[].url_pattern` | ✅ | Ruta interna del microservicio |
| `backend[].host` | ✅ | Host y puerto del microservicio (ej: `http://product-service:5000`) |

#### 4.1.2 Ejemplo de endpoint

A continuación se muestra un ejemplo completo del archivo `krakend-endpoints.json` que el desarrollador debe incluir en su repositorio:

```json
{
  "endpoint": "/api/v1/products",
  "method": "GET",
  "output_encoding": "json",
  "cache_ttl": "60s",
  "extra_config": {
    "auth/validator": {
      "alg": "RS256",
      "jwk_url": "https://auth.tudominio.com/.well-known/jwks.json",
      "cache": true,
      "cache_duration": 900,
      "roles_key": "roles",
      "roles": ["admin", "manager", "user", "guest"]
    },
    "qos/ratelimit/router": {
      "max_rate": 2000,
      "client_max_rate": 100,
      "strategy": "ip"
    }
  },
  "backend": [
    {
      "url_pattern": "/products",
      "method": "GET",
      "host": ["http://product-service:5000"],
      "encoding": "json",
      "extra_config": {
        "qos/circuit-breaker": {
          "interval": 60,
          "timeout": 15,
          "max_errors": 10,
          "log_status_change": true
        },
        "qos/ratelimit/proxy": {
          "max_rate": 1000,
          "capacity": 1000
        }
      }
    }
  ]
}
```

---

### 4.2 Paso 2: Push al repositorio del microservicio

1. El desarrollador agrega el archivo `krakend-endpoints.json` a la raíz de su repositorio.
2. Realiza el commit y push hacia la rama correspondiente.

```bash
git add krakend-endpoints.json
git commit -m "feat: agregar definición de endpoint para KrakenD"
git push origin release/1.0.1
```

3. Este push dispara automáticamente el pipeline de Jenkins mediante webhook.

---

### 4.3 Paso 3: Ejecución del pipeline en Jenkins

#### 4.3.1 Etapas existentes del pipeline

El pipeline actual ejecuta las siguientes etapas de forma secuencial:

| Etapa | Descripción | Herramienta |
|:---|:---|:---|
| 1. Análisis de código | Revisión de calidad y vulnerabilidades | SonarQube |
| 2. Pruebas unitarias | Ejecución de tests automatizados | JUnit / pytest |
| 3. Linting | Análisis estático de sintaxis y estilo | ESLint / Flake8 |
| 4. Build Docker | Construcción de la imagen del contenedor | Docker |
| 5. Despliegue a QA | Deploy del microservicio al clúster | Kubernetes |

#### 4.3.2 Nueva etapa: Registro en KrakenD

> [!NOTE]
> Esta es la etapa que se agrega al pipeline para automatizar el proceso que actualmente se realiza de forma manual.

La nueva etapa se ejecuta **después del despliegue exitoso** del microservicio y realiza los siguientes sub-pasos de forma automática:

| Sub-paso | Acción | Detalle |
|:---|:---|:---|
| 6.1 | Leer endpoint del desarrollador | Lee el archivo `krakend-endpoints.json` del repositorio del microservicio |
| 6.2 | Abrir archivo maestro | Carga el `krakend.json` maestro del servidor de KrakenD |
| 6.3 | Verificar duplicados | Comprueba si el endpoint ya existe para actualizarlo en vez de duplicarlo |
| 6.4 | Insertar endpoint | Agrega el nuevo endpoint dentro del array `"endpoints"` del archivo maestro |
| 6.5 | Validar configuración | Ejecuta `krakend check` para verificar que el JSON resultante sea válido |
| 6.6 | Guardar archivo | Persiste el `krakend.json` actualizado |
| 6.7 | Reiniciar KrakenD | Ejecuta `docker restart krakend-gateway` para aplicar la nueva configuración |

**Flujo visual del pipeline completo:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    PIPELINE DE JENKINS                          │
├─────────────────────────────────────────────────────────────────┤
│  Etapa 1: SonarQube (análisis de código)                        │
│  Etapa 2: Pruebas Unitarias                                     │
│  Etapa 3: Linting (análisis estático)                            │
│  Etapa 4: Build Docker (construir imagen)                        │
│  Etapa 5: Despliegue del microservicio a QA                      │
│  ★ Etapa 6: Registro en KrakenD (NUEVA - AUTOMATIZADA)           │
│      6.1 → Leer krakend-endpoints.json del repo                  │
│      6.2 → Abrir krakend.json maestro                             │
│      6.3 → Verificar que no exista duplicado                      │
│      6.4 → Insertar endpoint en el array endpoints[]              │
│      6.5 → Validar con krakend check                              │
│      6.6 → Guardar krakend.json actualizado                       │
│      6.7 → Reiniciar contenedor krakend-gateway                   │
└─────────────────────────────────────────────────────────────────┘
```

---

### 4.4 Paso 4: Validación del despliegue

Una vez completada la ejecución del pipeline, se debe verificar:

1. Que **todas las etapas** del pipeline hayan finalizado exitosamente (en verde).
2. Que el contenedor `krakend-gateway` esté corriendo.
3. Que el nuevo endpoint responda correctamente.

**Verificación del contenedor:**

```bash
docker ps | grep krakend-gateway
```

| Estado esperado | Indicador |
|:---|:---|
| Contenedor activo | `Up X minutes` |
| Puerto expuesto | `0.0.0.0:8080->8080/tcp` |
| Sin reinicios inesperados | Restart count = 0 |

**Verificación del endpoint registrado:**

```bash
curl -s http://localhost:8080/api/v1/products | head -20
```

El endpoint debe responder con datos del microservicio o con `401 Unauthorized` si requiere token JWT (lo cual confirma que KrakenD lo tiene registrado y está validando la autenticación).

---

## 5. Script de automatización (merge-endpoint.py)

> [!NOTE]
> Este script es el componente central de la automatización. Lee el endpoint del desarrollador y lo inserta en el archivo maestro `krakend.json`.

```python
#!/usr/bin/env python3
"""
Script para insertar automáticamente un endpoint nuevo
en el archivo maestro krakend.json del API Gateway KrakenD.

Uso:
    python3 merge-endpoint.py <ruta_krakend_maestro> <ruta_endpoint_nuevo>

Ejemplo:
    python3 merge-endpoint.py /etc/krakend/krakend.json ./krakend-endpoints.json
"""
import json
import sys
import shutil
from datetime import datetime


def backup_master(master_path):
    """Crea una copia de seguridad del archivo maestro antes de modificarlo."""
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    backup_path = f"{master_path}.backup_{timestamp}"
    shutil.copy2(master_path, backup_path)
    print(f"📦 Backup creado: {backup_path}")
    return backup_path


def merge_endpoint(master_path, new_endpoint_path):
    """Lee el endpoint nuevo y lo inserta en el archivo maestro de KrakenD."""

    # 1. Crear backup del archivo maestro
    backup_master(master_path)

    # 2. Leer el archivo maestro de KrakenD
    with open(master_path, 'r', encoding='utf-8') as f:
        krakend_config = json.load(f)

    # 3. Leer el endpoint nuevo del desarrollador
    with open(new_endpoint_path, 'r', encoding='utf-8') as f:
        new_endpoint = json.load(f)

    # 4. Verificar que no exista ya (evitar duplicados)
    for i, existing in enumerate(krakend_config['endpoints']):
        if (existing['endpoint'] == new_endpoint['endpoint']
                and existing.get('method', 'GET') == new_endpoint.get('method', 'GET')):
            print(f"⚠️  Endpoint {new_endpoint['endpoint']} [{new_endpoint.get('method', 'GET')}] ya existe. Actualizando...")
            krakend_config['endpoints'][i] = new_endpoint
            break
    else:
        # 5. Si no existe, insertar el nuevo endpoint
        krakend_config['endpoints'].append(new_endpoint)
        print(f"✅ Endpoint {new_endpoint['endpoint']} [{new_endpoint.get('method', 'GET')}] agregado.")

    # 6. Guardar el archivo maestro actualizado
    with open(master_path, 'w', encoding='utf-8') as f:
        json.dump(krakend_config, f, indent=2, ensure_ascii=False)

    print(f"✅ Archivo {master_path} actualizado correctamente.")
    print(f"📊 Total de endpoints registrados: {len(krakend_config['endpoints'])}")


if __name__ == '__main__':
    if len(sys.argv) != 3:
        print("Uso: python3 merge-endpoint.py <krakend_maestro.json> <endpoint_nuevo.json>")
        sys.exit(1)

    master = sys.argv[1]
    new_ep = sys.argv[2]
    merge_endpoint(master, new_ep)
```

---

## 6. Etapa en Jenkinsfile

> [!IMPORTANT]
> El siguiente bloque debe agregarse al `Jenkinsfile` del microservicio, después de la etapa de despliegue a QA.

```groovy
stage('Registrar Endpoint en KrakenD') {
    steps {
        script {
            echo '★ Iniciando registro de endpoint en KrakenD...'

            // 1. Verificar que el archivo de endpoints existe
            sh '''
                if [ ! -f krakend-endpoints.json ]; then
                    echo "⚠️ No se encontró krakend-endpoints.json. Omitiendo registro en KrakenD."
                    exit 0
                fi
            '''

            // 2. Ejecutar el script que inserta el endpoint en el archivo maestro
            sh '''
                python3 scripts/merge-endpoint.py \
                    /etc/krakend/krakend.json \
                    ./krakend-endpoints.json
            '''

            // 3. Validar que el JSON resultante sea correcto
            sh 'krakend check -c /etc/krakend/krakend.json'

            // 4. Reiniciar KrakenD para aplicar la nueva configuración
            sh 'docker restart krakend-gateway'

            echo '✅ Endpoint registrado y KrakenD reiniciado exitosamente.'
        }
    }
}
```

---

## 7. Rollback

> [!CAUTION]
> En caso de que se presente algún inconveniente durante o después del registro del endpoint en KrakenD, se deberá ejecutar el procedimiento de rollback para restaurar la configuración anterior del API Gateway.

### 7.1 Restaurar krakend.json

El script de merge genera automáticamente un backup antes de cada modificación con el formato:
```
krakend.json.backup_20261015_143000
```

Para restaurar la configuración anterior:

1. Conectarse al servidor donde corre KrakenD.
2. Identificar el archivo de backup más reciente:
   ```bash
   ls -lt /etc/krakend/krakend.json.backup_* | head -1
   ```
3. Restaurar el backup:
   ```bash
   cp /etc/krakend/krakend.json.backup_YYYYMMDD_HHMMSS /etc/krakend/krakend.json
   ```
4. Validar que el archivo restaurado sea correcto:
   ```bash
   krakend check -c /etc/krakend/krakend.json
   ```

### 7.2 Reiniciar contenedor KrakenD

1. Reiniciar el contenedor para aplicar la configuración restaurada:
   ```bash
   docker restart krakend-gateway
   ```
2. Verificar que el contenedor esté corriendo:
   ```bash
   docker ps | grep krakend-gateway
   ```
3. Verificar los logs para confirmar que no hay errores:
   ```bash
   docker logs --tail 50 krakend-gateway
   ```

| Verificación post-rollback | Estado esperado |
|:---|:---|
| Contenedor en estado `Up` | ✅ |
| Sin errores en logs | ✅ |
| Endpoints anteriores respondiendo | ✅ |
| Endpoint problemático removido | ✅ |

---

> **Documento elaborado por:** José Gael Choque Serrano
> **Fecha de elaboración:** 15 de octubre de 2026
> **Versión:** 1.0
