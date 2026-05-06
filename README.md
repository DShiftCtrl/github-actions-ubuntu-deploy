# github-actions-ubuntu-deploy

## Descripción

Este proyecto corresponde al deber académico **"Automatización de Despliegue con GitHub Actions en un Servidor Ubuntu"**.

El objetivo es crear una aplicación básica con Node.js y Express, y configurar un flujo CI/CD con GitHub Actions para que, al hacer `push` a la rama `main`, GitHub se conecte por SSH a un servidor Ubuntu, actualice el código con `git pull`, instale dependencias y reinicie la aplicación usando PM2.

## Tecnologías utilizadas

- Node.js
- Express
- GitHub Actions
- Ubuntu Server
- SSH
- PM2

## Estructura del proyecto

```text
github-actions-ubuntu-deploy/
├── app.js
├── package.json
├── package-lock.json
├── README.md
├── .gitignore
└── .github/
    └── workflows/
        └── deploy.yml
```

## Explicación de la aplicación

La aplicación está desarrollada con Express y escucha en el puerto definido por la variable de entorno `PORT`. Si no existe esa variable, usa el puerto `3000`.

Rutas disponibles:

- `GET /`: muestra un mensaje HTML simple indicando que la aplicación fue desplegada automáticamente con GitHub Actions en un servidor Ubuntu.
- `GET /health`: devuelve un JSON para comprobar que el servidor está funcionando correctamente.

Respuesta esperada de `/health`:

```json
{
  "status": "ok",
  "message": "Servidor funcionando correctamente"
}
```

## Probar localmente

Instalar dependencias:

```bash
npm install
```

Iniciar la aplicación:

```bash
npm start
```

Probar el endpoint de salud:

```bash
curl http://localhost:3000/health
```

También se puede abrir en el navegador:

```text
http://localhost:3000
```

## Configuración del servidor Ubuntu

### 1. Actualizar paquetes

```bash
sudo apt update
sudo apt upgrade -y
```

### 2. Instalar Git

```bash
sudo apt install git -y
git --version
```

### 3. Instalar Node.js y npm

Una opción recomendada es instalar Node.js desde NodeSource:

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install nodejs -y
node -v
npm -v
```

### 4. Instalar PM2

```bash
sudo npm install -g pm2
pm2 -v
```

### 5. Clonar el repositorio

Reemplazar `<URL_DEL_REPOSITORIO>` por la URL real del repositorio en GitHub:

```bash
cd /var/www
sudo git clone <URL_DEL_REPOSITORIO> github-actions-ubuntu-deploy
cd github-actions-ubuntu-deploy
```

Si el usuario del servidor no tiene permisos sobre la carpeta, se puede asignar propiedad:

```bash
sudo chown -R $USER:$USER /var/www/github-actions-ubuntu-deploy
```

### 6. Instalar dependencias

```bash
npm install --production
```

### 7. Iniciar la app con PM2

```bash
pm2 start app.js --name github-actions-ubuntu-deploy
pm2 save
pm2 status
```

Para configurar PM2 al reiniciar el servidor:

```bash
pm2 startup
```

Luego ejecutar el comando que PM2 muestre en pantalla.

## Generación de llave SSH para GitHub Actions

En la máquina local o en una terminal segura, generar una llave SSH exclusiva para el despliegue:

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy"
```

Se recomienda guardar la llave con un nombre identificable, por ejemplo:

```text
github-actions-deploy
```

Esto generará dos archivos:

- `github-actions-deploy`: llave privada.
- `github-actions-deploy.pub`: llave pública.

### Copiar la llave pública al servidor

En el servidor Ubuntu, agregar el contenido de la llave pública al archivo `~/.ssh/authorized_keys` del usuario que hará el despliegue:

