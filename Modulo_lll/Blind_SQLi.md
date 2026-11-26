# Módulo 3 — Inyección SQL (SQLi)
## Objetivo 1: El Arte de la Inyección Ciega (*Blind SQLi*)

---

## 1. Introducción: ¿Qué es la Blind SQL Injection?

En aplicaciones web endurecidas donde los mensajes de error detallados del motor de base de datos están deshabilitados y la consulta inyectada no refleja sus resultados directamente en la respuesta HTTP (descartando las técnicas *In-Band* como *Error-Based* o *UNION-Based*), el sistema puede aparentar ser invulnerable a simple vista.

Sin embargo, si el backend sigue concatenando entradas del usuario sin parametrizar, es posible explotar una **Blind SQL Injection (Inyección SQL a ciegas)**. En este escenario, los datos no se leen directamente en pantalla, sino que se **infieren carácter por carácter** formulando preguntas lógicas cerradas (`TRUE` o `FALSE`) al motor de base de datos y observando señales indirectas en el comportamiento del servidor.

---

## 2. Diferencia Técnica: Boolean-Based vs. Time-Based Blind SQLi

La diferencia fundamental radica en el **canal lateral (oráculo)** que utiliza el atacante para determinar si su hipótesis lógica fue evaluada como verdadera o falsa por el motor SQL:

| Característica | Boolean-Based Blind SQLi | Time-Based Blind SQLi |
| :--- | :--- | :--- |
| **Mecanismo de detección** | Diferencias observables en el contenido, estructura o metadatos de la respuesta HTTP. | Diferencias medibles en la latencia (tiempo de respuesta) del servidor. |
| **Señal de `TRUE` (Verdadero)** | La página carga con normalidad, devuelve los registros esperados (ej. un catálogo de productos o perfil) o mantiene su longitud estándar en bytes. | El motor ejecuta una función de retardo intencional (ej. `SLEEP(5)`), demorando la respuesta HTTP exactamente el tiempo estipulado. |
| **Señal de `FALSE` (Falso)** | La página devuelve una respuesta vacía, un mensaje genérico (*"No se encontraron resultados"*), altera su código HTTP o varía su tamaño en bytes. | La condición de retardo no se cumple y el servidor responde de inmediato (en milisegundos). |
| **Ejemplo de Payload** | `' AND 1=1 -- -` (True) vs. `' AND 1=2 -- -` (False) | `' AND IF(1=1, SLEEP(5), 0) -- -` |
| **Velocidad y ruido** | Rápida y confiable, pero exige que la respuesta cambie visual o estructuralmente. | Lenta y sensible a la latencia de red (*jitter*), pero funciona incluso cuando la interfaz responde siempre igual. |

---

## 3. Extracción Letra por Letra ante una Pantalla Genérica ("Cargando...")

Cuando una aplicación responde invariablemente con el mismo estado HTTP (`200 OK`) y una pantalla estática o genérica (como *"Cargando..."* o *"Solicitud procesada"*), el atacante pierde el oráculo booleano visual y debe recurrir a **Time-Based Blind SQLi**, utilizando el **reloj** como canal de comunicación.

Para extraer el nombre de la base de datos (o enumerar las tablas internas consultando el diccionario de metadatos `information_schema.tables`) carácter por carácter, se ejecuta el siguiente flujo técnico:

### Paso 1: Confirmar el control del tiempo
Se inyecta un retardo incondicional adaptado al dialecto del motor (MySQL, PostgreSQL, MSSQL u Oracle) para comprobar que la base de datos ejecuta instrucciones temporales:
```sql
' AND (SELECT SLEEP(5)) -- -
```
Si la respuesta HTTP demora aproximadamente 5 segundos en regresar, se confirma la vulnerabilidad.

### Paso 2: Determinar la longitud del nombre de la base de datos
Antes de inferir los caracteres, se acota cuántas iteraciones serán necesarias utilizando la función `LENGTH()` (o `LEN()`):
```sql
' AND IF(LENGTH(database()) = 8, SLEEP(5), 0) -- -
```
Cuando la petición tarde 5 segundos en responder, el atacante sabe que el nombre de la base de datos tiene exactamente 8 caracteres.

### Paso 3: Extracción carácter por carácter mediante ASCII y Subcadenas
Se combinan las funciones `SUBSTRING()` (para aislar una posición específica de la cadena) y `ASCII()` (para convertir el carácter en su valor numérico estándar):

```sql
-- Evaluando la primera letra (posición 1)
' AND IF(ASCII(SUBSTRING(database(), 1, 1)) = 112, SLEEP(5), 0) -- -
```
* **Si es Verdadero:** Si el primer carácter tiene el código ASCII `112` (la letra `'p'`), el motor SQL ejecuta `SLEEP(5)` y el cronómetro del atacante confirma el acierto.
* **Si es Falso:** La condición retorna `0` y el servidor responde al instante, descartando ese carácter y avanzando al siguiente.

### Optimización mediante Búsqueda Binaria
Evaluar todo el conjunto de caracteres imprimibles de forma lineal requeriría decenas de peticiones por cada letra. Para minimizar el ruido en red y acelerar la extracción, herramientas como **sqlmap** o scripts automatizados aplican **búsqueda binaria** utilizando operadores de comparación (`>`):
1. ¿El valor ASCII de la posición 1 es `> 64`? $\rightarrow$ Tarda 5 segundos (**Sí**).
2. ¿Es `> 96`? $\rightarrow$ Tarda 5 segundos (**Sí**).
3. ¿Es `> 112`? $\rightarrow$ Responde en milisegundos (**No**).

Acotando el rango numérico a la mitad en cada consulta, el script deduce con exactitud cada letra en un máximo de **7 peticiones por carácter**, logrando reconstruir el nombre de la base de datos, las tablas ocultas (`information_schema.tables`) y los registros sensibles detrás de una pantalla completamente silenciosa.

---

## 4. Mitigación Recomendada

1. **Consultas Preparadas (Sentencias Parametrizadas):** Separar estrictamente la estructura de la consulta SQL de los parámetros ingresados por el usuario, asegurando que el motor trate toda entrada como un valor literal y nunca como código ejecutable.
2. **Validación Estricta de Entradas:** Validar tipos de datos, longitudes y formatos en el backend antes de construir o ejecutar cualquier operación contra la base de datos.
3. **Principio de Menor Privilegio:** Configurar el usuario de base de datos utilizado por la aplicación web con los permisos mínimos indispensables, restringiendo el acceso innecesario a vistas de metadatos del sistema o funciones de retardo.
