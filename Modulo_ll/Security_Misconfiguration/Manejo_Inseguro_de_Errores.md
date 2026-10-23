# Objetivo 4: Manejo Inseguro de Errores (Stack Traces)

### Contexto y Definiciones
El manejo de errores en una aplicación web determina qué tipo de información se devuelve al cliente cuando ocurre una falla o excepción no controlada en el backend.

**Mensaje de Error Genérico:** Es una respuesta sanitizada y orientada al usuario final que notifica la falla de forma abstracta 
(por ejemplo, 500 Internal Server Error o "Ocurrió un error inesperado"). No revela ningún detalle sobre la arquitectura interna, el código o el estado del servidor.

**Stack Trace (Traza de la Pila):** Es un informe detallado generado por el entorno de ejecución (ej. Java, .NET, Node.js, Python) al producirse una excepción no capturada. 
Muestra la secuencia de llamadas a funciones y métodos que estaban en ejecución al momento del fallo, incluyendo la línea exacta de código, nombres de clases, archivos y 
rutas del sistema operativo.

--- 

### Impacto Técnico: Exposición en Entornos de Producción
Revelar un Stack Trace en producción constituye una falla de configuración y fuga de información (Information Disclosure) clasificada bajo las 
categorías de Security Misconfiguration y Mishandling of Exceptional Conditions según OWASP.

Un atacante puede enviar solicitudes malformadas de forma deliberada (inyectando caracteres especiales, tipos de datos incompatibles o parámetros fuera de rango) 
con el objetivo de forzar excepciones no controladas y analizar la respuesta del servidor.

### Razones por las que facilita la fase de explotación:
**Reconocimiento del Stack Tecnológico (Fingerprinting):**
La traza expone el lenguaje de programación, los frameworks en uso y sus versiones específicas (ej. Spring Framework, Express, ASP.NET). 
Esto permite al atacante buscar directamente vulnerabilidades conocidas (CVEs) y exploits diseñados para esas versiones particulares.

**Fuga de la Estructura Interna del Sistema (Path Disclosure):**
Muestra las rutas absolutas del sistema de archivos en el servidor (ej. /var/www/backend/src/controllers/ o C:\inetpub\wwwroot\app\). 
Esta información facilita la construcción de ataques como Path Traversal o Local File Inclusion (LFI).

**Facilitación de Inyecciones (SQL / ORM / Lógica):**
Si la excepción proviene de la capa de persistencia (como errores de base de datos Oracle ORA-, PostgreSQL o MySQL), el Stack Trace suele revelar nombres de tablas, 
columnas o fragmentos de la consulta fallida. Esto permite al atacante calibrar sus payloads de inyección SQL con precisión.

**Reducción del Tiempo de Explotación:**
Elimina la necesidad de realizar pruebas a ciegas (blind testing). La traza de la pila actúa como un mapa del código fuente que indica exactamente en qué punto la 
aplicación falló al validar la entrada.

---

## Estrategias de Mitigación
Para prevenir la exposición de información sensible a través del manejo de errores:

**Configuración del Entorno de Producción:** Asegurar que las opciones de depuración (debug = false, display_errors = Off) estén estrictamente desactivadas en entornos de 
producción.

**Manejo Global de Excepciones:** Implementar un controlador centralizado de errores (Global Exception Handler) que capture cualquier falla no prevista y devuelva una 
respuesta estandarizada y genérica.

**Registros (Logging) Internos Seguros:** Registrar el Stack Trace completo únicamente en los archivos de logs internos del servidor o en herramientas de monitoreo privadas, 
restringiendo el acceso solo al personal autorizado.
