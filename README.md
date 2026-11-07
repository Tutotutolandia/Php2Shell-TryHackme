# Informe Técnico CTF: TryHackMe - PHP2Shell

## 1. Objetivo del Laboratorio
El objetivo de este laboratorio es comprender y explotar una cadena de vulnerabilidades críticas que residen en el **motor de ejecución interno de PHP** (y no en el código de la aplicación web). Se busca lograr un salto a la protección de memoria ASLR mediante divulgación de memoria (Memory Disclosure) y, finalmente, obtener la Ejecución Remota de Código (RCE) abusando del manejo de errores durante la subida de archivos (sesiones).

---

## 2. Explicación de la Vulnerabilidad: ¿Qué es y por qué ocurre?

**¿Qué se está explotando?**
Se explota una falla en cómo PHP maneja su memoria interna cuando procesa subidas de archivos mediante la función `session.upload_progress`. Esta función se ejecuta antes de que el script web de la aplicación empiece a correr. 

**¿Por qué ocurre esto?**
El ataque fuerza deliberadamente una condición de "Falta de Memoria" (Out-of-Memory / OOM) en un trabajador (worker) del servidor PHP. 
1. Al quedarse sin memoria, PHP ejecuta un cierre de emergencia o *bailout* (utilizando la función `longjmp()`). 
2. Este cierre brusco omite los procesos normales de limpieza de memoria, dejando un **puntero colgante** (*dangling pointer*): una referencia a un espacio de memoria que ya fue liberado pero sigue siendo apuntado por el sistema.
3. Cuando llega una segunda solicitud maliciosa a ese mismo *worker*, el atacante puede abusar de este estado inconsistente para:
   * **CVE-2026-109618 (Divulgación de Memoria):** Leer fragmentos de la memoria del proceso (que se filtran a través de la cabecera `Set-Cookie: PHPSESSID`) para descubrir en qué direcciones de memoria operan los binarios. Esto derrota la protección del sistema operativo conocida como **ASLR** (Address Space Layout Randomization).
   * **CVE-2026-109619 (RCE):** Con el mapa de memoria expuesto, se sobrescribe un puntero de limpieza interna de PHP para ejecutar comandos del sistema.

---

## 3. Metodología, Ejecución y Evidencias (Desglose por Tareas)

En este escenario, el atacante ya dispone de acceso local a la máquina de salto vía SSH y debe explotar el contenedor Docker vulnerable que corre localmente.

### Tarea 4: Reconocimiento (Confirmando el objetivo)
Antes de ejecutar el ataque, la metodología exige confirmar las versiones y el entorno objetivo. Estando conectados por SSH en la máquina local, procedemos a inspeccionar el servidor web, el cual se ejecuta de forma aislada dentro de un contenedor Docker.

* **Verificación de contenedores activos:**
  ```bash
  docker ps
  ```
  * **Por qué se usa:** Este comando lista los contenedores en ejecución en la máquina. Sirve para identificar cómo se llama nuestro objetivo. 
  * **Explicación para el informe:** Al ejecutarlo, descubrimos un contenedor llamado `php2shell-target` que está exponiendo el servicio en el puerto local `8088`.

* **Comprobación de la versión vulnerable de PHP:**
  ```bash
  docker exec php2shell-target php -v
  ```
  * **Por qué se usa:** El comando `docker exec` nos permite inyectar y ejecutar comandos de terminal directamente *dentro* del contenedor en ejecución. Le pedimos al contenedor que ejecute `php -v`.
  * **Explicación para el informe:** El resultado revela que el contenedor utiliza **PHP 8.4.26**. Esto confirma que el objetivo es vulnerable, ya que esta versión es exactamente una iteración anterior a la versión parchada oficialmente (8.4.27).

* **Validación del código de la aplicación (El punto clave del laboratorio):**
  ```bash
  curl -s http://127.0.0.1:8088/index.php
  ```
  ```bash
  docker exec php2shell-target cat /var/www/html/index.php
  ```
  * **Por qué se usan:** Primero, usamos `curl` para ver qué responde la página web de manera normal (responde "OK"). Luego, usamos nuevamente `docker exec` para utilizar el comando `cat` y leer el código fuente original del archivo `index.php` alojado en `/var/www/html/` dentro del contenedor.
  * **Explicación para el informe:** Al inspeccionar el código fuente, confirmamos que la aplicación consta de una sola línea de código que únicamente imprime la palabra "OK". Esta evidencia demuestra que la aplicación **no contiene código complejo ni fallos de desarrollo clásicos** (como inyecciones SQL o subidas de archivos mal programadas). Toda la vulnerabilidad de Ejecución Remota de Código (RCE) reside intrínsecamente en el motor de ejecución del lenguaje PHP.

