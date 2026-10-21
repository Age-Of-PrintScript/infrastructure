# Infraestructura - Docker Swarm

Repositorio de infraestructura y orquestación con **Docker Swarm** para los servicios del sistema de snippets.

---

## Descripción General y Servicios

El stack está orquestado mediante el archivo [`docker-stack.yml`](file:///C:/Users/lubet/projects/ingsis/infrastructure/docker-stack.yml) y actualmente contiene los siguientes servicios:

| Servicio | Imagen | Puerto Host:Contenedor | Descripción |
| :--- | :--- | :--- | :--- |
| **`snippet-service`** | `ghcr.io/age-of-printscript/snippet-service:${ENV}` | `8080:8080` | Servicio principal de gestión de snippets. Conecta a la base de datos PostgreSQL. |
| **`snippet-db`** | `postgres:16-alpine` | Interno (`5432`) | Base de datos PostgreSQL para el almacenamiento de snippets. |
| **`azurite`** | `mcr.microsoft.com/azure-storage/azurite` | `10000:10000`<br>`10001:10001`<br>`10002:10002` | Emulador local de Azure Storage (Blob, Queue y Table storage). |
| **`asset-service`** | `ghcr.io/austral-ingsis/snippet-asset-service:main.14` | `8083:8080` | Microservicio de gestión de assets que almacena archivos en Azurite. |

Todos los servicios se comunican a través de la red overlay interna **`app-network`**.

---

## Requisitos Previos

- **Docker**: versión 20.10 o superior con Docker Compose v2.
- **Docker Desktop** (en Windows/macOS) o **Docker Engine** (en Linux) en ejecución.
- Acceso y permisos para descargar imágenes en caso de que sean privadas (por ejemplo, desde GitHub Container Registry `ghcr.io`).

---

## Configuración Inicial de Variables de Entorno

El archivo [`docker-stack.yml`](file:///C:/Users/lubet/projects/ingsis/infrastructure/docker-stack.yml) requiere ciertas variables de entorno (`${ENV}`, `${POSTGRES_PASSWORD}`, etc.).

1. Copiar el archivo de plantilla [`.env.example`](file:///C:/Users/lubet/projects/ingsis/infrastructure/.env.example) para crear tu archivo `.env`

2. Completar los valores requeridos en `.env`

---

## Cómo Levantar el Docker Swarm y Desplegar el Stack

### Paso 1: Inicializar Docker Swarm

Verifica si Docker Swarm ya está activo en tu máquina:

```powershell
docker info --format '{{.Swarm.LocalNodeState}}'
```

- Si el resultado es `inactive`, inicializa el Swarm:
  ```powershell
  docker swarm init
  ```
  *(En entornos con múltiples tarjetas de red o adaptadores virtuales, puedes especificar tu IP local: `docker swarm init --advertise-addr <TU_IP_LOCAL>`)*.


- Si el resultado es `active`, puedes continuar al siguiente paso.



### Paso 2: Autenticación en Container Registry (Opcional, si aplica)

Si alguna de las imágenes se encuentra en un registro privado (como `ghcr.io` de GitHub):

```powershell
docker login ghcr.io -u <TU_USUARIO_GITHUB>
```
*(Ingresa tu Personal Access Token - PAT con permisos de `read:packages` cuando te solicite la contraseña).*


### Paso 3: Cargar variables y desplegar el Stack

> [!IMPORTANT]
> `docker stack deploy` **no lee automáticamente** el archivo `.env` para interpolar variables en el YAML (por ejemplo `${ENV}` o `${POSTGRES_PASSWORD}`). Por ello, debes exportar las variables a tu terminal antes de ejecutar el deploy.

Define un nombre para el stack (por ejemplo, `ingsis` o `snippet-stack`):

#### Opción A: En Windows (PowerShell)
```powershell
# 1. Cargar las variables del .env en la sesión actual de PowerShell
Get-Content .env | ForEach-Object {
    $line = $_.Trim()
    if ($line -and -not $line.StartsWith("#")) {
        $key, $value = $line -split "=", 2
        [System.Environment]::SetEnvironmentVariable($key.Trim(), $value.Trim())
    }
}

# 2. Desplegar el stack (usar --with-registry-auth si hay imágenes privadas)
docker stack deploy -c docker-stack.yml ingsis --with-registry-auth
```

#### Opción B: En Linux / macOS (Bash / Zsh)
```bash
# 1. Exportar variables del .env y desplegar
export $(grep -v '^#' .env | xargs)
docker stack deploy -c printscript-stack.yml ingsis --with-registry-auth
```


### Paso 4: Verificar el estado del despliegue

1. **Listar servicios y réplicas en ejecución:**
   ```powershell
   docker stack services ingsis
   ```
   Deberías observar las réplicas en estado `1/1` para cada servicio.

2. **Listar tareas individuales y estado de los contenedores:**
   ```powershell
   docker stack ps ingsis
   ```

3. **Ver logs de un servicio específico:**
   ```powershell
   docker service logs -f ingsis_snippet-service
   docker service logs -f ingsis_asset-service
   ```



### Paso 5: Detener o remover el stack

- **Para remover todos los servicios del stack:**
  ```powershell
  docker stack rm ingsis
  ```
  *(Los volúmenes persistentes se mantienen a salvo y no se borran al eliminar el stack).*


- **Para desactivar Docker Swarm en el nodo local (opcional):**
  ```powershell
  docker swarm leave --force
  ```

---

## Cómo Agregar un Nuevo Servicio

Sigue esta guía cada vez que necesites incorporar un nuevo microservicio, worker, base de datos o herramienta auxiliar al stack.

### Paso 1: Definir la imagen y puertos

1. **Imagen:** Asegúrate de conocer la URL de la imagen y su tag (por ejemplo, `ghcr.io/austral-ingsis/mi-nuevo-servicio:1.0.0` o `nginx:alpine`).
2. **Puertos:** Verifica qué puertos del host ya están utilizados en [`docker-stack.yml`](file:///C:/Users/lubet/projects/ingsis/infrastructure/docker-stack.yml) para evitar colisiones:
   - `8080` (usado por `snippet-service`)
   - `8083` (usado por `asset-service`)
   - `10000`, `10001`, `10002` (usados por `azurite`)
   Elige un puerto libre en el host para el nuevo servicio (ej. `8084`, `8085`, etc.).



### Paso 2: Registrar el servicio en `docker-stack.yml`

Abre [`docker-stack.yml`](file:///C:/Users/lubet/projects/ingsis/infrastructure/docker-stack.yml) y agrega la definición bajo la sección `services:`.

Usa la siguiente estructura estándar:

```yaml
  nombre-del-servicio:
    image: repo/imagen:tag
    ports:
      - "PUERTO_HOST:PUERTO_CONTENEDOR"
    environment:
      - VARIABLE_1=valor
      - OTRA_VARIABLE=${VARIABLE_DESDE_ENV}
    env_file:
      - .env
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure  # on-failure o any
    networks:
      - app-network
```

> [!NOTE]
> **Comunicación entre servicios:** Cualquier servicio conectado a `app-network` puede comunicarse con otro usando el **nombre del servicio como hostname** y su **puerto interno de contenedor**. Por ejemplo:
> - Para conectar a PostgreSQL: `jdbc:postgresql://snippet-db:5432/snippet_db`
> - Para conectar a Azurite: `http://azurite:10000`



### Paso 3: Variables de entorno

Si el nuevo servicio necesita nuevas configuraciones o secretos:
1. Agrégalas al archivo de ejemplo [`.env.example`](file:///C:/Users/lubet/projects/ingsis/infrastructure/.env.example) con un valor descriptivo o placeholder.
2. Agrega los valores reales a tu archivo local `.env`.
3. Haz referencia a ellas en [`docker-stack.yml`](file:///C:/Users/lubet/projects/ingsis/infrastructure/docker-stack.yml) mediante `${NOMBRE_VARIABLE}` o a través de `env_file: - .env`.


### Paso 4: Volúmenes persistentes (si aplica)

Si el servicio genera datos que deben persistir al reiniciar (como una base de datos o almacenamiento en disco):
1. Monta el volumen en el servicio:
   ```yaml
       volumes:
         - nuevo-volumen-data:/ruta/dentro/del/contenedor
   ```
2. Declara el volumen en la sección global `volumes:` al final de [`docker-stack.yml`](file:///C:/Users/lubet/projects/ingsis/infrastructure/docker-stack.yml):
   ```yaml
   volumes:
     snippet-db-data:
     blob:
     nuevo-volumen-data:
   ```



### Paso 5: Aplicar y verificar los cambios

No es necesario tumbar todo el stack para agregar un servicio. Simplemente vuelve a ejecutar el deploy:

```powershell
docker stack deploy -c docker-stack.yml ingsis --with-registry-auth
```

Docker Swarm detectará los cambios de forma incremental y levantará únicamente el nuevo servicio sin interrumpir los que ya están funcionando.

Luego verifica su funcionamiento:
```powershell
docker stack services ingsis
docker service logs -f ingsis_nombre-del-servicio
```

---



## Comandos Frecuentes de Docker Swarm

| Acción | Comando |
| :--- | :--- |
| **Listar stacks activos** | `docker stack ls` |
| **Listar servicios de un stack** | `docker stack services <STACK_NAME>` |
| **Ver tareas y estado de réplicas** | `docker stack ps <STACK_NAME>` |
| **Ver logs en tiempo real** | `docker service logs -f <STACK_NAME>_<SERVICE_NAME>` |
| **Escalar réplicas de un servicio** | `docker service scale <STACK_NAME>_<SERVICE_NAME>=3` |
| **Inspeccionar configuración de un servicio** | `docker service inspect --pretty <STACK_NAME>_<SERVICE_NAME>` |
| **Ver nodos del cluster Swarm** | `docker node ls` |
| **Eliminar el stack** | `docker stack rm <STACK_NAME>` |

