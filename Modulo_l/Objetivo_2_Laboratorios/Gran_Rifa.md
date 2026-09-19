# Laboratorio: Gran Rifa 2019

## Objetivo
Obtener una rifa para el usuario John Bacus sin realizar el pago correspondiente.

## Herramientas utilizadas
Inspector del navegador (Tecla F12).

## Paso a paso de la resolución

1. **Reconocimiento :** Al cargar la página, inspeccionamos la petición `GET` original que trae la lista de números.
Observamos que la estructura de datos JSON incluye un parámetro interno llamado `"esta_pago"`, el cual se encuentra en `false`.
2. **Análisis del comportamiento normal:** Utilizamos el botón "Editar" provisto por la interfaz, el cual solo permite modificar el nombre del comprador (ID 4).
Esta acción genera y envía una petición `POST` al endpoint `/api/numeros/4/editar` con el JSON `{"comprador": "John Bacus"}`.
3. **Ataque:** Para evadir la restricción visual del frontend, hacemos clic derecho sobre esa petición capturada y seleccionamos **Editar y reenviar**.
4. **Exploit:** En el cuerpo de la petición (body), eliminamos el campo original del comprador e inyectamos directamente el parámetro de estado, enviando el JSON modificado:
`{"esta_pago": true}`
5. **Ejecución y captura:** Forzamos el envío de esta petición manteniendo el método `POST`. El servidor, al carecer de validaciones estrictas sobre qué campos tiene
permitido modificar el usuario, acepta el cambio y nos devuelve la flag de resolución del laboratorio.