```bash
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Pegar dentro de `authorized_keys` el contenido de:

```bash
cat github-actions-deploy.pub
```

### Guardar la llave privada como secret en GitHub

El contenido completo de la llave privada debe guardarse como secret del repositorio con el nombre:

```text
SSH_PRIVATE_KEY
```

No se debe subir la llave privada al repositorio.

## Configuración de secrets en GitHub

En el repositorio de GitHub, entrar a:

```text
Settings > Secrets and variables > Actions > New repository secret
```

Crear los siguientes secrets:

| Secret | Descripción | Ejemplo de placeholder |
| --- | --- | --- |
| `SSH_HOST` | Dirección o dominio del servidor Ubuntu | `<TU_HOST_O_DOMINIO>` |
| `SSH_USER` | Usuario SSH del servidor | `<TU_USUARIO_SSH>` |
| `SSH_PRIVATE_KEY` | Llave privada SSH para despliegue | `<CONTENIDO_DE_LA_LLAVE_PRIVADA>` |
| `SSH_PORT` | Puerto SSH del servidor | `22` o `<TU_PUERTO_SSH>` |
| `PROJECT_PATH` | Ruta del proyecto en el servidor | `/var/www/github-actions-ubuntu-deploy` |

Importante: no colocar valores reales en archivos del repositorio. Los datos sensibles deben estar únicamente en GitHub Secrets.

## Funcionamiento del workflow deploy.yml

El archivo `.github/workflows/deploy.yml` define el flujo de despliegue automático.

El workflow:

1. Se llama `Deploy to Ubuntu Server`.
2. Se ejecuta cuando hay un `push` en la rama `main`.
3. Usa un runner `ubuntu-latest`.
4. Utiliza la acción `appleboy/ssh-action@v1.0.3` para conectarse al servidor por SSH.
5. Lee los datos de conexión desde GitHub Secrets.
6. Ejecuta estos comandos en el servidor:

```bash
cd ${{ secrets.PROJECT_PATH }}
git pull origin main
npm install --production
pm2 restart github-actions-ubuntu-deploy || pm2 start app.js --name github-actions-ubuntu-deploy
pm2 save
```

Con esto, cada cambio enviado a `main` actualiza el código en el servidor y reinicia la aplicación.

## Evidencia esperada

Para documentar el deber, se recomienda incluir:

- Captura del repositorio en GitHub con la estructura del proyecto.
- Captura de los secrets configurados en GitHub sin mostrar sus valores.
- Captura del workflow ejecutado correctamente en la pestaña **Actions**.
- Captura del servidor respondiendo en navegador o con `curl`.

Ejemplo de prueba con `curl`:

```bash
curl http://<TU_HOST_O_DOMINIO>:3000/health
```

Respuesta esperada:

```json
{
  "status": "ok",
  "message": "Servidor funcionando correctamente"
}
```

## Errores comunes y soluciones

### Error: Permission denied publickey

Verificar que:

- La llave pública esté dentro de `~/.ssh/authorized_keys` en el servidor.
- La llave privada esté guardada correctamente en el secret `SSH_PRIVATE_KEY`.
- El usuario definido en `SSH_USER` sea el mismo usuario que tiene la llave pública autorizada.

### Error: PROJECT_PATH no existe

Verificar que el repositorio esté clonado en el servidor y que el secret `PROJECT_PATH` tenga la ruta correcta.

Ejemplo:

```bash
ls /var/www/github-actions-ubuntu-deploy
```

### Error: npm command not found

Instalar Node.js y npm en el servidor:

```bash
node -v
npm -v
```

Si no aparecen versiones, repetir la instalación de Node.js.

### Error: pm2 command not found

Instalar PM2 globalmente:

```bash
sudo npm install -g pm2
```

### La app no responde en el navegador

Revisar el estado de PM2:

```bash
pm2 status
pm2 logs github-actions-ubuntu-deploy
```

También verificar que el puerto esté habilitado en el firewall o reglas de seguridad del proveedor del servidor.

## Buenas prácticas de seguridad

- No subir archivos `.env` al repositorio.
- No exponer llaves privadas en commits, capturas o documentación pública.
- Usar GitHub Secrets para guardar credenciales y datos sensibles.
- Crear una llave SSH exclusiva para despliegue, no reutilizar llaves personales.
- Limitar los permisos del usuario SSH para que solo tenga acceso a lo necesario.
- Revisar periódicamente las llaves autorizadas en el servidor.

## Comandos útiles en el servidor

Ver procesos administrados por PM2:

```bash
pm2 status
```

Ver logs:

```bash
pm2 logs github-actions-ubuntu-deploy
```

Reiniciar manualmente:

```bash
pm2 restart github-actions-ubuntu-deploy
```

Detener la aplicación:

```bash
pm2 stop github-actions-ubuntu-deploy
```
