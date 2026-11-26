# SSRF y el Peligro en la Nube (AWS/GCP)

## 1. El Concepto: El Servidor como "Proxy" del Atacante

En una vulnerabilidad de **Server-Side Request Forgery (SSRF)**, una aplicación web recibe una URL o destino proporcionado por el usuario (por ejemplo, al importar una foto de perfil desde un enlace, generar una vista previa o configurar un *webhook*) y realiza una petición HTTP desde su propio backend **sin validar ni filtrar el destino**.

Cuando esto ocurre en una infraestructura Cloud (como Amazon Web Services o Google Cloud Platform), el impacto escala de un simple fallo de aplicación a un **compromiso total de la infraestructura**, ya que el servidor web reside dentro de una red interna privilegiada y tiene acceso directo a servicios administrativos que están completamente bloqueados desde internet pública.

---

## 2. ¿Qué ocurre al consultar la dirección `169.254.169.254`?

La dirección IP **`169.254.169.254`** pertenece al rango reservado de direcciones *Link-Local* (`169.254.0.0/16`) definido por el estándar RFC 3927. En entornos Cloud (AWS, GCP, Azure, DigitalOcean), esta IP no apunta a un servidor externo, sino al **Instance Metadata Service (IMDS)** (Servicio de Metadatos de la Instancia).

* **Funcionamiento:** Es una API interna no enrutable desde internet. Solo el propio servidor virtual (por ejemplo, una instancia EC2 de AWS) puede hacerle peticiones HTTP locales para consultarse a sí mismo sobre su configuración, red y permisos.
* **La explotación vía SSRF:** Cuando el atacante inyecta `http://169.254.169.254/latest/meta-data/` en el parámetro vulnerable, el servidor backend realiza la petición `GET` localmente. Como el servicio IMDS asume que cualquier petición que llega a esa interfaz proviene de un proceso legítimo dentro de la misma máquina virtual, **responde entregando toda la información interna sin pedir autenticación adicional** (en su versión v1).

*(Nota: En Google Cloud Platform ocurre exactamente lo mismo apuntando a `http://metadata.google.internal/computeMetadata/v1/` o a la misma IP `169.254.169.254`, aunque GCP exige por defecto la cabecera HTTP `Metadata-Flavor: Google`, lo cual requiere un SSRF que permita inyectar cabeceras).*

---

## 3. ¿Qué información altamente sensible espera extraer el atacante?

El objetivo principal de apuntar al servicio de metadatos no es ver la IP privada del servidor, sino **escalar privilegios hacia la cuenta de la nube**. La ruta crítica en AWS es:

```http
GET /latest/meta-data/iam/security-credentials/<NOMBRE-DEL-ROL-IAM> HTTP/1.1
Host: 169.254.169.254
```

Al consultar este endpoint, el atacante espera extraer:

1. **Credenciales Temporales de AWS IAM (El premio mayor):**
   Si la instancia EC2 tiene asignado un *IAM Role* (algo muy común para que la app pueda leer buckets S3, bases de datos DynamoDB o enviar correos con SES), el servidor devuelve un objeto JSON con:
   * `AccessKeyId`
   * `SecretAccessKey`
   * `Token` (Session Token)
   * *Impacto:* El atacante copia estas tres claves en su propia computadora (`aws configure`), asume la identidad del servidor y obtiene acceso directo a los servicios de AWS (descargar bases de datos completas, leer archivos privados en S3, crear usuarios administradores o destruir servidores) **desde fuera de la red de la víctima**.

2. **User Data (`http://169.254.169.254/latest/user-data`):**
   Contiene los scripts de inicialización que se ejecutaron cuando arrancó la máquina virtual por primera vez. Es habitual que los desarrolladores cometan el error de hardcodear allí contraseñas de bases de datos, claves SSH, tokens de GitHub o variables de entorno sensibles.

3. **Información de Red y Topología Interna:**
   * Direcciones IP privadas, subredes (VPC), IDs de grupos de seguridad (*Security Groups*) y zonas de disponibilidad.
   * Esto permite al atacante mapear la red interna para realizar **pivoting** hacia otros microservicios, paneles administrativos internos o bases de datos sin exposición pública.

4. **Claves Públicas SSH y Certificados:**
   A través de `/latest/meta-data/public-keys/`, el atacante puede identificar qué llaves SSH tienen acceso administrativo a la instancia.

---

## 4. Mitigación y Defensa en Profundidad

Para neutralizar este vector en entornos Cloud, se deben aplicar controles tanto a nivel de código como de infraestructura:

1. **Migración obligatoria a IMDSv2 (Nivel Infraestructura AWS):**
   Amazon introdujo **IMDSv2**, que reemplaza el modelo simple de peticiones `GET` por un esquema basado en sesiones. Exige realizar primero una petición `PUT` con una cabecera específica (`X-aws-ec2-metadata-token-ttl-seconds`) para obtener un token efímero, y luego enviar ese token en cada `GET`. Como la mayoría de los SSRF solo permiten controlar la URL de una petición `GET` sin modificar métodos ni cabeceras, IMDSv2 bloquea el ataque de raíz.
2. **Validación estricta por Lista Blanca (*Allowlist*):**
   Nunca utilizar listas negras (bloquear `127.0.0.1` o `169.254.169.254` es fácil de evadir usando formatos alternativos de IP como decimal `http://2852039166/`, octal, hexadecimal, IPv6 o redirecciones DNS). Se debe validar que el dominio o IP destino coincida estrictamente con una lista blanca de proveedores autorizados.
3. **Deshabilitar redirecciones HTTP en el cliente del servidor:**
   Configurar las librerías HTTP del backend (`curl`, `axios`, `requests`, `fetch`) para que no sigan redirecciones `301`/`302` automáticamente, evitando que un dominio permitido redirija internamente hacia `169.254.169.254`.
4. **Principio de Mínimo Privilegio (IAM):**
   Otorgar al rol IAM de la instancia únicamente los permisos estrictamente necesarios para operar, limitando el radio de explosión en caso de compromiso.