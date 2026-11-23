# Recomendaciones de mitigación

## 1. Parchear / actualizar el runtime de PHP

- Aplicar la versión de PHP que corrige **CVE-2026-109618** y **CVE-2026-109619** tan
  pronto como el proyecto PHP publique el parche (la sala indica que el proyecto PHP
  liberó *fixes* de emergencia el 21 de octubre). Esta es la **corrección real** del
  problema: las mitigaciones de configuración de abajo reducen el riesgo mientras se
  despliega el parche, pero no eliminan el bug de memoria subyacente.
- Mantener un proceso de gestión de parches que permita desplegar *hotfixes* de
  seguridad del runtime en horas, no semanas, dada la severidad (RCE no autenticada).

## 2. Mitigación de configuración mientras se parchea

- Si la aplicación no necesita la barra de progreso de subida de archivos, **deshabilitar
  `session.upload_progress.enabled`** en `php.ini` para eliminar la superficie de ataque
  concreta usada por este exploit.
- Revisar `display_errors` y asegurarse de que esté **desactivado en producción**
  (`display_errors = Off`) — no evita el bug, pero reduce la información que un atacante
  podría obtener durante la explotación.
- Limitar `upload_max_filesize` / `post_max_size` a lo estrictamente necesario para
  reducir la ventana de manipulación de las peticiones `multipart/form-data`.

## 3. Defensa en profundidad a nivel de infraestructura

- Ejecutar el proceso de PHP/Apache con el **mínimo privilegio posible** (usuario
  dedicado, sin acceso a datos sensibles fuera de lo necesario), para limitar el impacto
  si la explotación de memoria tiene éxito.
- Aislar el servicio en contenedores/VMs con *seccomp*, *AppArmor*/*SELinux* y sistemas de
  archivos de solo lectura donde sea posible, de forma que incluso con RCE el atacante
  tenga un radio de acción reducido.
- Desplegar un **WAF** con reglas que detecten patrones anómalos de peticiones
  `multipart/form-data` repetidas con el mismo identificador de progreso
  (`PHP_SESSION_UPLOAD_PROGRESS`), típicas de intentos de *heap spraying* o *leak*.

## 4. Detección

- Monitorear logs de Apache/PHP-FPM en busca de ráfagas de peticiones de subida
  incompletas o abortadas hacia el mismo endpoint en corto periodo de tiempo.
- Alertar ante procesos hijos inesperados generados por el proceso de PHP/Apache
  (indicativo de ejecución de comandos del sistema operativo tras una RCE lograda).
- Incluir reglas de EDR/HIDS que detecten ejecución de intérpretes de comandos
  (`sh`, `bash`) como proceso hijo directo de `php-fpm`/`apache2`/`httpd`.

## 5. Proceso

- Incluir el runtime de PHP (no solo las dependencias de la aplicación vía Composer) en
  el inventario de componentes a los que se les da seguimiento de CVEs.
- Suscribirse a los avisos de seguridad oficiales del proyecto PHP para reaccionar a
  *fixes* de emergencia como el descrito en esta sala.

## Referencias

- PHP: https://www.php.net/ (avisos de seguridad oficiales)
- OWASP — Using Components with Known Vulnerabilities:
  https://owasp.org/www-project-top-ten/
