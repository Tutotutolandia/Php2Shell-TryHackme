Informe Técnico CTF: TryHackMe - PHP2Shell

1. Objetivo del Laboratorio

El objetivo de este laboratorio es comprender y explotar una cadena de vulnerabilidades críticas que residen en el motor de ejecución interno de PHP (y no en el código de la aplicación web). Se busca lograr un salto a la protección de memoria ASLR mediante divulgación de memoria (Memory Disclosure) y, finalmente, obtener la Ejecución Remota de Código (RCE) abusando del manejo de errores durante la subida de archivos (sesiones).

2. Explicación de la Vulnerabilidad: ¿Qué es y por qué ocurre?

¿Qué se está explotando?
Se explota una falla en cómo PHP maneja su memoria interna cuando procesa subidas de archivos mediante la función session.upload_progress. Esta función se ejecuta internamente antes de que el script web de la aplicación empiece a correr.

¿Por qué ocurre esto?
El ataque fuerza deliberadamente una condición de "Falta de Memoria" (Out-of-Memory / OOM) en un trabajador (worker) del servidor PHP.

Al quedarse sin memoria, PHP ejecuta un cierre de emergencia o bailout (utilizando la función longjmp()).

Este cierre brusco omite los procesos normales de limpieza de memoria, dejando un puntero colgante (dangling pointer): una referencia a un espacio de memoria que ya fue liberado pero sigue siendo apuntado por el sistema.

Cuando llega una segunda solicitud maliciosa a ese mismo worker, el atacante puede abusar de este estado inconsistente para:

Vulnerabilidad de Divulgación de Memoria: Leer fragmentos de la memoria del proceso (que se filtran a través de la cabecera Set-Cookie: PHPSESSID) para descubrir en qué direcciones de memoria operan los binarios. Esto derrota la protección del sistema operativo conocida como ASLR (Address Space Layout Randomization).

Vulnerabilidad de RCE: Con el mapa de memoria expuesto, se sobrescribe un puntero de limpieza interna de PHP para ejecutar comandos del sistema.

3. Metodología, Ejecución y Evidencias (Desglose por Tareas)

En este escenario, asumimos la posición de un atacante que dispone de acceso local a la máquina de salto vía SSH y debe explotar el contenedor Docker vulnerable que corre localmente.

Tarea 4: Reconocimiento (Confirmando el objetivo)

Antes de ejecutar el ataque, la metodología exige confirmar las versiones y el entorno objetivo. Estando conectados por SSH en la máquina local, procedemos a inspeccionar el servidor web, el cual se ejecuta de forma aislada dentro de un contenedor Docker.

Verificación de contenedores activos:

docker ps


Por qué se usa: Este comando lista los contenedores en ejecución en la máquina. Sirve para identificar cómo se llama nuestro objetivo.

Explicación: Al ejecutarlo, descubrimos un contenedor llamado php2shell-target que está exponiendo el servicio en el puerto local 8088.

Comprobación de la versión vulnerable de PHP:

docker exec php2shell-target php -v


Por qué se usa: El comando docker exec nos permite inyectar y ejecutar comandos de terminal directamente dentro del contenedor en ejecución, sin necesidad de conectarnos a él iterativamente. En este caso, le pedimos al contenedor que ejecute php -v.

Explicación: El resultado del comando revela que el contenedor utiliza PHP 8.4.26. Esto confirma que el objetivo es vulnerable, ya que esta versión corresponde a la iteración vulnerable previo al parche oficial.

Validación del código de la aplicación (El punto clave del laboratorio):

curl -s http://127.0.0.1:8088/index.php


docker exec php2shell-target cat /var/www/html/index.php


Por qué se usan: Primero, usamos curl para ver qué responde la página web de manera normal (responde un simple "OK"). Luego, usamos nuevamente docker exec para utilizar el comando cat y leer el código fuente original del archivo index.php alojado en la ruta /var/www/html/ dentro del contenedor.

Explicación: Al inspeccionar el código fuente, confirmamos que la aplicación consta de una sola línea de código que únicamente imprime la palabra "OK" (<?php echo "OK"; ?>). Esta es la evidencia más importante de la fase de reconocimiento: demuestra que la aplicación no contiene código complejo ni fallos de desarrollo clásicos. Toda la vulnerabilidad reside intrínsecamente en el motor de ejecución (runtime) del propio lenguaje PHP.

