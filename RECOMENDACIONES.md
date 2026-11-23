# Recomendaciones de mitigación y endurecimiento — sshole (CVE-2026-123158)

Más allá de aplicar el parche oficial (**OpenSSH 10.6p2 o superior**), se recomiendan al
menos las siguientes medidas de defensa en profundidad a nivel de configuración:

## a) Deshabilitar la autenticación por contraseña

En `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
KbdInteractiveAuthentication no
```

Esta medida elimina por completo la ruta de código vulnerable: al forzar autenticación
por llave pública, nunca se llega a invocar `crypt()` con una contraseña provista por el
atacante, independientemente de su longitud.

## b) Habilitar y configurar correctamente PAM

```
UsePAM yes
```

Con PAM activo, la verificación de contraseña pasa por el módulo `pam_unix` en lugar de
la comparación interna de `sshd` contra el resultado crudo de `crypt()`, sacando al
sistema del conjunto de condiciones necesarias para la explotación.

## c) Medidas complementarias

- **Auditar `/etc/shadow`** para confirmar que ninguna cuenta privilegiada (p. ej.
  `root` o cuentas de servicio por defecto) termine con un hash real configurado por
  accidente, dado que la condición de explotación exige específicamente un campo `*` o
  similar.
- **Restringir la exposición de red** del servicio SSH (VPN/bastión, listas de control
  de acceso) y desplegar `fail2ban` o un IPS que detecte intentos de autenticación con
  contraseñas anómalamente largas (>256 caracteres), lo cual constituye un indicador de
  compromiso muy específico de este vector.
- **Monitoreo de logs:** alertar sobre patrones de "failed password" seguido
  inmediatamente de "accepted password" para el mismo usuario/origen en pocos segundos
  — tal como se observó en el incidente analizado en [`EVIDENCIA.md`](./EVIDENCIA.md).
