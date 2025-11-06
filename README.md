# EMQX Certbot

Proyecto Docker Compose para configurar EMQX (broker MQTT) con certificados SSL/TLS generados automáticamente mediante Let's Encrypt y Certbot.

## Descripción

Este proyecto automatiza la obtención de certificados SSL/TLS de Let's Encrypt para un dominio específico y los configura en EMQX para habilitar conexiones MQTT seguras sobre TLS en el puerto 8883.

### Componentes

- **Certbot**: Obtiene y gestiona certificados SSL de Let's Encrypt
- **EMQX**: Broker MQTT con soporte TLS/SSL

## Requisitos Previos

- Docker y Docker Compose instalados
- Un dominio apuntando a la IP del servidor donde se ejecuta este proyecto
- Puerto 80 disponible (requerido por Let's Encrypt para la validación HTTP)

## Configuración

1. Copia el archivo de plantilla de variables de entorno:
   ```bash
   cp .env.template .env
   ```

2. Edita el archivo `.env` y configura las siguientes variables:
   - `DOMAIN`: Tu dominio (ej: `mqtt.tudominio.com`)
   - `EMAIL`: Tu email para notificaciones de Let's Encrypt
   - `EMQX_ADMIN_USER`: Usuario administrador del dashboard de EMQX
   - `EMQX_ADMIN_PASS`: Contraseña del administrador de EMQX
   - `EMQX_COOKIE`: Cookie del nodo EMQX (secreto para clustering)

3. Configura el archivo ACL (opcional):
   - Edita `emqx-acl.conf` para definir las reglas de control de acceso (ACL)
   - El archivo se monta automáticamente en el contenedor de EMQX
   - Puedes personalizar las reglas según tus necesidades de seguridad

## Ejecución

### Primera ejecución (obtener certificados)

```bash
docker compose up -d
```

El contenedor de Certbot obtendrá automáticamente los certificados SSL para tu dominio y los copiará a la carpeta `./emqx-certs` con los permisos correctos para EMQX.

### Verificar el estado

```bash
docker compose ps
```

### Ver logs

```bash
# Logs de Certbot
docker compose logs certbot

# Logs de EMQX
docker compose logs emqx
```

### Acceder al Dashboard de EMQX

Una vez que los contenedores estén corriendo, accede al dashboard en:
- URL: `http://localhost:18083`
- Usuario: El valor configurado en `EMQX_ADMIN_USER`
- Contraseña: El valor configurado en `EMQX_ADMIN_PASS`

## Estructura de Carpetas

```
.
├── compose.yaml          # Configuración de Docker Compose
├── .env                  # Variables de entorno (secretos)
├── .env.template         # Plantilla de variables de entorno
├── emqx-acl.conf         # Archivo ACL para control de acceso MQTT
├── certs/                # Certificados de Let's Encrypt (generados)
└── emqx-certs/           # Certificados copiados para EMQX (generados)
```

## Puertos

- **80**: HTTP (usado por Certbot para validación)
- **8883**: MQTT sobre TLS
- **18083**: Dashboard web de EMQX

## Renovación de Certificados

Los certificados de Let's Encrypt expiran cada 90 días. Para renovarlos automáticamente, puedes:

1. Ejecutar manualmente:
   ```bash
   docker compose run --rm certbot certbot renew
   ```

2. Configurar un cron job que ejecute el comando anterior periódicamente

3. Descomentar la línea en `compose.yaml` que mantiene Certbot corriendo en un loop para renovación automática

## Notas Importantes

- La primera vez que se ejecuta, Certbot obtiene los certificados. Si ya existen, no se emitirán nuevos.
- Los certificados se copian automáticamente a `./emqx-certs` con los permisos correctos.
- Asegúrate de que el dominio apunta correctamente a la IP del servidor antes de ejecutar.
- El puerto 80 debe estar disponible y accesible desde internet para la validación de Let's Encrypt.

## Detener los Servicios

```bash
docker compose down
```

Para detener y eliminar volúmenes:

```bash
docker compose down -v
```

## Control de Acceso (ACL)

El proyecto incluye un archivo ACL (`emqx-acl.conf`) para controlar qué clientes pueden publicar o suscribirse a qué temas. El archivo se monta automáticamente en el contenedor de EMQX.

### Formato del ACL

El archivo ACL usa sintaxis Erlang. Ejemplos:

```erlang
%% Permitir al usuario "admin" publicar y suscribirse a todos los temas
{allow, {user, "admin"}, pubsub, ["$SYS/#", "#"]}.

%% Permitir a todos los clientes suscribirse a temas públicos
{allow, all, subscribe, ["public/#"]}.

%% Denegar acceso a temas del sistema
{deny, all, subscribe, ["$SYS/#", "$queue/#"]}.
```

### Personalizar el ACL

1. Edita el archivo `emqx-acl.conf` con tus reglas
2. Reinicia el contenedor de EMQX:
   ```bash
   docker compose restart emqx
   ```

Las reglas se evalúan en orden, la primera coincidencia aplica. Para más información sobre ACL en EMQX, consulta la [documentación oficial de EMQX](https://www.emqx.io/docs).

## Solución de Problemas

### Certbot no puede obtener certificados

- Verifica que el dominio apunta a la IP del servidor
- Asegúrate de que el puerto 80 está abierto y accesible desde internet
- Revisa los logs: `docker compose logs certbot`

### EMQX no inicia

- Verifica que los certificados existen en `./emqx-certs`
- Revisa los logs: `docker compose logs emqx`
- Asegúrate de que las variables de entorno están correctamente configuradas
- Verifica que el archivo `emqx-acl.conf` tiene sintaxis válida (formato Erlang)