Ubicación y verificación del Exploit:

ls ~/php2shell-bypass-aslr-exp


Por qué se usa: El comando ls lista los archivos y directorios. En este paso, inspeccionamos el directorio que contiene la herramienta de explotación, ubicado en el directorio home (~) del usuario ubuntu.

Explicación: Esto nos permite confirmar visualmente que el entorno cuenta con el script automatizado (como exploit.py y cualquier recurso auxiliar) necesario para ejecutar el ataque en las fases posteriores, dejando todo listo para la explotación.

Evidencias Tarea 4:
[Inserta aquí la captura de tu consola mostrando la ejecución de docker ps, la salida de la versión vulnerable con php -v, la lectura del archivo index.php con el docker exec y el resultado del ls confirmando el exploit]

Tareas 5 y 6: Defeating ASLR & Remote Code Execution

Al ejecutar el script automatizado proporcionado en el entorno (exploit.py), se desencadena toda la cadena del ataque hacia el contenedor que acabamos de inspeccionar.

Navegación al directorio:

cd ~/php2shell-bypass-aslr-exp


Ejecución del Exploit de prueba (Bypass ASLR + RCE básico):

python3 exploit.py


Por qué se usa: El script bombardea al servidor en el puerto 8088 con peticiones pesadas tipo multipart/form-data hasta forzar el bailout por falta de memoria. Al no colocarle ningún parámetro de comando, el script extrae la memoria filtrada (saltándose el ASLR) y luego lanza por defecto el comando id a nivel de sistema operativo.

Resultado: En la terminal se observa cómo se filtran las direcciones de memoria y finalmente la consola devuelve el output de uid=33(www-data), lo que confirma exitosamente la Ejecución Remota de Código.

Obtención de la Flag:

python3 exploit.py "cat /f > out/flag.txt"


Por qué se usa: Una vez confirmado el vector de ejecución, le pasamos como argumento un comando específico al script. Debido a las limitaciones de memoria (sólo se disponen de unos pocos bytes para la inyección de comandos en este exploit en particular), le ordenamos que lea el contenido del archivo de la flag (ubicado en /f) y lo redirija a un directorio compartido (out/flag.txt) para poder visualizarlo posteriormente desde nuestro usuario.

Evidencias Tareas 5 y 6:
[Inserta aquí las capturas de pantalla de la terminal mostrando el bypass de ASLR, la ejecución del comando id y la flag resultante obtenida]

4. Impacto de la Vulnerabilidad

El impacto de esta vulnerabilidad es CRÍTICO. Al ser un error de runtime (a nivel del lenguaje), anula la gran mayoría de controles de seguridad implementados por el desarrollador en su código fuente (como validaciones lógicas).
Un atacante remoto y no autenticado puede ejecutar comandos arbitrarios en el servidor con los permisos del usuario del servicio web (www-data), lo que abre la puerta a robo de bases de datos, despliegue de malware, exfiltración de secretos o un compromiso total de la máquina hospedadora.

5. Recomendaciones y Mitigación

A diferencia de los típicos ataques a aplicaciones web, corregir esta falla requiere intervenir a nivel de infraestructura y configuración del lenguaje:

Parcheo de Infraestructura (Mitigación Definitiva):

Se debe actualizar el binario del motor de PHP a una versión segura oficial (donde el manejo del bailout y limpieza de punteros haya sido corregido, típicamente la versión inmediatamente superior a la auditada, ej. PHP 8.4.27). Tras el parche, es obligatorio reiniciar todos los procesos workers (como PHP-FPM o Apache).

Mitigaciones Provisionales (Workarounds):
Si no es posible actualizar de inmediato el motor de PHP, se deben aplicar cambios temporales para cerrar la vía de entrada:

Deshabilitar upload_progress: Configurar session.upload_progress.enabled = 0 (u Off) en el archivo php.ini. Esto desactiva el seguimiento de estado que dispara la corrupción de memoria en primer lugar.

Bloqueo preventivo en Proxy Inverso/WAF: Si la aplicación web no requiere subir archivos legítimos para su funcionamiento, se deben bloquear a nivel perimetral todas las solicitudes HTTP que utilicen el encabezado Content-Type: multipart/form-data.

Nota final: Es inútil intentar mitigar esta falla modificando el código fuente de la aplicación web en PHP, ya que el ataque ocurre a nivel del intérprete, antes de que el propio código de la aplicación se ejecute.
