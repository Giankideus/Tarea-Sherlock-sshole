# Explicación técnica — sshole (CVE-2026-123158)

## 1. Arquitectura y Dependencias

Para que un entorno sea vulnerable a sshole se requiere la coincidencia exacta de tres
condiciones:

- **`musl libc`** como implementación de la librería C del sistema (no `glibc`), ya que
  es la que contiene la función `crypt()` con el comportamiento de error problemático
  ante contraseñas excesivamente largas.
- **OpenSSH `sshd`** en una versión anterior a **10.6p2**, configurado para aceptar
  autenticación por contraseña.
- **PAM deshabilitado** (`UsePAM no`), de modo que `sshd` invoca directamente a
  `crypt()` y compara su resultado contra el campo de hash de `/etc/shadow`, en lugar de
  delegar esa verificación a la pila de PAM.

Los sistemas basados en `glibc` con soporte PAM quedan exentos por dos razones
independientes:

1. Cuando PAM está habilitado, `sshd` no realiza la comparación de hash por sí mismo:
   delega la verificación al módulo `pam_unix`, cuyo flujo interno de validación y
   manejo de errores nunca expone al demonio SSH el valor crudo devuelto por `crypt()`.
2. La implementación de `crypt()` en `glibc` no posee el límite estricto de longitud
   que en `musl` provoca el retorno del carácter de error `*`; por lo tanto, incluso en
   ausencia de PAM, `glibc` no entra en el mismo camino de fallo.

## 2. Mecánica de la Colisión

La cadena de causalidad combina cuatro elementos:

1. **Origen histórico (2016):** para corregir un fallo de *timing* que permitía
   enumerar nombres de usuario válidos (CVE-2016-6210), OpenSSH modificó `sshd` para
   que, ante una cuenta sin contraseña utilizable, igualmente calcule un hash usando un
   salt tomado de una cuenta real "señuelo", de forma que el tiempo de respuesta sea
   indistinguible entre usuarios existentes e inexistentes.
2. **Marcado de cuentas sin contraseña:** las cuentas bloqueadas, deshabilitadas o sin
   password configurado almacenan en `/etc/shadow` un valor centinela como `*`, que por
   diseño nunca puede coincidir con la salida real de una función de hash.
3. **El fallo en `musl`:** cuando `sshd` entrega a `crypt()` una contraseña de entre
   **257 y 1024 caracteres**, la función, en lugar de hashear la cadena o fallar de
   forma segura, retorna directamente el literal `*` como código de error por longitud
   excesiva, descartando el salt que se le había suministrado.
4. **La colisión final:** `sshd` compara el resultado de `crypt()` contra el campo
   `pw_passwd` mediante una comparación de cadenas simple. Si la contraseña enviada por
   el atacante es suficientemente larga, `crypt()` devuelve `*`, que coincide carácter
   por carácter con el `*` ya almacenado para una cuenta sin contraseña real, por lo que
   la comparación resulta verdadera y la autenticación se da por exitosa.

En otras palabras: el demonio sí procesa y revisa la contraseña, pero el código de error
que recibe de vuelta es indistinguible, desde el punto de vista de una comparación de
cadenas, de un hash de cuenta deshabilitada — produciendo un falso positivo de
autenticación sin que exista ningún error de lógica en OpenSSH en sí mismo.

## Referencias

- TryHackMe Blog — OpenSSH CVE-2026-123158: The Password Bug That Lets Attackers Walk Right In.
- Help Net Security — OpenSSH flaw exploited in the wild just hours after disclosure (CVE-2026-123158).
- itnews.com.au — Lightweight Linux distros vulnerable to *sshole password bypass.
- Astra Security — CVE-2026-123158 (*sshole) technical overview.
