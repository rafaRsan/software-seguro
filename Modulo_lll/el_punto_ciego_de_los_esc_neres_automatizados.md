# Análisis Técnico: El Punto Ciego de los Escáneres Automatizados frente a la Lógica de Negocio

## 1. Introducción

Los escáneres de vulnerabilidades automatizados —como **Nessus**, **Acunetix**, **Nuclei** o el escáner activo de **Burp Suite**— están diseñados para detectar **fallas técnicas de implementación** o **configuraciones inseguras** conocidas (como inyecciones SQL, Cross-Site Scripting, cabeceras ausentes o versiones de software desactualizadas). Sin embargo, cuando se enfrentan a vulnerabilidades de **Lógica de Negocio (*Business Logic Flaws*)**, estas herramientas presentan un "punto ciego" estructural que les impide identificarlas.

---

## 2. ¿Por qué la Lógica de Negocio es invisible para la automatización?

El fracaso de las herramientas automatizadas frente a fallas lógicas no se debe a una deficiencia de software, sino a la naturaleza misma de estas vulnerabilidades:

1. **Ausencia de errores sintácticos o firmas reconocibles:**
   Los escáneres operan inyectando *payloads* anómalos (comillas simples `'`, etiquetas `<script>`, caracteres de control) y analizando si la respuesta del servidor rompe su sintaxis o devuelve mensajes de error conocidos. En una falla de lógica de negocio, **el tráfico malicioso es sintácticamente idéntico al tráfico legítimo**. Enviar un precio modificado, cambiar el estado de un pedido o saltarse un paso del carrito son peticiones HTTP válidas y bien formadas que no disparan ninguna firma de error técnico.

2. **Falta de comprensión del contexto y la intención del negocio:**
   Un algoritmo automatizado analiza peticiones aisladas y estructuras de código, pero carece de contexto semántico sobre las reglas comerciales de la organización. Por ejemplo, un escáner no tiene forma de deducir que un usuario con rol `"Cliente"` no debería poder invocar un endpoint que cambia el estado de una orden a `"Enviado"`, ni que reservar 500 asientos simultáneamente sin pagar constituye una denegación de servicio lógica, o si por el contrario es una funcionalidad intencional (*feature*) para compras corporativas.

3. **Inexistencia de diccionarios o *payloads* estándar:**
   Mientras que para fuerza bruta o *fuzzing* existen listas universales, no existe un diccionario genérico de "fallas lógicas". Cada aplicación tiene sus propios flujos multipaso, máquinas de estado y reglas de dominio diseñadas a medida; por lo tanto, cada vector de explotación lógica es único para ese sistema.

---

## 3. ¿Por qué la Auditoría Manual es indispensable?

Dado que las herramientas no pueden inferir la intención detrás del diseño de una aplicación, la intervención del analista humano se vuelve insustituible por tres razones clave:

* **Modelado del "Happy Path" y máquinas de estado:** El auditor manual navega la aplicación como un usuario legítimo para comprender el flujo esperado (por ejemplo: *Selección $\rightarrow$ Carrito $\rightarrow$ Pago $\rightarrow$ Confirmación*). Solo entendiendo cómo debe funcionar el negocio puede identificar qué ocurre al alterar el orden de los pasos (*Workflow Bypass*) o al omitir peticiones intermedias.
* **Pensamiento lateral e interpretación de anomalías sutiles:** Un pentester utiliza proxies de intercepción (como Burp Suite Proxy y Repeater) para observar cómo reacciona el backend ante cambios lógicos (como cantidades negativas, reutilización de tokens en distintas fases o manipulación de parámetros ocultos) e interpretar diferencias sutiles en el comportamiento del sistema que un escáner descartaría como respuestas normales (`HTTP 200 OK`).
* **Evaluación de impacto real y criterios de aceptación:** El criterio humano permite distinguir entre un comportamiento esperado del negocio y una vulnerabilidad crítica (como fraude financiero o escalada de privilegios), evaluando el impacto real sobre la Confidencialidad, Integridad y Disponibilidad de la operación.