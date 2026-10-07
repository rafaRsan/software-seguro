# Laboratorio: Votación (Primera Versión)

## Objetivo
Superar el umbral restrictivo de 4000 votos para asegurar que la UTN gane la votación, evadiendo el control que impide votar más de una vez.

## Herramientas Utilizadas
* Burp Suite (Módulos Proxy y Repeater)
* Terminal (Intérprete Bash y comando `curl`)

## Paso a paso de la resolución

1. Se interceptó la petición original de la votación mediante Burp Suite. Al analizar el flujo normal, se observó que tras emitir el primer voto mediante una petición `POST`, el servidor respondía estableciendo la cookie `voto`.
   
2. Al intentar emitir un segundo voto desde el navegador, el sistema mostró un mensaje de error indicando que no se podía votar más de una vez. Al analizar la petición bloqueada, se confirmó que el navegador adjuntaba la cookie `voto` obtenida previamente.

3. Se envió la petición de voto original al módulo **Repeater** de Burp Suite. Al eliminar deliberadamente la cookie `voto` de las cabeceras de la petición y enviarla repetidas veces, el servidor procesó los votos con éxito de manera continua, confirmando la vulnerabilidad.

4. Para superar la desventaja numérica (alrededor de 3600 votos faltantes) de manera eficiente y eludir las limitaciones de velocidad de Burp Suite Community, se exportó la petición limpia (sin la cookie `voto`) como un comando `curl`.
   
5. Se encapsuló el comando `curl` dentro de un ciclo `for` en la terminal de Bash para ejecutar el envío masivo en segundo plano de forma silenciosa:
   
   ```bash
   for i in {1..3600}; do
     curl --path-as-is -s -k -X $'POST' \
     -H $'Host: chl-4bf9bd71-e777-42a6-b434-407e833832d3-votacion.softwareseguro.com.ar' \
     ... [Resto de cabeceras legítimas SIN la cookie "voto"] ... \
     --data-binary $'opUniversidad=1' \
     $'https://chl-4bf9bd71-e777-42a6-b434-407e833832d3-votacion.softwareseguro.com.ar/src/ctl/votacion.ctl.php'
   done
   ```

6. Una vez que el ciclo iterativo devolvió el control de la terminal, se recargó la interfaz web. El sistema validó que la UTN superó los 4000 votos y entregó la flag de resolución del laboratorio.
