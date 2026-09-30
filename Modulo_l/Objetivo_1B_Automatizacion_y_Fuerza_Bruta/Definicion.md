## Analisis de automatizacion y fuerza bruta

# Fuerza bruta
Es un metodo de ataque que busca vulnerar un sistema de forma sistematica y secuencial probando cada
combinacion matematica posible de caracteres (letras, números, símbolos) hasta dar con la 
credencial correcta, es conocida como fuerza bruta por que se basa unicamente en capacidad de computo
para realizar el ataque, si se quiere realizar simplemente se automatiza (o se hace de forma manual)
tipear contraseñas, por ejemplo podria ser tipear toda combinacion de 8 caracteres de las letras del
abecedario y números, esto llevaria mas o menos tiempo segun la capacidad de computo que se tenga.

# Ataque de diccionario
Este ataque seria una version mejorada del anterior metodo, mientras que la fuerza bruta prueba 
aleatoriamente hasta dar con el objetivo, este metodo utiliza una lista predefinida, es decir recopila
contraseñas reales, filtradas o de uso comun para que la herramienta simplente pruebe con cada una 
de ellas, esto optimiza el requerimiento de hardware y el tiempo necesario ya que no es aleatorio sino
con contraseña con una probabilidad mucho mas grande.

# Mecanismos de defensa
Existen varios metodos de defensa para este tipo de atques sin embargo aca nombro 2 que ami parecer
son las mas utiles o las que mas me gustaron:
- **Rate Limiting:** incrementa el tiempo de espera o bloquea temporalmente el envio de solicitudes
tras una determinada cantidad determinada de intentos haciendo inviable un ataque masivo debido al
tiempo que se requeriria.

- **WAF (Web Application Firewall):** coloca un intermediario (como cloudflare) que actua como un escudo
entre el cliente y el servidor web; el WAF analiza el trafico entrante, detecta firmas de herramientas
de automatizacion y bloquea peticiones masivas o maliciosas antes de que lleguen a tocar la logica
de la aplicacion.
