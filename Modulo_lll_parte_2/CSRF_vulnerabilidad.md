# Anatomía de un Ataque CSRF (Cross-Site Request Forgery)


## 1. ¿Por qué ocurre el fallo de seguridad? (La raíz del problema)

La vulnerabilidad **Cross-Site Request Forgery (CSRF)** no existe porque el atacante haya robado una contraseña o interceptado una cookie (a diferencia de XSS o *sniffing*), sino por una **confianza excesiva del servidor** combinada con el **comportamiento estándar de los navegadores web**.

Por diseño, cuando un usuario inicia sesión en una aplicación web (por ejemplo, `banco.com`) y recibe una cookie de sesión, el navegador la guarda en su almacén interno (*Cookie Jar*). A partir de ese momento, **el navegador adjunta automáticamente esa cookie en absolutamente todas las peticiones HTTP dirigidas a `banco.com`**, sin importar desde qué pestaña, dominio o sitio externo se haya originado la petición.

El fallo ocurre cuando el backend de `banco.com` cumple estas dos condiciones:
1. **Confía ciegamente en la cookie de sesión** como única prueba de que el usuario quiso realizar una acción.
2. **No verifica la intencionalidad ni el origen real** de la solicitud (es decir, no distingue si el usuario hizo clic en un botón dentro de `banco.com` o si la petición fue disparada ocultamente desde `evil.com`).

*(Nota: No hace falta que la víctima tenga la pestaña del banco abierta en ese instante; basta con que tenga una sesión activa cuya cookie no haya expirado).*

---

## 2. Flujo exacto de un ataque CSRF exitoso

Supongamos un escenario clásico donde el objetivo es cambiar la contraseña (o transferir dinero) de una víctima en `banco.com`.

### Fase 1: Reconocimiento y armado del *exploit* (PoC)
1. **Mapeo del endpoint:** El atacante crea una cuenta legítima en `banco.com`, realiza la acción crítica (ej. cambiar su contraseña) e intercepta el tráfico con un proxy como **Burp Suite**.
2. **Análisis de la petición:** Identifica el endpoint, el método HTTP y los parámetros exactos. Por ejemplo:
   ```http
   POST /api/change-password HTTP/1.1
   Host: banco.com
   Content-Type: application/x-www-form-urlencoded

   new_password=PasswordDelAtacante123
   ```
3. **Creación del sitio trampa:** El atacante aloja una página maliciosa (`https://evil.com/promo.html`) con un formulario HTML oculto que apunta al endpoint vulnerable y se envía automáticamente mediante JavaScript apenas carga la página:
   ```html
   <html>
     <body>
       <form id="csrfForm" action="https://banco.com/api/change-password" method="POST">
         <input type="hidden" name="new_password" value="PasswordDelAtacante123" />
       </form>
       <script>
         document.getElementById("csrfForm").submit();
       </script>
     </body>
   </html>
   ```

### Fase 2: Ejecución sobre la víctima
4. **Sesión activa:** La víctima inicia sesión en `banco.com` y recibe su cookie (`Set-Cookie: session_id=xyz789`).
5. **Ingeniería social:** Sin cerrar sesión en el banco, la víctima recibe un enlace malicioso por correo o mensajería (`https://evil.com/promo.html`) y lo abre en el mismo navegador.
6. **Disparo silencioso:** Al cargar `evil.com`, el script envía el formulario `POST` hacia `https://banco.com/api/change-password`.
7. **Inyección automática de credenciales:** El navegador detecta que el destino de la petición es `banco.com`, busca en su almacén de cookies y **pega automáticamente** la cabecera `Cookie: session_id=xyz789` en la solicitud forjada.
8. **Procesamiento y compromiso:** El servidor de `banco.com` recibe la petición, valida que `session_id=xyz789` pertenece a la víctima y ejecuta el cambio de contraseña. El atacante toma control total de la cuenta (*Account Takeover*).

---

## 3. Mitigación mediante el atributo `SameSite`

La bandera `SameSite` en la cabecera `Set-Cookie` le indica al navegador bajo qué contexto de origen tiene permiso para enviar la cookie en peticiones salientes. Al restringir el envío automático en peticiones *cross-site* (de terceros), corta de raíz el mecanismo del ataque.

### Valores críticos de `SameSite`

| Valor | Comportamiento del Navegador | Impacto frente a CSRF |
| :--- | :--- | :--- |
| **`SameSite=None`** | Envía la cookie en **todas** las peticiones, tanto internas (*same-site*) como desde sitios externos (*cross-site*). Exige obligatoriamente el atributo `Secure` (HTTPS). | **Vulnerable.** Deja la puerta abierta a CSRF si no hay tokens anti-CSRF. |
| **`SameSite=Lax`** *(Estándar moderno por defecto)* | Bloquea el envío de la cookie en peticiones *cross-site* subyacentes (como envíos de formularios `POST`, peticiones `fetch`/XHR, `<iframe>` o etiquetas `<img>`). **Excepción:** Sí la envía en navegaciones de nivel superior (*top-level navigation*) que usen métodos seguros como **`GET`** (por ejemplo, cuando un usuario hace clic en un enlace normal `<a href="...">`). | **Mitigación alta.** Frena todos los ataques CSRF basados en formularios `POST` automáticos. **Advertencia:** Si la aplicación comete el error de ejecutar acciones sensibles mediante `GET` (ej. `GET /borrar-cuenta?id=1`), sigue siendo vulnerable. |
| **`SameSite=Strict`** | Bloqueo total. El navegador **nunca** envía la cookie si la petición se origina fuera del dominio exacto, ni siquiera si el usuario hace clic en un enlace legítimo desde otro sitio (obligándolo a volver a autenticarse si entra desde un link externo). | **Mitigación máxima.** Elimina por completo cualquier vector CSRF, incluso en peticiones `GET`, a costa de una leve fricción en la experiencia de usuario. |

---

## 4. Conclusión y Defensa en Profundidad

Aunque configurar `Set-Cookie: session_id=...; HttpOnly; Secure; SameSite=Lax` (o `Strict` para operaciones bancarias/críticas) neutraliza la gran mayoría de los ataques CSRF modernos, **no debe ser la única barrera**. 

Como buena práctica de seguridad en el desarrollo (*defensa en profundidad*), esta configuración debe complementarse siempre con:
* **Tokens Anti-CSRF (Synchronizer Token Pattern):** Un valor aleatorio, impredecible y de un solo uso (*nonce*) generado por el backend e incluido en cada formulario o cabecera personalizada (`X-CSRF-Token`), el cual un sitio externo no puede leer ni adivinar debido a la *Same-Origin Policy (SOP)*.
* **Uso estricto de métodos HTTP:** Nunca procesar cambios de estado o acciones sensibles a través de peticiones `GET`.
