# Análisis Arquitectónico: El Espejismo del Cifrado en el Cliente (*Frontend Encryption*)

## 1. Diagnóstico: ¿Por qué es una falsa sensación de seguridad?

La afirmación de los arquitectos del sistema parte de una premisa errónea: asumir que un atacante actúa únicamente como un observador pasivo en la red externa (como en un ataque *Man-in-the-Middle* tradicional sobre Wi-Fi) y que el navegador del usuario es un entorno cerrado e inviolable.

En un escenario de pentesting web o fraude interno, **el atacante es el propio usuario legitimado controlando su navegador**. Por definición, todo código JavaScript que el servidor envía al cliente para ejecutarse en el navegador es **público, inspeccionable y manipulable**. Intentar ocultar la lógica de negocio o los mecanismos criptográficos dentro del frontend es una variante de **Seguridad por Oscuridad (*Security through Obscurity*)**, una práctica explícitamente desaconsejada en el diseño de software seguro (*OWASP A04: Insecure Design* / *A02: Cryptographic Failures*).

### ¿Dónde reside el fallo fundamental de diseño?

El fallo estructural se apoya en tres errores de arquitectura:

1. **Exposición inevitable de la llave y del algoritmo en el cliente:**
   Para que el código JavaScript del navegador pueda cifrar el cuerpo (`body`) de la petición `POST` antes de enviarlo, el navegador **debe conocer obligatoriamente** tanto el algoritmo utilizado (por ejemplo, AES-CBC, AES-GCM o RSA) como la clave de cifrado (y el vector de inicialización `IV`, si aplica). Ya sea que la clave esté *hardcodeada* en un archivo `.js`, ofuscada, o descargada dinámicamente desde un endpoint inicial, reside en la memoria del navegador que el atacante controla por completo.

2. **Confusión entre Confidencialidad en Tránsito e Integridad de Datos:**
   El transporte cifrado ya lo resuelve de forma estandarizada el protocolo **TLS/HTTPS**. Añadir una capa artesanal de cifrado en JavaScript no impide que el dueño de la máquina modifique los datos **antes** de que pasen por la función de cifrado o que replique exactamente la misma rutina criptográfica sobre datos alterados.

3. **Falta de validación y control de autorización en el Backend:**
   Si el servidor confía ciegamente en cualquier *payload* que logre descifrar correctamente (asumiendo que "si viene bien cifrado, proviene de mi interfaz legítima y no fue tocado"), está delegando la integridad del negocio al cliente.

## 2. Pasos Tácticos del Atacante para Evadir la Protección

Un atacante equipado con las herramientas de desarrollo del navegador (*DevTools*) y un proxy como **Burp Suite** puede desmontar esta defensa en cuatro fases tácticas:

### Fase 1: Reconocimiento y Localización de la Rutina Criptográfica
* El atacante abre las *DevTools* del navegador (`F12`) o intercepta los archivos `.js` desde el historial HTTP de Burp Suite.
* Utiliza búsquedas globales en el código fuente del frontend buscando palabras clave como `encrypt`, `CryptoJS`, `SubtleCrypto`, `AES`, `RSA`, `JSON.stringify` o el nombre del endpoint destino (ej. `/api/transferir`).
* Si el código está minificado u ofuscado, utiliza la función *Pretty-Print* (`{}`) del navegador para formatearlo y hacerlo legible.

### Fase 2: Depuración Dinámica y Extracción de Parámetros (*Hooking* / *Breakpoints*)
* En lugar de perder tiempo leyendo miles de líneas de código ofuscado, el atacante coloca un **Breakpoint** en el navegador justo antes de que se dispare la petición `XHR` / `Fetch` o dentro de la función de cifrado.
* Al ejecutar una operación legítima en la web, la ejecución se congela en ese punto exacto. Desde la consola de depuración, el atacante inspecciona las variables en memoria en tiempo real y extrae:
  * El objeto JSON en texto plano antes de ser cifrado.
  * El algoritmo exacto, modo de operación y *padding*.
  * La clave secreta (`key`) y el vector de inicialización (`IV`).

### Fase 3: Manipulación Pre-Cifrado (Evasión directa desde el Navegador)
La vía más rápida para el atacante ni siquiera requiere usar Burp Suite para re-cifrar:
* Estando detenido en el *breakpoint* justo antes de llamar a `encrypt(payload)`, el atacante modifica directamente los valores de la variable en la consola del navegador (por ejemplo, cambia `monto: 100` a `monto: 50000` o altera el `destino_id`).
* Luego reanuda la ejecución (*Resume script execution*). El propio código JavaScript legítimo de la aplicación cifra el *payload* alterado con las llaves válidas y lo envía al backend como si nada hubiera pasado.

### Fase 4: Automatización y Fuerza Bruta mediante Proxy (Burp Suite)
Si el atacante necesita enviar múltiples peticiones automatizadas (por ejemplo, para probar un IDOR o *fuzzing* con **Burp Intruder**):
* Replica la función de cifrado en un pequeño script externo (en Python o Node.js) o utiliza una extensión de Burp Suite (como *Custom Actions* o *PyCrypto*) configurada con la clave y el algoritmo extraídos en la Fase 2.
* De este modo, genera *payloads* maliciosos, los cifra con el formato exacto que espera el servidor y los envía en masa a través de Burp Repeater o Intruder.

## 3. Remediación Recomendada

Para proteger realmente las transacciones financieras, el equipo debe descartar el cifrado artesanal como mecanismo de integridad y aplicar controles reales del lado del servidor:

1. **Confiar en TLS (HTTPS) para el transporte:** Utilizar TLS 1.2/1.3 correctamente configurado con HSTS para proteger los datos en tránsito frente a terceros.
2. **Validación estricta en el Backend (*Zero Trust*):** Validar en el servidor cada parámetro descifrado (tipos de datos, rangos positivos, existencia de fondos, propiedad de la cuenta origen mediante el JWT/sesión).
3. **Autenticación de Operaciones Críticas:** Implementar mecanismos de autorización fuerte por transacción (como **2FA / OTP** o firma digital con biometría/llaves de hardware FIDO2) validados íntegramente en el backend antes de ejecutar movimientos de dinero.
