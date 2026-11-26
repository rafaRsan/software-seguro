# Módulo 3 — Inyección SQL (SQLi)
## Objetivo 3: La Única Mitigación Definitiva (*Prepared Statements*)

### 1. ¿Por qué fallan las Listas Negras y el Escape Manual?

Antes de analizar la solución definitiva, es necesario comprender por qué los enfoques reactivos tradicionales son insuficientes frente a una Inyección SQL:

1. **Listas Negras (*Blacklisting*):** Bloquear palabras clave como `SELECT`, `UNION`, `OR` o caracteres como `--` falla sistemáticamente porque la sintaxis SQL es sumamente flexible. Un atacante puede evadir estos filtros utilizando variaciones de capitalización (`sElEcT`), comentarios intercalados (`UN/**/ION`), codificaciones alternativas (URL-encoding, Hex, Unicode), operadores equivalentes (`||` en lugar de `OR`, `LIKE` en lugar de `=`) o funciones nativas del motor.
2. **Escape Manual de Caracteres:** Depende de la codificación de caracteres (*charset*) de la conexión y del contexto sintáctico. Por ejemplo, escapar comillas simples (`'`) no sirve de nada cuando el parámetro inyectado es numérico (`WHERE id = 5 OR 1=1`) porque no existen comillas que romper, y puede verse comprometido ante vulnerabilidades de codificación multibyte (como *GBK* o *Shift-JIS*).

Ambos métodos comparten el mismo defecto arquitectónico: **siguen concatenando la entrada del usuario dentro de la misma cadena de texto que contiene las instrucciones SQL**.

---

### 2. Arquitectura Interna de una Consulta Parametrizada (*Prepared Statement*)

Una **Consulta Parametrizada** (o *Prepared Statement*) erradica la vulnerabilidad a nivel arquitectónico porque **separa físicamente el código (la estructura de la consulta) de los datos (los valores enviados por el usuario)**, ejecutando el procesamiento en dos fases independientes dentro del motor de base de datos (DBMS):

#### Fase 1: Compilación y Generación del Plan de Ejecución (*Prepare*)
1. El backend envía al motor de base de datos una plantilla SQL que contiene **marcadores de posición (*placeholders*)** en lugar de los valores reales:
   ```sql
   SELECT id, email, role FROM users WHERE username = ? AND status = ?;
   ```
2. El motor de base de datos recibe esta plantilla, ejecuta el análisis léxico (*lexing*), el análisis sintáctico (*parsing*), construye el **Árbol de Sintaxis Abstracta (AST)** y genera el **Plan de Ejecución** optimizado.
3. En este instante, **la estructura lógica de la consulta queda sellada e inmutable**. El motor ya definió exactamente qué tablas va a consultar, qué columnas va a devolver y qué operadores lógicos va a evaluar.

#### Fase 2: Vinculación de Parámetros y Ejecución (*Bind & Execute*)
1. Posteriormente, el backend envía los valores ingresados por el usuario a través de un protocolo binario separado (como argumentos vinculados a los *placeholders* `?` o `:param`).
2. El motor inserta esos valores directamente en los nodos hoja (literales) del plan de ejecución ya compilado, **sin volver a pasarlos por el analizador sintáctico (parser) de SQL**.

---

### 3. ¿Por qué neutraliza por completo cualquier intento de inyección?

Supongamos que un atacante envía como nombre de usuario el *payload* clásico:
```text
admin' OR '1'='1' -- -
```

* **En una consulta concatenada (vulnerable):** El string rompe las comillas del código SQL y el *parser* interpreta `OR '1'='1'` como una nueva rama lógica ejecutiva dentro del AST, alterando la intención original de la consulta.
* **En un *Prepared Statement* (seguro):** Como el AST ya fue compilado en la Fase 1, el motor **nunca interpreta las comillas simples (`'`), los operadores (`OR`) ni los comentarios (`--`) como sintaxis SQL**. Para el motor de base de datos, el string completo `admin' OR '1'='1' -- -` es simplemente una cadena literal opaca. La base de datos buscará literalmente un usuario cuyo campo `username` sea igual a esa cadena de 22 caracteres y, al no encontrarlo, devolverá `0 filas` de manera segura.

---

### 4. Consideración Crítica de Auditoría (El Límite de los *Prepared Statements*)

Como regla fundamental de arquitectura segura (mencionada en la Clase 8), los *Prepared Statements* protegen únicamente los **valores de datos** (dentro de cláusulas `WHERE`, `INSERT`, `UPDATE`, `VALUES`).

Los motores SQL **no permiten parametrizar identificadores estructurales**, tales como:
* Nombres de tablas (`FROM ?`)
* Nombres de columnas (`SELECT ?`)
* Cláusulas de ordenamiento y dirección (`ORDER BY ? ASC/DESC`)

Para estos casos específicos donde la consulta deba ser dinámica, la mitigación complementaria obligatoria es aplicar **Listas Blancas (*Whitelisting*) estrictas** en el backend (validando mediante un `switch` o diccionario cerrado que el parámetro coincida exactamente con una columna permitida antes de construir la sentencia).