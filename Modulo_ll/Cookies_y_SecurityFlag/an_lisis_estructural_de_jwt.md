# Auditoría Técnica de JSON Web Token (JWT)

## 1. Extracción y Decodificación del Token

Para esta actividad se interceptó un token de autenticación válido almacenado en el `localStorage` del navegador bajo la clave `auth`, correspondiente a una sesión autenticada de una aplicación web moderna.

Al inspeccionar su estructura, se verificó el formato estándar compuesto por tres bloques codificados en **Base64Url** separados por puntos (`Header.Payload.Signature`). Utilizando decodificación local, se extrajeron los objetos JSON de las dos primeras secciones.

### Header Decodificado
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### Payload Decodificado *(Datos personales sensibles ofuscados por OpSec)*
```json
{
  "id": 1084,
  "fullname": "[OFUSCADO]",
  "email": "[OFUSCADO]@gmail.com",
  "role_id": 1,
  "plan_id": 2,
  "pagado": true,
  "fecha_pago": "2026-09-03T20:34:25.172Z",
  "plan_expires_at": null,
  "email_confirmed": true,
  "pending_payment": null,
  "confirmation_email_uuid": "[UUID_OFUSCADO]",
  "created_at": "2026-09-20T23:56:27.277Z",
  "is_superadmin": false,
  "iat": 1795693315,
  "exp": 1795714915
}
```

---

## 2. Identificación de la Estructura Técnica

### ¿Qué algoritmo de firma se está utilizando en el Header?
En el encabezado (`Header`), el parámetro `"alg"` indica que el token está firmado utilizando **`HS256` (HMAC con SHA-256)**, mientras que `"typ"` confirma que el tipo de objeto es `"JWT"`.

* **Implicancia criptográfica:** `HS256` es un algoritmo de **firma simétrica**. Esto significa que el servidor utiliza una única clave secreta compartida tanto para generar la firma al momento del *login* como para verificar su integridad en las peticiones posteriores. Si dicha clave secreta es débil o predecible, un atacante que intercepte el token puede realizar un ataque de fuerza bruta offline (mediante herramientas como `hashcat` o `jwt_tool`) para descubrir el secreto y forjar tokens arbitrarios.

### ¿Qué datos sensibles o roles se están enviando en el Payload en texto plano?
Dado que los JWT estándar **no están cifrados** (únicamente codificados en Base64Url), cualquier actor que acceda al token puede leer la totalidad de su contenido. En el `Payload` analizado se identificaron los siguientes datos sensibles, identificadores internos y atributos de autorización:

1. **Roles y Privilegios de Acceso:**
   * `"role_id": 1` (identificador numérico del rol asignado al usuario).
   * `"is_superadmin": false` (bandera booleana explícita que define privilegios administrativos globales).
   * **Riesgo:** Expone exactamente cómo el backend estructura su modelo de control de acceso, facilitando la identificación de vectores para **escalada de privilegios vertical** (por ejemplo, probando ataques de *Mass Assignment* en endpoints de actualización de perfil o manipulando el token si la firma no se valida estrictamente).

2. **Identificadores Directos Predecibles:**
   * `"id": 1084` (clave primaria entera secuencial en base de datos).
   * **Riesgo:** Confirma que los registros de usuarios son secuenciales y enumerables, sirviendo como punto de partida para buscar vulnerabilidades **IDOR / BOLA** en los endpoints de la API.

3. **Información Personal Identificable (PII) y Estado Comercial:**
   * `"fullname"` y `"email"` (identidad real y correo electrónico del usuario).
   * `"plan_id": 2`, `"pagado": true`, `"fecha_pago"` (estado de suscripción y facturación).
   * **Riesgo:** Exposición innecesaria de datos personales (*Excessive Data Exposure*) en cada petición HTTP y en el almacenamiento del cliente (`localStorage`), los cuales podrían ser exfiltrados ante una vulnerabilidad XSS.

4. **Tokens Internos de Verificación:**
   * `"confirmation_email_uuid"` (identificador único utilizado para procesos internos de confirmación de correo).
   * **Riesgo:** Un token de validación de cuenta nunca debería viajar dentro del token de sesión general; si un endpoint de confirmación o recuperación no invalida este UUID tras su uso, su filtración compromete la integridad de la cuenta.

5. **Control Temporal de Sesión:**
   * `"iat": 1795693315` (timestamp de emisión) y `"exp": 1795714915` (timestamp de expiración).
   * **Análisis:** La ventana de validez (`exp - iat`) es de **21.600 segundos (6 horas)**. Al almacenarse en `localStorage` (accesible vía JavaScript por carecer de bandera `HttpOnly`), el robo del token permite el secuestro de sesión durante todo ese lapso sin posibilidad de revocación inmediata por cierre de ventana en el cliente.