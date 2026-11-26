# Auditoría de Banderas de Seguridad en Cookies

## 1. Análisis Técnico de Sitios Web Públicos

Para esta auditoría se inspeccionó el almacenamiento del navegador (*DevTools -> Almacenamiento -> Cookies*) sobre tres plataformas públicas de alto tráfico: **X (Twitter)** en estado autenticado, y **YouTube** junto a **Crunchyroll** en estado de navegación anónima (sin sesión iniciada).

---

### Sitio 1: X (Twitter) — `x.com` (Sesión Autenticada)

* **Nombre de la cookie:** `auth_token`
* **Tipo de cookie:** Cookie de sesión autenticada (identificador principal de la cuenta).
* **Valor:** `[OFUSCADO_POR_SEGURIDAD]` *(Longitud: 50 caracteres)*
* **Dominio / Ruta:** `.x.com` / `/`
* **Banderas detectadas:**
  * `HttpOnly`: ✅ Activada (`true`)
  * `Secure`: ✅ Activada (`true`)
  * `SameSite`: ⚠️ `"None"`
  * `HostOnly`: `false`
* **Diagnóstico:** 
  * **Protección XSS y Transporte:** Implementación sólida. Al tener `HttpOnly: true` y `Secure: true`, el identificador de sesión no puede ser exfiltrada mediante scripts (`document.cookie`) ante una vulnerabilidad XSS ni interceptado en texto plano sobre redes no cifradas.
  * **Observación crítica sobre `SameSite` y alcance:** La cookie tiene `SameSite="None"` y `Domain=".x.com"` (`HostOnly: false`), lo que permite que se envíe en peticiones de origen cruzado (*cross-site*) y hacia cualquier subdominio. Para evitar ataques de **Cross-Site Request Forgery (CSRF)** bajo esta configuración, la plataforma delega la protección en una segunda cookie complementaria (`ct0`, con `HttpOnly: false` y `SameSite="Lax"`), obligando al frontend a leerla y enviarla como cabecera personalizada (`x-csrf-token`) en cada petición que modifica estado.

---

### Sitio 2: YouTube — `youtube.com` (Navegación Anónima)

* **Nombre de la cookie:** `__Secure-ROLLOUT_TOKEN`
* **Tipo de cookie:** Token técnico de estado anónimo / despliegue progresivo (*A/B testing* de interfaz y funcionalidades). *(Nota: Al no haber iniciado sesión, no se genera aún la cookie de autenticación de cuenta como `__Secure-1PSID`)*.
* **Valor:** `[OFUSCADO_POR_SEGURIDAD]` *(Longitud: 64 caracteres)*
* **Dominio / Ruta:** `.youtube.com` / `/`
* **Banderas detectadas:**
  * `HttpOnly`: ✅ Activada (`true`)
  * `Secure`: ✅ Activada (`true`)
  * `SameSite`: ⚠️ `"None"`
  * `HostOnly`: `false`
* **Diagnóstico:**
  * Implementación adecuada para su propósito funcional. Utiliza el prefijo estandarizado **`__Secure-`**, el cual obliga al navegador a rechazar la cookie si no proviene de un origen HTTPS y no tiene la bandera `Secure: true` explícita.
  * Mantiene `HttpOnly: true` para evitar manipulación desde el DOM del cliente. Su atributo `SameSite="None"` responde a la necesidad de mantener consistencia en reproductores embebidos (*iframes*) en sitios de terceros, sin representar un riesgo crítico de CSRF al no estar vinculada a privilegios de cuenta autenticada.

---

### Sitio 3: Crunchyroll — `crunchyroll.com` (Navegación Anónima)

* **Nombre de la cookie:** `__cf_bm`
* **Tipo de cookie:** Cookie técnica de mitigación de bots (*Cloudflare Bot Management*), no de sesión de usuario.
* **Valor:** `[OFUSCADO_POR_SEGURIDAD]` *(Longitud: 228 caracteres)*
* **Dominio / Ruta:** `.crunchyroll.com` / `/`
* **Banderas detectadas:**
  * `HttpOnly`: ✅ Activada (`true`)
  * `Secure`: ✅ Activada (`true`)
  * `SameSite`: ⚠️ `"None"`
  * `HostOnly`: `false`
