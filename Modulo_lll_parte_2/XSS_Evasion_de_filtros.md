# Módulo III — Cross-Site Scripting (XSS)
## Objetivo 1: El Arte de la Evasión de Filtros (WAF Bypass)

---

### 1. ¿Por qué es inútil eliminar únicamente las etiquetas `<script>`?

Intentar mitigar vulnerabilidades de **Cross-Site Scripting (XSS)** mediante una lista negra (*blacklist*) que busca y elimina las cadenas `<script>` y `</script>` es una defensa completamente ineficaz por tres razones técnicas fundamentales:

1. **Superficie de ejecución alternativa en HTML5:** El estándar HTML moderno permite ejecutar código JavaScript fuera de la etiqueta `<script>` utilizando **manejadores de eventos (*Event Handlers*)** como `onerror`, `onload`, `onfocus`, `onmouseover` o `ontoggle` asociados a etiquetas multimedia, formularios o vectores gráficos (`<img>`, `<svg>`, `<body>`, `<input>`, `<details>`).
2. **Fallas de validación no recursiva (Evasión por anidamiento):** Si el filtro realiza una única pasada de reemplazo (`input.replace("<script>", "")`), un atacante puede enviar `<scr<script>ipt>alert(1)</scr</script>ipt>`. Al eliminarse la cadena interna, los extremos restantes vuelven a ensamblarse formando una etiqueta `<script>` válida.
3. **Sensibilidad a mayúsculas/minúsculas y codificaciones:** Filtros mal implementados que no normalizan la entrada antes de evaluarla pueden evadirse alterando el *casing* (`<ScRiPt>`) o utilizando entidades HTML / codificaciones alternativas según el contexto donde se renderice el dato.

---

### 2. Payloads alternativos sin utilizar `<script>`

A continuación se presentan tres vectores distintos capaces de ejecutar JavaScript arbitrario evadiendo un filtro que elimine exclusivamente etiquetas `<script>`:

#### Payload 1: Etiqueta `<img>` con evento `onerror` (Ejecución automática)
```html
<img src="x" onerror="alert(document.domain)">
```
Mecanismo: El navegador intenta cargar una imagen desde la ruta inexistente "x". Al fallar la carga de forma inmediata, se dispara el evento onerror y se ejecuta el código JavaScript sin requerir ninguna interacción de la víctima.

#### Payload 2: Vector `<svg>` con evento `onload`
```html
<svg onload="alert('XSS_Bypass')">
```
Mecanismo: Los gráficos vectoriales escalables (<svg>) forman parte del DOM estándar en HTML5 y soportan el atributo onload, ejecutando el payload apenas el navegador renderiza el nodo en pantalla.

#### Payload 3: Etiqueta `<input>` con `autofocus` y `onfocus` (Sin interacción)
```html
<input type="text" autofocus onfocus="alert(1)">
```
Mecanismo: El atributo autofocus obliga al navegador a posicionar el cursor automáticamente sobre el campo al cargar la página, disparando de forma instantánea el evento onfocus sin que el usuario tenga que hacer clic.

### 3. Mitigación Recomendada
Para erradicar el XSS en lugar de depender de filtros frágiles:

1. **Codificación de salida contextual (Context-Aware Output Encoding):** Convertir caracteres especiales de HTML (<, >, ", ', &) en sus entidades seguras (&lt;, &gt;, &quot;, etc.) antes de insertarlos en el DOM, logrando que el navegador los trate como texto literal y no como etiquetas ejecutables.

2. **Sanitización basada en listas blancas (Whitelisting):** Si la aplicación necesita aceptar HTML enriquecido (por ejemplo, en comentarios con formato), utilizar librerías probadas y mantenidas como DOMPurify, configuradas para permitir únicamente etiquetas y atributos estrictamente inofensivos.

3. **Content Security Policy (CSP):** Implementar la cabecera HTTP Content-Security-Policy deshabilitando la ejecución de scripts en línea (unsafe-inline), lo que bloquea incluso los manejadores de eventos como onerror u onload como capa de defensa en profundidad.
