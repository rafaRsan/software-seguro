# Módulo 3 — Inyección SQL (SQLi)

## Objetivo 2: Automatización vs Explotación Manual (`sqlmap`)

### 1. Introducción

`sqlmap` es el estándar de la industria para automatizar la detección y explotación de vulnerabilidades de Inyección SQL (*SQLi*). Es sumamente eficaz para identificar patrones tradicionales (como parámetros GET/POST directos sin protecciones de estado). Sin embargo, depender exclusivamente de herramientas automatizadas introduce un sesgo crítico en las auditorías: existen arquitecturas modernas y flujos de lógica complejos donde `sqlmap` arroja falsos negativos, pero un pentester analizando el tráfico manualmente con **Burp Suite** logra explotar la vulnerabilidad.

---

### 2. Escenarios Reales donde `sqlmap` Falla y la Explotación Manual Triunfa

#### Escenario 1: *Second-Order SQL Injection* (Inyección SQL de Segundo Orden)

* **¿Por qué falla `sqlmap`?**
  `sqlmap` opera bajo un modelo síncrono de estímulo-respuesta inmediata: inyecta un *payload* en una petición $A$ y analiza si la respuesta a esa misma petición $A$ presenta anomalías (errores SQL, cambios booleanos o retardos de tiempo). En una inyección de segundo orden, el dato malicioso ingresa por un *endpoint* seguro (por ejemplo, `POST /api/register` al crear un usuario llamado `admin'--`), donde el backend utiliza consultas parametrizadas para guardarlo en la base de datos. Como el servidor responde `200 OK` sin errores ni demoras, `sqlmap` descarta el parámetro como seguro y pasa al siguiente.
* **¿Cómo lo explota un atacante manual?**
  El auditor comprende el ciclo de vida del dato dentro de la aplicación. Sabe que ese valor queda almacenado en la base de datos y que, posteriormente, otra funcionalidad interna (como un panel de reportes, un proceso *batch* o el *endpoint* `POST /api/change-password`) recupera ese registro de la base y lo **concatena sin sanitizar** en una segunda consulta SQL. Utilizando **Burp Repeater**, el atacante registra el *payload* en el paso 1 y luego ejecuta manualmente la petición del paso 2 para detonar la inyección y observar el impacto.

#### Escenario 2: Tokens Anti-CSRF de un Solo Uso y Flujos Secuenciales Complejos

* **¿Por qué falla `sqlmap`?**
  Muchos formularios sensibles (como transferencias, modificaciones de perfil o búsquedas autenticadas) exigen un token anti-CSRF dinámico válido por una única petición o dependen de una máquina de estados estricta (paso 1 $\rightarrow$ paso 2). Si se lanza `sqlmap` de forma estándar, la primera prueba consume el token válido y las cientos de peticiones posteriores son rechazadas por el servidor con errores `403 Forbidden` o redirecciones al *login*, provocando que la herramienta aborte el escaneo o reporte falsos negativos.
* **¿Cómo lo explota un atacante manual?**
  Aunque `sqlmap` dispone de parámetros avanzados (`--csrf-token`), falla cuando el token viaja dentro de un objeto JSON anidado, una cabecera personalizada o requiere resolver un desafío previo en JavaScript. Un pentester experimentado en **Burp Suite** configura una **Session Handling Rule** acompañada de una **Macro** (o utiliza *Burp Repeater* renovando el token antes de cada envío) para mantener la validez de la sesión y el estado en cada iteración mientras prueba manualmente la sintaxis SQL en el parámetro vulnerable.

#### Escenario 3: Parámetros Encapsulados, Codificados o Firmados en el Cliente

* **¿Por qué falla `sqlmap`?**
  Cuando el parámetro vulnerable no viaja como un par `clave=valor` estándar, sino encapsulado dentro de una estructura compleja (como un objeto JSON codificado en Base64, datos serializados o una petición que exige un *checksum*/firma HMAC calculada por el frontend sobre todo el *body*), las sondas crudas de `sqlmap` corrompen el formato del paquete. El servidor rechaza la solicitud en la capa de validación de esquema o integridad (`400 Bad Request`) antes de que el dato llegue siquiera al motor de base de datos.
* **¿Cómo lo explota un atacante manual?**
  El auditor inspecciona el código JavaScript del frontend o el historial de **Burp Proxy**, identifica cómo se empaqueta o firma la información, coloca el *payload* SQL **dentro** del atributo específico antes de aplicar la codificación/firma correspondiente y reenvía la petición estructuralmente válida para que el backend la desempaquete y la ejecute en la consulta SQL.

#### Escenario 4: Interfaces Conversacionales (Chatbots) y Cláusulas No Estándar (`ORDER BY` / `LIMIT`)

* **¿Por qué falla `sqlmap`?**
  En aplicaciones donde la entrada del usuario atraviesa un intérprete intermedio (como un *chatbot* que traduce lenguaje natural a filtros de búsqueda o APIs GraphQL complejas) o donde la inyección ocurre en cláusulas estructurales como `ORDER BY` o nombres de columnas (donde las consultas preparadas no pueden aplicarse), los *payloads* genéricos de `sqlmap` suelen romper el *parser* intermedio antes de alcanzar el motor SQL.
* **¿Cómo lo explota un atacante manual?**
  Mediante prueba y error controlado en **Burp Repeater**, el pentester observa el comportamiento lógico de la aplicación (por ejemplo, diferencias en el ordenamiento o presencia/ausencia de resultados como oráculo booleano), deduce la estructura exacta de la consulta subyacente y construye condiciones lógicas a medida (`CASE WHEN`, `XOR`, subconsultas) adaptadas al contexto sintáctico exacto.

---

### 3. Conclusión

Las herramientas automatizadas como `sqlmap` son indispensables para acelerar el reconocimiento y explotar hallazgos confirmados, pero **no reemplazan el criterio analítico**. La comprensión manual del flujo HTTP, el manejo de estados de sesión y el análisis del contexto de la aplicación son lo que permite descubrir las vulnerabilidades más críticas y ocultas.
