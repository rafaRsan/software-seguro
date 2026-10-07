# Resolución de Laboratorio: Votación (Primera Versión)

## Objetivo
Superar el umbral restrictivo de 4000 votos para asegurar que la opción "UTN" (opUniversidad=1) gane la votación, evadiendo el control que impide votar más de una vez.

## Vulnerabilidad Identificada
**Manejo Inseguro de Estado / Evasión de Restricción del Lado del Cliente.**
El servidor delega la validación de estado (saber si un usuario ya votó) al cliente mediante la asignación de una cookie denominada `voto`. Al eliminar o no enviar esta cookie en peticiones subsecuentes, el servidor falla en identificar al usuario y procesa cada solicitud como un voto nuevo y legítimo.

## Herramientas Utilizadas
* Burp Suite (Módulos Proxy y Repeater)
* Terminal (Intérprete Bash y comando `curl`)

## Paso a paso de la resolución

1. **Reconocimiento y Captura:** 
   Se interceptó la petición original de la votación mediante Burp Suite. Al analizar el flujo normal, se observó que tras emitir el primer voto mediante una petición `POST`, el servidor respondía estableciendo la cookie `voto`.
   
2. **Análisis del Mecanismo de Bloqueo:**
   Al intentar emitir un segundo voto desde el navegador, el sistema mostró un mensaje de error indicando que no se podía votar más de una vez. Al analizar la petición bloqueada, se confirmó que el navegador adjuntaba la cookie `voto` obtenida previamente.

3. **Prueba de Concepto (Evasión):**
   Se envió la petición de voto original al módulo **Repeater** de Burp Suite. Al eliminar deliberadamente la cookie `voto` de las cabeceras de la petición y enviarla repetidas veces, el servidor procesó los votos con éxito de manera continua, confirmando la vulnerabilidad.

4. **Automatización del Ataque:**
   Para superar la desventaja numérica (alrededor de 3600 votos faltantes) de manera eficiente y eludir las limitaciones de velocidad de Burp Suite Community, se exportó la petición limpia (sin la cookie `voto`) como un comando `curl`.
   
5. **Ejecución del Script (Fuerza Bruta):**
   Se encapsuló el comando `curl` dentro de un ciclo `for` en la terminal de Bash para ejecutar el envío masivo en segundo plano de forma silenciosa:
   
   ```bash
   for i in {1..3600}; do
     curl --path-as-is -s -k -X $'POST' \
     -H $'Host: chl-4bf9bd71-e777-42a6-b434-407e833832d3-votacion.softwareseguro.com.ar' \
     ... [Resto de cabeceras legítimas SIN la cookie "voto"] ... \
     --data-binary $'opUniversidad=1' \
     $'https://chl-4bf9bd71-e777-42a6-b434-407e833832d3-votacion.softwareseguro.com.ar/src/ctl/votacion.ctl.php'
   done
   ```

6. **Validación y Captura de la Flag:**
   Una vez que el ciclo iterativo devolvió el control de la terminal, se recargó la interfaz web. El sistema validó que la UTN superó los 4000 votos y entregó la flag de resolución del laboratorio.