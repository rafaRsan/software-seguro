# Análisis Técnico: Manipulación Transaccional mediante Cantidades Negativas

## 1. Descripción del Escenario y Clasificación

En el escenario planteado, un usuario intercepta la petición HTTP encargada de añadir ítems al carrito de compras (por ejemplo, mediante un proxy como Burp Suite) y modifica el parámetro `cantidad` de un valor positivo (`1`) a un valor negativo (`-5`). El servidor acepta el dato sin rechazarlo, multiplica el precio unitario por `-5` y obtiene un subtotal negativo que, al procesarse en el *checkout*, invierte el flujo financiero: en lugar de debitar dinero del usuario, acredita saldo a su favor.

Desde la perspectiva de seguridad ofensiva, este hallazgo corresponde a una **Vulnerabilidad de Lógica de Negocio (*Business Logic Flaw*)** combinada con **Validación Insuficiente de Entradas (*CWE-20: Improper Input Validation*)** y manipulación de parámetros (*Parameter Tampering*).

---

## 2. Explicación Técnica: ¿Por qué ocurre esta falla?

A diferencia de una inyección SQL o un XSS, aquí no se rompe la sintaxis del lenguaje ni se inyecta código malicioso; el atacante utiliza la funcionalidad legítima del sistema enviando un tipo de dato válido (un número entero con signo) que el desarrollador no previó. La falla se produce por la convergencia de cuatro errores arquitectónicos:

1. **Validación delegada exclusivamente al Frontend (Confianza en el Cliente):**
   El desarrollador suele restringir el campo de cantidad únicamente en la interfaz gráfica (por ejemplo, utilizando atributos HTML como `<input type="number" min="1">` o controles visuales `-` / `+` en JavaScript). Sin embargo, estas restricciones son meramente cosméticas y se ejecutan en el navegador del cliente. Al interceptar la petición con un proxy antes de que llegue al servidor, el atacante elude por completo la interfaz y envía cualquier valor directamente a la API.

2. **Ausencia de Validación de Dominio y Reglas de Negocio en el Backend:**
   El controlador del backend verifica únicamente el **tipo de dato** (que `cantidad` sea un entero, por ejemplo `int` o `number`), pero omite validar su **rango lógico permitido** dentro del contexto comercial (en una compra, una cantidad solo tiene sentido físico y financiero si es un entero estrictamente positivo: `cantidad >= 1`).

3. **Operaciones Aritméticas Ciegas sin Verificación de Signo:**
   La lógica de cálculo del carrito ejecuta una multiplicación directa sin comprobar el signo de los operandos ni del resultado acumulado:
   $$\text{Subtotal} = \text{Precio\_Unitario} \times \text{Cantidad}$$
   Si $\text{Precio\_Unitario} = 100$ y $\text{Cantidad} = -5$, el motor matemático calcula $\text{Subtotal} = -500$.

4. **Falta de Invariantes Financieras en el Procesamiento del Pago:**
   El módulo de facturación o billetera virtual ejecuta la transacción restando el total del carrito al saldo del usuario ($\text{Saldo\_Nuevo} = \text{Saldo\_Actual} - \text{Total\_Carrito}$). Por ley de signos, restar un monto negativo equivale a una suma ($\text{Saldo\_Actual} - (-500) = \text{Saldo\_Actual} + 500$), provocando un enriquecimiento ilícito y desbalanceando la integridad contable del sistema.

---

## 3. Remediación: Estrategia de Validación Estricta en el Servidor

Para erradicar esta vulnerabilidad de raíz, el desarrollador debe aplicar el principio de **Zero Trust hacia el cliente** y construir controles en múltiples capas dentro del backend:

### A. Validación Estricta de Esquema y Rango (Capa de Entrada / API)
Antes de ejecutar cualquier lógica de negocio, el endpoint debe validar que el parámetro recibido cumpla con restricciones matemáticas exactas mediante esquemas de validación (como *Zod*, *Joi*, *Pydantic* o *Bean Validation*):
* **Tipo estricto:** Debe ser un número entero (`integer`), rechazando flotantes (`1.5`), cadenas de texto (`"-5"`), booleanos o valores nulos.
* **Límite inferior y superior:** Exigir explícitamente que `cantidad >= 1` (o `> 0`) y establecer un máximo razonable según stock o política comercial (ej. `cantidad <= 100`). Si la condición falla, rechazar la petición inmediatamente con un código de estado **HTTP 400 (Bad Request)**.

### B. Validación de Reglas de Negocio e Invariantes (Capa de Servicio)
Independientemente de los datos individuales de entrada, el servicio encargado de calcular el carrito y procesar el pago debe verificar **invariantes del sistema** antes de confirmar la operación:
* **Verificación de totales:** Validar que cada ítem en el carrito tenga `cantidad > 0` y `precio_unitario > 0`.
* **Bloqueo de montos totales inválidos:** Antes de invocar la pasarela de pagos o modificar el saldo de la cuenta, verificar que `total_carrito > 0` (salvo que exista un cupón del 100% explícitamente validado en el servidor, en cuyo caso el mínimo permitido es `0`, **nunca `< 0`**).
* **Recálculo integral en el backend:** Nunca aceptar precios, descuentos ni subtotales enviados desde el frontend; el servidor debe consultar el precio real del producto desde su base de datos utilizando el `product_id`.

### C. Restricciones a Nivel de Base de Datos y Transaccionalidad (Capa de Datos)
Como última línea de defensa (*Defense in Depth*), la integridad debe protegerse en el motor de base de datos:
* **Restricciones `CHECK` y tipos `UNSIGNED`:** Definir las columnas de cantidad como enteros sin signo (`INT UNSIGNED`) o aplicar restricciones explícitas a nivel de tabla:
  ```sql
  ALTER TABLE carrito_items
  ADD CONSTRAINT chk_cantidad_positiva CHECK (cantidad >= 1);
  ```
* **Transacciones Atómicas y Auditoría:** Encapsular la actualización de inventario, el cálculo del total y el débito de saldo dentro de una **transacción atómica** (`BEGIN TRANSACTION ... COMMIT / ROLLBACK`), registrando cualquier intento de envío de valores fuera de rango en los logs de seguridad para alertar sobre posibles intentos de manipulación.