* **Ubicación del Exploit:**
  ```bash
  ls ~/php2shell-bypass-aslr-exp
  ```
  * **Por qué se usa:** Sirve como transición entre la fase de reconocimiento y la de explotación. Verifica el directorio de trabajo donde se aloja el *exploit* público en Python y sus recursos auxiliares, listos para ser utilizados.

> **Evidencias Tarea 4:** 
> *[Inserta aquí las capturas de tu consola mostrando la ejecución de `docker ps`, `php -v`, la lectura del `index.php` y el listado `ls` del directorio del exploit]*

### Tareas 5 y 6: Defeating ASLR & Remote Code Execution
Al ejecutar el *script* automatizado proporcionado en el entorno, se desencadena toda la cadena del ataque.

* **Navegación al directorio:** 
  ```bash
  cd ~/php2shell-bypass-aslr-exp
  ```
* **Ejecución del Exploit de prueba:** 
  ```bash
  python3 exploit.py
  ```
  * **Por qué se usa:** El script bombardea al servidor con peticiones pesadas tipo *multipart/form-data* hasta forzar el *bailout* por falta de memoria. Al no colocarle ningún parámetro de comando, el script extrae la memoria filtrada (saltándose el ASLR) y luego lanza por defecto el comando `id` a nivel de sistema operativo.
  * **Resultado:** En la terminal se observa cómo se filtran las direcciones de memoria y finalmente la consola devuelve el output de `uid=33(www-data)`, lo que confirma exitosamente el **RCE**.

* **Obtención de la Flag:** 
  ```bash
  python3 exploit.py "cat /f > out/flag.txt"
  ```
  * **Por qué se usa:** Una vez confirmado el vector de ejecución, le pasamos como argumento un comando específico al script. Debido al pequeño tamaño del *payload* (sólo se disponen de 14 bytes para la inyección de comandos), le ordenamos que lea el contenido del archivo de la flag (ubicado en `/f`) y lo redirija a un directorio compartido (`out/flag.txt`) para poder visualizarlo desde nuestro usuario de bajos privilegios.

> **Evidencias Tareas 5 y 6:** 
> *[Inserta aquí las capturas de pantalla de la terminal mostrando el bypass de ASLR, la ejecución del comando `id` y la flag resultante obtenida del archivo `flag.txt`]*

---

## 4. Impacto de la Vulnerabilidad
El impacto de esta vulnerabilidad es **CRÍTICO (CVSS 9.2+)**. Al ser un error de *runtime* (a nivel del lenguaje), anula todos los controles de seguridad implementados por el desarrollador en su código fuente (como firewalls de aplicación o validaciones lógicas). 
Un atacante remoto y **no autenticado** puede ejecutar comandos arbitrarios en el servidor con los permisos del usuario del servicio web (`www-data`), lo que abre la puerta a robo de bases de datos, despliegue de malware o un compromiso total de la máquina.

---

## 5. Recomendaciones y Mitigación

A diferencia de los típicos ataques a aplicaciones web, corregir esta falla requiere intervenir a nivel de infraestructura y configuración del lenguaje:

1. **Parcheo de Infraestructura (Mitigación Definitiva):**
   * Se debe actualizar el binario del motor de PHP a una versión segura oficial (por ejemplo, PHP 8.4.27, 8.3.36, 8.2.35 o superiores) donde el manejo del *bailout* y limpieza de punteros haya sido corregido. Tras el parche, es obligatorio **reiniciar todos los procesos *workers*** (como PHP-FPM o Apache).

2. **Mitigaciones Provisionales (Workarounds):**
   Si no es posible actualizar de inmediato, se deben aplicar cambios en el archivo `php.ini` para cerrar la vía de entrada:
   * **Deshabilitar upload_progress:** Configurar `session.upload_progress.enabled = 0` (u `Off`). Esto desactiva el seguimiento de estado que dispara la corrupción de memoria.
   * **Bloqueo preventivo en Proxy Inverso/WAF:** Si la aplicación no requiere subir archivos legítimos, se deben bloquear a nivel perimetral todas las solicitudes que utilicen el *Content-Type* `multipart/form-data`. 
   
*Nota: Es inútil intentar mitigar esta falla desde el código PHP de la aplicación o usando WAFs convencionales como módulos de PHP, ya que el ataque ocurre antes de que dichos componentes procesen la solicitud*.
