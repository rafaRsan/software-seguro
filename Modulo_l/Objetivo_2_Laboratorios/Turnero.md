# Laboratorio: Turnero

## Objetivo
Eliminar todos los turnos médicos del usuario "xdalvik" aprovechando una vulnerabilidad IDOR, sin afectar los registros del resto de los pacientes legítimos en la base de datos.

## Herramientas utilizadas
Burp Suite (módulos Proxy, Intruder y Repeater).

## Paso a paso de la resolución

1. **Reconocimiento y captura:** Al navegar por la aplicación, interceptamos la petición `GET` encargada de cargar los turnos en el frontend.
Identificamos que el sistema utiliza un parámetro numérico en la ruta (`/api/1/appointments/`) para solicitar los datos del usuario actual.
2. **Automatización y descubrimiento (Intruder):** Para encontrar el ID de "xdalvik" sin afectar a terceros, enviamos la petición `GET` al módulo Intruder.
Envolvimos el ID original (el `1`) como variable y configuramos un ataque de tipo *Numbers* para iterar un rango numérico
(incluyendo la pista de los IDs 95 al 105 que el profe envió por Whatsapp) entre el número 0 y el número 150 .
3. **Análisis de respuestas:** Al ordenar los resultados del ataque por tamaño (columna *Length*), descubrimos que las peticiones hacia los IDs 1, 49 y 101
tenían un peso significativamente mayor al resto, lo que confirmó que contenían información real de distintos usuarios.
4. **Investigacion de IDs:** Para aislar los datos del atacante, enviamos una petición `GET` manual hacia los IDs que aparecieron en el anterior paso:
`/api/49/appointments/` , `/api/101/appointments/`  mediante el módulo Repeater. El servidor devolvió dos  JSON revelando que los turnos médicos exactos de "xdalvik"
correspondían a los IDs 10, 11, 12 y 13.
5. **Explotación del IDOR (Repeater):** Interceptamos una acción legítima de borrado en la interfaz, la cual utilizaba el método `DELETE` apuntando a un turno propio.
Enviamos esta petición al Repeater y reemplazamos secuencialmente el identificador final por los IDs 10, 11, 12 y 13. Por cada envío, el servidor devolvió un código
HTTP indicando éxito.
6. **Captura de la flag:** Al eliminar los turnos de "xdalvik" de forma quirúrgica sin borrar los registros del usuario 49, se cumplió la condición principal.
Al recargar la interfaz, el sistema validó la limpieza y entregó el código de resolución del laboratorio.
