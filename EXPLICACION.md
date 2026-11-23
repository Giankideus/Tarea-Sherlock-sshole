# Explicación técnica — PHP2Shell (session.upload_progress)

## Contexto: ¿qué es `session.upload_progress`?

`session.upload_progress` es una característica del runtime de PHP (no de una aplicación
en particular) pensada para que un formulario pueda mostrarle al usuario el progreso de
la subida de un archivo grande. Cuando está habilitada, PHP crea internamente, durante el
procesamiento de la petición `multipart/form-data`, una entrada de sesión con metadatos
del progreso de la subida (bytes recibidos, nombre de archivo, etc.), antes incluso de que
el desarrollador de la aplicación escriba una sola línea de código para manejarla. Es,
por tanto, una funcionalidad que corre "por debajo" de cualquier aplicación PHP que acepte
subida de archivos — de ahí que la sala la describa como una "funcionalidad inofensiva".

## El bug: una cancelación en el momento equivocado

Según la sala, el problema aparece cuando la subida se **aborta o falla a mitad de
camino** (el equivalente a un "bailout" o cancelación anticipada): el manejo de errores
de esa ruta libera o reutiliza una estructura de memoria en el heap que otra parte del
código todavía cree válida. Esto es la forma clásica de un bug de **use-after-free /
corrupción de heap**: una cancelación prematura deja una referencia "colgando" (dangling)
hacia una zona de memoria que después se reutiliza para otra cosa, sin que el runtime la
invalide correctamente.

Este tipo de fallos en C (lenguaje en que está escrito el intérprete de PHP) no suelen
ser explotables de forma directa y determinista: primero hace falta **información sobre
el layout de memoria del proceso** (direcciones reales de heap y de las librerías
cargadas), porque los sistemas operativos modernos aleatorizan esas direcciones en cada
ejecución (ASLR — *Address Space Layout Randomization*).

## CVE-2026-109618 — Fuga de memoria y evasión de ASLR

Aprovechando el propio mecanismo de `session.upload_progress` (y su interacción con el
bug de cancelación anterior), es posible inducir al servidor a **filtrar fragmentos de
memoria interna** a través de las respuestas HTTP observables por un atacante remoto
(sin necesidad de acceso previo ni credenciales). Repitiendo el envío de peticiones
cuidadosamente formadas ("rotating worker over HTTP" en la evidencia), el exploit:

1. Obtiene la dirección base de un *chunk* del heap.
2. A partir de esa fuga, deriva la dirección base de `libphp` (la librería que implementa
   el runtime de PHP).

Con esas dos direcciones, el atacante conoce exactamente dónde está cargado el proceso en
memoria — **ASLR deja de ser una protección efectiva** para este vector.

## CVE-2026-109619 — De la fuga a ejecución remota de código

Con las direcciones ya conocidas, un segundo encadenamiento de la misma clase de
corrupción de memoria permite calcular offsets concretos (en la evidencia:
`HTDATA` y `ZIF`, es decir, punteros hacia estructuras internas de datos de PHP y hacia
una función interna, *zend internal function*) y **sobrescribir o invocar código**
controlado por el atacante dentro del propio proceso del servidor. El resultado práctico,
demostrado en el laboratorio, es la ejecución de comandos arbitrarios del sistema
operativo con los privilegios del usuario bajo el cual corre PHP/Apache (`www-data`),
**sin necesidad de autenticarse** en la aplicación.

## Por qué es especialmente grave

- **No depende de la aplicación**: afecta a cualquier aplicación PHP que acepte subida de
  archivos, independientemente de qué tan bien esté programada la aplicación en sí.
- **No requiere autenticación**: cualquier cliente HTTP anónimo puede iniciar la cadena.
- **Evade ASLR**, una de las mitigaciones estándar del sistema operativo contra
  explotación de memoria.
- Es una cadena de dos vulnerabilidades (leak + RCE) que combinadas elevan mucho el
  impacto frente a tenerlas por separado.

## Relación con vulnerabilidades reales conocidas

El patrón general (bugs de memoria en el manejo de `session.upload_progress` en el
intérprete de PHP) tiene precedentes públicos reales en el historial de PHP (por ejemplo,
fallos de *use-after-free*/doble liberación reportados a lo largo de los años en esa
misma área del código). La sala construye un escenario didáctico sobre esa misma
superficie de ataque para enseñar la metodología de explotación de memoria paso a paso
(leak → bypass de ASLR → RCE), más que para documentar un CVE público verificable tal
cual con ese identificador.
