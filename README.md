# Write-up: PHP2Shell (TryHackMe)

Write-up personal de la sala [PHP2Shell](https://tryhackme.com/room/cvephp2shell) de
TryHackMe — *"Exploit the PHP2Shell runtime CVEs via session.upload_progress for
unauthenticated RCE"*.

**Estado:** sala completada al 100% (8/8 tareas). Ver [`evidencias/10-room-completed-100.png`](./evidencias/10-room-completed-100.png).

## ⚠️ Aviso ético y de responsabilidad

Este repositorio documenta una actividad realizada en un **laboratorio de práctica
autorizado** (TryHackMe), con fines exclusivamente educativos. La flag de la sala fue
**redactada** en las capturas, conforme a las normas de TryHackMe sobre no publicar flags.
No reproduzcas estas técnicas contra sistemas que no te pertenezcan o para los que no
tengas autorización explícita.

## 📁 Estructura del repositorio

```
.
├── README.md              # Este archivo
├── EVIDENCIA.md            # Evidencias paso a paso (capturas reales)
├── EXPLICACION.md          # Explicación técnica de la vulnerabilidad
├── RECOMENDACIONES.md      # Remediaciones y buenas prácticas
└── evidencias/              # Capturas de pantalla numeradas
```

## 🧩 Estructura de la sala (8 tareas)

| # | Tarea |
|---|---|
| 1 | Introduction |
| 2 | A Harmless Feature: Upload Progress |
| 3 | The Bug: A Bailout at the Wrong Moment |
| 4 | Reconnaissance: Confirming the Target |
| 5 | Leaking Memory and Defeating ASLR (CVE-2026-109618) |
| 6 | From a Leak to a Shell (CVE-2026-109619) |
| 7 | Detection, Mitigation and the Real Fix |
| 8 | Conclusion |

## 🎯 Resumen ejecutivo

La sala gira sobre dos CVEs encadenados en el runtime de PHP relacionados con la
característica `session.upload_progress`:

- **CVE-2026-109618** — fuga de memoria del heap que permite reconstruir direcciones
  base (chunk del heap y de `libphp`), derrotando ASLR.
- **CVE-2026-109619** — a partir de esa fuga, corrupción de memoria que deriva en
  ejecución remota de código no autenticada.

El entorno objetivo era un contenedor Docker (`php2shell-target`, PHP 8.4.26 sobre
Apache) expuesto en `http://127.0.0.1:8088`. Usando el exploit provisto en el laboratorio
(`exploit.py`, dentro de `php2shell-bypass-aslr-exp/`) se encadenaron 3 etapas (leak de
heap, leak de `libphp` y disparo de la ejecución) para lograr ejecución de comandos como
el usuario `www-data`, confirmando RCE no autenticada.

Ver el detalle completo en [`EVIDENCIA.md`](./EVIDENCIA.md) y la explicación técnica en
[`EXPLICACION.md`](./EXPLICACION.md).