* **Diagnóstico:**
  * Configuración endurecida correctamente para una cookie de infraestructura perimetral (WAF/CDN). Tiene un tiempo de vida corto (30 minutos) y bloquea tanto el acceso por JavaScript (`HttpOnly: true`) como el envío por HTTP plano (`Secure: true`), evitando que un atacante pueda extraer el token de validación anti-bot mediante scripts o *sniffing* para reutilizarlo en herramientas automatizadas.
  * Al igual que en los casos anteriores, `SameSite="None"` permite que la validación de tráfico legítimo persista entre redirecciones y recursos externos sin comprometer la integridad de una sesión autenticada.

---

### Cuadro Comparativo de Resultados

| Sitio Web | Cookie Analizada | Rol de la Cookie | `HttpOnly` | `Secure` | `SameSite` | Evaluación General |
| :--- | :--- | :--- | :---: | :---: | :---: | :--- |
| **X (Twitter)** | `auth_token` | Sesión autenticada | `true` | `true` | `None` | **Correcta** *(Mitiga CSRF con token `ct0`)* |
| **YouTube** | `__Secure-ROLLOUT_TOKEN` | Estado anónimo / UI | `true` | `true` | `None` | **Correcta** *(Usa prefijo `__Secure-`)* |
| **Crunchyroll** | `__cf_bm` | Anti-bot (Cloudflare) | `true` | `true` | `None` | **Correcta** *(Vida corta y protegida)* |

---

## 2. Fundamentos Teóricos de Seguridad

### ¿Qué protege exactamente la bandera `HttpOnly`?

La bandera `HttpOnly` protege la **confidencialidad del identificador de sesión (o token almacenado en la cookie) frente al acceso mediante scripts ejecutados en el navegador del cliente**.

Técnicamente, instruye al navegador para que oculte esa cookie ante APIs del DOM como `document.cookie`. Su función principal es **mitigar el impacto de una vulnerabilidad de Cross-Site Scripting (XSS)**: si un atacante logra inyectar y ejecutar código JavaScript malicioso en la aplicación web, el navegador bloqueará cualquier intento del script por leer y exfiltrar la cookie de sesión hacia un servidor externo para realizar un *Session Hijacking* (secuestro de sesión).

*(Nota técnica: `HttpOnly` no previene que exista ni que se ejecute un XSS —un atacante aún podría realizar acciones en nombre de la víctima dentro de la misma página—, pero impide el robo directo del token de sesión).*

---

### ¿Qué riesgo corre el usuario si la cookie de sesión no tiene la bandera `Secure` activada?

Si una cookie de sesión carece de la bandera `Secure`, el navegador tiene permitido enviarla tanto en conexiones cifradas (**HTTPS**) como en conexiones sin cifrar en texto plano (**HTTP**).

El riesgo directo para el usuario es el **secuestro de sesión (*Session Hijacking*) mediante intercepción de tráfico en red (ataques Man-in-the-Middle o *sniffing*)**:

1. **Exposición en tránsito:** Si la víctima se conecta a una red compartida o no confiable (por ejemplo, un Wi-Fi público) y el navegador realiza una petición HTTP hacia el dominio afectado (ya sea porque el usuario escribió la URL sin `https://` antes de ser redirigido, por un ataque de *SSL Stripping*, o al solicitar un subdominio/recurso sin cifrar), el navegador adjuntará automáticamente la cookie de sesión en las cabeceras de esa petición sin cifrar.
2. **Suplantación de identidad:** Cualquier atacante que esté escuchando el tráfico en la misma red (usando herramientas como Wireshark o un proxy interceptador) podrá capturar el paquete HTTP, leer el identificador de sesión en texto plano e inyectarlo en su propio navegador para suplantar completamente la identidad de la víctima sin necesitar su usuario, contraseña ni segundo factor de autenticación (2FA).