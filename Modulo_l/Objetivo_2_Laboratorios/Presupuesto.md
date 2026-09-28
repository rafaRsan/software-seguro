## Laboratorio: Presupuesto

### Objetivo
Modificar los montos del presupuesto para coincidan con los requisitos solicitados.

### Herramientas utilizadas
Burp Suite (Módulos Proxy y Repeater).

### Paso a paso de la resolución
1. Interceptamos la petición original `GET /api/gastos/` para obtener el archivo JSON que lista los 13 gastos de la base de datos con sus
respectivos IDs y categorías.
2. Al usar el botón "Revisar" de la interfaz, capturamos una petición `POST` hacia `/api/gastos/[ID]/editar/` que enviaba el JSON `{"revisado": true}`. 
3. Para alcanzar los resultados pedidos: promedio general 8000, promedio de esenciales 16375 y promedio de varios 6000,
establecimos las siguientes modificaciones, respetando el mínimo de 500 por ítem:
   * Faltaban 3000 en Esenciales.
   * Faltaban 2000 en Varios.
   * Había que reducir el resto para que sumen exactamente 2500.
4. Enviamos la petición `POST` al módulo Repeater e inyectamos a la fuerza el parámetro `"monto"` no previsto por el frontend.
Ejecutamos las siguientes inyecciones:
   * **(Esenciales) Alquiler:** `POST /api/gastos/2/editar/` -> `{"monto": 13000, "revisado": true}`
   * **(Varios) Marketing:** `POST /api/gastos/7/editar/` -> `{"monto": 12000, "revisado": true}`
   * **(Transporte) Flete:** `POST /api/gastos/1/editar/` -> `{"monto": 1500, "revisado": true}`
   * **(Alimentos) Agua:** `POST /api/gastos/4/editar/` -> `{"monto": 500, "revisado": true}`
   * **(Impuestos) Luz:** `POST /api/gastos/11/editar/` -> `{"monto": 500, "revisado": true}`
5. Apagamos el Intercept en Burp Suite, volvimos a la interfaz gráfica y terminamos de marcar como "Revisado" los ítems restantes.
El servidor comprobó la nueva suma total inyectada y liberó la bandera del desafío.
