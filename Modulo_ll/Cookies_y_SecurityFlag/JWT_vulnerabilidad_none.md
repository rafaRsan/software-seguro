# Análisis Técnico: Vulnerabilidad de Algoritmo `none` en JWT

## 1. Contexto y Origen en el Estándar (RFC 7519)

Un **JSON Web Token (JWT)** está diseñado para transmitir información verificable entre dos partes mediante tres segmentos codificados en **Base64Url** separados por puntos:

```text
Header.Payload.Signature
```

La seguridad de este mecanismo no reside en ocultar los datos (ya que el `Header` y el `Payload` viajan en texto plano codificado), sino en la **integridad criptográfica** garantizada por el tercer segmento (`Signature`). Cuando el servidor emite el token, calcula una firma aplicando un algoritmo criptográfico (como `HS256` o `RS256`) sobre el `Header` y el `Payload` utilizando una clave secreta o privada que solo el backend conoce.

Sin embargo, la especificación oficial de JWT (**RFC 7519** / **JWS RFC 7515**) contempla un valor válido para el campo `"alg"` denominado **`"none"`**, definido para representar *Unsecured JWTs* (tokens cuya integridad ya fue garantizada por otro canal seguro, como mTLS, o en entornos internos de depuración). Un JWT con `alg: "none"` **no lleva firma criptográfica**.

---

## 2. Mecánica Técnica de la Vulnerabilidad

El fallo ocurre cuando la librería de validación en el backend confía ciegamente en el parámetro `"alg"` declarado dentro del propio **Header** que envía el cliente, en lugar de forzar una lista blanca (*allowlist*) de algoritmos permitidos en el servidor.

### ¿Por qué el atacante no necesita la clave secreta del servidor?

1. **Inversión del control de validación:**
   En una implementación vulnerable, el flujo de verificación del backend lee primero el `Header` del token entrante para decidir qué función criptográfica ejecutar. Si el atacante modifica el campo `"alg"` a `"none"` (o variantes de capitalización como `"None"`, `"NONE"`, `"nOnE"` si el parser es *case-insensitive*), la librería interpreta que se trata de un token sin firma y **omite por completo la rutina de verificación criptográfica**.

2. **Ausencia de cálculo de firma:**
   Como el algoritmo `"none"` no utiliza ninguna función HMAC ni asimétrica (RSA/ECDSA), el backend nunca llega a invocar su clave secreta (`SECRET_KEY`) ni su clave pública para validar el token. Simplemente comprueba que el formato estructural tenga los dos puntos separadores (`Header.Payload.`) con el tercer bloque vacío.

3. **Manipulación arbitraria del Payload:**
   Al no existir firma que proteja la integridad de los datos, el atacante puede decodificar el `Payload` original, alterar cualquier *claim* (como su `id` de usuario, `role_id` o cambiar `"is_superadmin": false` a `true`), volver a codificarlo en Base64Url y enviarlo al servidor. El backend procesará los nuevos atributos como totalmente legítimos.

---

## 3. Anatomía de la Falsificación (Paso a Paso)

Supongamos que un atacante intercepta su propio token legítimo firmado con `HS256`:

### Paso 1: Token Original Legítimo
* **Header:** `{"alg": "HS256", "typ": "JWT"}`
* **Payload:** `{"id": 1084, "role": "user", "is_superadmin": false}`
* **Signature:** `<Firma_HMAC_Valida_Generada_Con_Secreto_Del_Servidor>`

### Paso 2: Manipulación del Header y Payload
El atacante modifica ambos JSON localmente (sin conocer la clave secreta del servidor):

* **Header modificado:**
  ```json
  {
    "alg": "none",
    "typ": "JWT"
  }
  ```
  *Codificado en Base64Url:* `eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0`

* **Payload modificado (Escalada de privilegios):**
  ```json
  {
    "id": 1,
    "role": "admin",
    "is_superadmin": true
  }
  ```
  *Codificado en Base64Url:* `eyJpZCI6MSwicm9sZSI6ImFkbWluIiwiaXNfc3VwZXJhZG1pbiI6dHJ1ZX0`

### Paso 3: Ensamblaje del Token Falsificado
El atacante concatena ambos bloques codificados y **elimina la firma**, dejando únicamente el punto final obligatorio de la sintaxis JWT:

```text
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJpZCI6MSwicm9sZSI6ImFkbWluIiwiaXNfc3VwZXJhZG1pbiI6dHJ1ZX0.
```

### Paso 4: Ejecución en el Backend Vulnerable
Al enviar este token en la cabecera `Authorization: Bearer <token_falsificado>`, el servidor decodifica el header, lee `"alg": "none"`, salta la validación de firma y autentica al atacante como el usuario administrador (`id: 1`, `is_superadmin: true`).

---

## 4. Impacto y Remediación

### Impacto de Seguridad
* **Suplantación total de identidad (*Account Takeover*):** El atacante puede hacerse pasar por cualquier usuario registrado conociendo únicamente su identificador o correo.
* **Escalada de privilegios vertical y horizontal:** Permite atribuirse roles administrativos arbitrarios sin necesidad de realizar ataques de fuerza bruta sobre la clave secreta ni explotar inyecciones.

### Medidas de Mitigación
1. **Deshabilitar explícitamente `alg: "none"`:** Asegurar que la librería JWT utilizada esté actualizada y rechace por defecto cualquier token sin firma.
2. **Validación estricta de algoritmo en el servidor (*Allowlisting*):** Configurar el verificador del backend para que acepte **únicamente** el algoritmo esperado por la arquitectura (por ejemplo, `algorithms=["HS256"]` o `["RS256"]`), ignorando lo que declare el cliente en el `Header`.
3. **Verificación de presencia de firma:** Rechazar automáticamente cualquier token cuya tercera sección (`Signature`) tenga longitud cero.
