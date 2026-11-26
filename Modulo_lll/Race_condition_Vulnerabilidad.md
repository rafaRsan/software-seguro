# Anatomía de una Condición de Carrera (Race Condition / TOCTOU)

## 1. Fundamento Técnico: ¿Qué es una Race Condition y el fallo TOCTOU?

Una **Condición de Carrera (*Race Condition*)** es una vulnerabilidad de lógica de negocio y concurrencia que ocurre cuando un sistema multihilo procesa múltiples solicitudes simultáneas que acceden y modifican un mismo recurso compartido (por ejemplo, el estado de un cupón de descuento o el saldo de una cuenta), pero el resultado correcto depende de que dichas operaciones se ejecuten de forma estrictamente secuencial y no en paralelo.

En el escenario del canje de un cupón de descuento de un solo uso, la falla responde al patrón clásico **TOCTOU (*Time-of-Check to Time-of-Use*)**:

1. **Time-of-Check (Tiempo de Comprobación):** El backend recibe la petición `POST /api/cupon/canjear`, consulta la base de datos y verifica si el cupón cumple la condición de validez (`usado == false` o `usos_restantes > 0`).
2. **Ventana de Carrera (Race Window):** Existe un lapso crítico (generalmente de unos pocos milisegundos o microsegundos) entre el momento en que el servidor lee que el cupón es válido y el momento en que ejecuta la escritura en la base de datos para invalidarlo (`UPDATE cupones SET usado = true`).
3. **Time-of-Use (Tiempo de Uso):** Si un atacante logra que 50 hilos distintos del servidor ejecuten el paso 1 (*Check*) antes de que el primer hilo complete el paso 2 (*Update*), todos los hilos recibirán la respuesta de que el cupón sigue disponible y aplicarán el descuento múltiples veces.

---

## 2. El Obstáculo de Red (*Network Jitter*) y Cómo Lo Resuelve Turbo Intruder

Si un atacante simplemente envía 50 peticiones seguidas desde un script convencional o haciendo clic repetidas veces, el ataque suele fallar debido al **Network Jitter** (fluctuación e imprevisibilidad en la latencia de red). Los paquetes llegan escalonados al servidor: la primera petición invalida el cupón y las 49 restantes son rechazadas.

Para superar este problema y forzar la colisión exacta dentro de la *Race Window*, herramientas especializadas como **Turbo Intruder** (extensión de Burp Suite) implementan técnicas de sincronización a nivel de red:

### A. *Last-Byte Synchronization* (Para HTTP/1.1)
1. **Apertura paralela:** Turbo Intruder abre 50 conexiones TCP independientes contra el servidor objetivo.
2. **Retención del último byte:** Envía el 99% de cada petición HTTP (todas las cabeceras y el cuerpo excepto el último carácter) y las deja en espera. El servidor recibe los fragmentos pero mantiene los hilos bloqueados aguardando el cierre de cada petición para pasarlas a la capa de aplicación.
3. **Disparo sincronizado:** Una vez que las 50 conexiones están listas, la herramienta libera simultáneamente los 50 bytes finales. Al ser paquetes diminutos, viajan prácticamente sin desfase, provocando que el servidor dispare los 50 hilos de ejecución en el mismo instante.

### B. *Single-Packet Attack* (Para HTTP/2)
Cuando el servidor soporta **HTTP/2** (multiplexación de flujos sobre una única conexión TCP), Turbo Intruder aplica una técnica aún más precisa:
1. Empaqueta las 50 peticiones HTTP completas dentro de un **único paquete TCP** (o un número mínimo de tramas según el MTU de la red).
2. Al viajar todas las peticiones juntas dentro de la misma estructura física de red, el **jitter entre peticiones se reduce a cero**. Cuando el servidor recibe y desencapsula ese paquete TCP, procesa las 50 solicitudes en el mismo microsegundo.

---

## 3. Flujo Paso a Paso de la Explotación

1. **Captura de la petición legítima:** El atacante intercepta con Burp Suite la solicitud válida de aplicación del cupón (por ejemplo, `POST /checkout/apply-coupon` con su cookie o JWT de sesión y el código promocional).
2. **Envío a Turbo Intruder:** Envía esa petición a Turbo Intruder y configura un script en Python interno utilizando el parámetro `gate` (compuerta de sincronización) para encolar las 50 copias de la solicitud sin enviarlas todavía (`engine.queue(target.req, gate='race1')`).
3. **Apertura de la compuerta:** Ejecuta `engine.openGate('race1')`, disparando todas las peticiones sincronizadas mediante *Last-Byte Sync* o *Single-Packet Attack*.
4. **Explotación del estado inconsistente:**
   * Los 50 hilos del backend consultan la base de datos casi en simultáneo y leen `cupon_valido = true`.
   * Varios hilos (por ejemplo, 15 de los 50) logran aplicar el descuento sobre el carrito o acreditar el saldo promocional antes de que el primer hilo ejecute el `COMMIT` que marca el cupón como agotado.
5. **Resultado:** El atacante obtiene un descuento acumulativo (o saldo gratuito) utilizando un cupón diseñado estrictamente para un único canje.

---

## 4. Mitigación y Controles Arquitectónicos en el Backend

Para erradicar las condiciones de carrera en operaciones críticas, no basta con validar en el código de aplicación; se debe garantizar la **atomicidad** en la capa de persistencia:

* **Bloqueos Pesimistas a Nivel de Base de Datos (*Pessimistic Locking*):** Utilizar sentencias como `SELECT ... FOR UPDATE` dentro de una transacción ACID. Esto bloquea la fila específica del cupón en la base de datos desde el momento en que el primer hilo la lee, obligando a los otros 49 hilos a esperar en cola hasta que la primera transacción haga `COMMIT` (momento en el cual ya leerán `usado = true`).
* **Actualizaciones Atómicas Condicionales:** En lugar de separar la lectura de la escritura, ejecutar una única consulta atómica condicionada y verificar el número de filas afectadas:
  ```sql
  UPDATE cupones
  SET usado = true, usuario_id = :user_id
  WHERE codigo = :codigo AND usado = false;
  ```
  Si la consulta devuelve `0 filas afectadas`, el backend rechaza el canje inmediatamente.
* **Claves de Idempotencia (*Idempotency Keys*) y Locks Distribuidos:** En arquitecturas distribuidas o de microservicios, implementar cerrojos en memoria rápida (como **Redis** con `SETNX` / *Redlock*) asociados al identificador del cupón o del usuario para serializar las peticiones concurrentes.
