# Comparación Técnica Directa: XSS vs. CSRF

Aunque tanto el **Cross-Site Scripting (XSS)** como el **Cross-Site Request Forgery (CSRF)** se ejecutan en el contexto del navegador de la víctima y buscan comprometer su sesión o integridad dentro de una aplicación web, **sus arquitecturas de ataque, vectores de origen y modelos de confianza son diametralmente opuestos**.

---

## 1. La Diferencia Fundamental (El Modelo de Confianza)

La distinción técnica esencial radica en **quién confía en quién** de manera equivocada:

* **XSS (Fallo de confianza del Usuario hacia el Sitio):** El navegador de la víctima **confía ciegamente en todo el código que recibe desde el servidor legítimo** (`banco.com`). El atacante logra inyectar código JavaScript malicioso dentro de la aplicación vulnerable; cuando el navegador lo recibe, asume que es parte del código original del sitio y lo ejecuta con todos los privilegios de ese origen (*Same-Origin*).
* **CSRF (Fallo de confianza del Sitio hacia el Usuario):** El servidor legítimo (`banco.com`) **confía ciegamente en cualquier petición HTTP que llegue acompañada de una cookie de sesión válida**. El atacante engaña al navegador de la víctima desde un sitio externo (`evil.com`) para que dispare una petición no deseada hacia el servidor legítimo, aprovechando que el navegador adjunta automáticamente las cookies del dominio destino.

> **Resumen en una línea:** En **XSS**, el atacante ejecuta código *dentro* del dominio legítimo (rompiendo la *Same-Origin Policy* desde adentro). En **CSRF**, el atacante fuerza acciones *desde fuera* del dominio legítimo, sin poder leer jamás la respuesta del servidor.

---

## 2. Tabla Comparativa Técnica

| Dimensión Técnica | Cross-Site Scripting (XSS) | Cross-Site Request Forgery (CSRF) |
| :--- | :--- | :--- |
| **Naturaleza del Fallo** | **Inyección de código** (JavaScript/HTML) por falta de sanitización o codificación de salida (*Output Encoding*). | **Falla lógica de autorización** y gestión de estado por ausencia de validación de origen o tokens anti-CSRF. |
| **Origen de la Ejecución** | Se ejecuta **dentro del origen legítimo** (`https://victima.com`). | Se dispara **desde un origen externo** controlado por el atacante (`https://evil.com`). |
| **Capacidad de Lectura (Exfiltración)** | **Bidireccional (Lectura y Escritura):** El script puede leer el DOM, tokens anti-CSRF, respuestas de APIs y cookies (si no tienen `HttpOnly`) para enviarlos al atacante. | **Unidireccional (Solo Escritura / "Disparo a ciegas"):** El atacante envía una petición para cambiar un estado, pero la *Same-Origin Policy (SOP)* le impide leer la respuesta HTTP del servidor. |
| **Requisito de Sesión Activa** | **No es estrictamente obligatorio** (puede usarse para *defacement*, redirecciones o phishing en páginas públicas), aunque maximiza el impacto si hay sesión. | **Obligatorio:** Sin una cookie de sesión activa en el navegador de la víctima, la petición forjada llega sin autenticar y el ataque fracasa. |
| **Relación con Tokens Anti-CSRF** | **Los neutraliza:** Como el JavaScript corre dentro del mismo dominio, puede leer el token anti-CSRF oculto en el HTML y enviarlo en la petición maliciosa. | **Es bloqueado por ellos:** Al originarse en otro dominio, la SOP impide que `evil.com` lea el token generado por `victima.com`. |
| **Mitigación Principal** | Codificación contextual de salida (*Output Encoding*), sanitización con listas blancas (ej. DOMPurify) y *Content Security Policy (CSP)*. | Tokens Anti-CSRF aleatorios y de un solo uso (*nonces*), atributo `SameSite=Lax/Strict` en cookies y validación de cabeceras `Origin`/`Referer`. |

---

## 3. Impacto Arquitectónico: ¿Por qué XSS es más severo que CSRF?

Desde el punto de vista de una auditoría de seguridad, **XSS se considera una vulnerabilidad de mayor alcance** porque engloba y supera las capacidades del CSRF:

1. **Un XSS permite ejecutar acciones CSRF:** Si un sitio tiene protección perfecta contra CSRF (tokens aleatorios por petición y `SameSite=Strict`), un ataque CSRF externo fallará. Sin embargo, si ese mismo sitio sufre una vulnerabilidad XSS, el atacante puede inyectar un script que lea el token anti-CSRF directamente desde el DOM de la víctima y realice exactamente la misma petición maliciosa desde adentro.
2. **Alcance de la acción:** En CSRF, el atacante está limitado a ejecutar acciones predecibles cuyas estructuras y parámetros ya conoce de antemano (ej. cambiar una contraseña o transferir un monto fijo). En XSS, el atacante tiene control total sobre la sesión del usuario en tiempo real, pudiendo leer mensajes privados, extraer datos personales, capturar pulsaciones de teclas (*keylogging*) o modificar la interfaz visual para robar credenciales.

---

## 4. Mitigación Independiente (Por qué una defensa no cubre a la otra)

Un error frecuente en el desarrollo seguro es creer que asegurar las cookies resuelve ambos problemas:

* Configurar **`HttpOnly`** en la cookie de sesión impide que un **XSS** robe el identificador mediante `document.cookie`, pero **no frena un CSRF** (porque el navegador sigue adjuntando la cookie automáticamente a nivel de red) ni impide que el XSS ejecute acciones en nombre del usuario.
* Configurar **`SameSite=Strict`** en la cookie frena en seco un **CSRF** proveniente de un sitio externo, pero **es completamente inútil contra un XSS**, ya que en este último la petición maliciosa se origina dentro del mismo dominio (*same-site*).