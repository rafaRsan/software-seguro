# Objetivo 3: Peligros del "Directory Listing" (Indexación de Directorios)

El **Directory Listing** (o listado de directorios) es una funcionalidad presente en la mayoría de los servidores web (como Apache, Nginx o IIS) que, por defecto o por mala configuración del administrador, muestra una lista del contenido de un directorio cuando no existe un archivo de índice primario.

### ¿Qué ocurre técnicamente cuando un atacante navega a esta ruta?

En el flujo normal, cuando un usuario solicita un directorio (por ejemplo, `https://equisde.com/uploads/`), el servidor web busca automáticamente archivos predeterminados configurados en su directiva de índice, tales como `index.html`, `index.php`, `index.htm` o `default.aspx`; si ninguno de estos archivos existe dentro del directorio solicitado y la directiva de *Directory Listing* está **activa**, el servidor no devuelve un error de restricción de acceso (`403 Forbidden`). En su lugar, el motor del servidor genera automaticamente un HTML (generalmente con el título `Index of /nombre-directorio`). La página generada renderiza en el navegador del atacante una estructura en árbol con todos los archivos y subcarpetas contenidos en esa ruta del sistema de archivos, incluyendo detalles como el tamaño, la extensión y la fecha de última modificación.
El atacante puede navegar directamente haciendo clic en los archivos exposed o utilizando herramientas automatizadas de *fuzzing* (como `ffuf`, `feroxbuster` o *Google Dorks* como `intitle:"Index of"`) para enumerar y descargar masivamente todo el contenido sin necesidad de adivinar nombres de archivos.

---

## Tipos de Archivos Sensibles Expuestos

Cuando el Directory Listing queda expuesto, un atacante puede acceder libremente a archivos que comprometen la confidencialidad e integridad de la aplicación:

* **Archivos de Configuración y Entorno (`.env`, `config.php`, `web.config`, `settings.json`):** Suelen contener credenciales de bases de datos en texto plano, claves API, llaves secretas para firma de tokens JWT y parámetros de conexión a servicios de terceros.
* **Repositorios de Control de Versiones (`.git/` o `.svn/`):** Si la carpeta oculta `.git` queda expuesta en la raíz pública, un atacante puede utilizar herramientas como `git-dumper` para descargar todos los objetos del repositorio y reconstruir el código fuente completo del proyecto localmente.
* **Copias de Seguridad y Dumps (`backup.sql`, `db_dump.tar.gz`, `copia_sitio.zip`, `.bak`):** Copias antiguas o automatizadas que contienen la base de datos completa de la aplicación, incluyendo hashes de contraseñas y datos personales de usuarios (PII).
* **Archivos subidos por Usuarios (`/uploads/`):** Documentos privados como facturas, documentos de identidad, planillas con datos sensibles (`.xlsx`) o imágenes con metadatos EXIF no sanitizados.
* **Archivos de Registro (`.log`):** Logs de errores internas del sistema (*stack traces*) que revelan rutas absolutas del servidor, versiones de lenguajes y librerías, o incluso tokens de sesión de usuarios capturados en los logs.

---

## Mitigación y Buenas Prácticas

La remediación de este error de configuración debe abordarse en múltiples capas:

### A. Deshabilitar el Directory Listing a Nivel de Servidor Web (Solución Principal)

* **En servidores Apache:**
  Se debe desactivar la opción `Indexes` en el archivo de configuración global (`httpd.conf` / `apache2.conf`) o mediante el archivo `.htaccess` situado en la raíz del sitio web:
  ```apache
  Options -Indexes
  ```

* **En servidores Nginx:**
  Por defecto, Nginx mantiene desactivada esta función. Si fue activada manualmente, se debe asegurar que la directiva `autoindex` dentro del bloque `server` o `location` del archivo `nginx.conf` esté configurada en `off`:
  ```nginx
  autoindex off;
  ```

### B. Creación de Archivos de Índice Opcionales
Colocar un archivo `index.html` o `index.php` en blanco en todas las carpetas públicas. De esta forma, si una solicitud llega al directorio, el servidor responderá cargando la página vacía en lugar de listar los archivos.

### C. Almacenamiento Seguro Fuera de la Raíz Pública (`webroot`)
Los archivos sensibles (como `.env`, respaldos SQL o código fuente backend) **nunca** deben almacenarse en la carpeta accesible por la web (`public_html` o `www`). Deben residir en directorios superiores fuera del alcance del servidor web y utilizar administradores de secretos dedicados.
