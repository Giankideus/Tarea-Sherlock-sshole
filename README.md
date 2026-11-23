# Análisis de la vulnerabilidad sshole (CVE-2026-123158)

Write-up de **Giancarlo Guarda** — bypass de autenticación en OpenSSH sobre `musl libc`
sin PAM — fundamentos teóricos y análisis forense del reto [Sherlock: sshole](https://tryhackme.com)
de TryHackMe.

> **Estado:** Sherlock resuelto. Ver [`evidencias/12-sherlock-resuelto.png`](./evidencias/12-sherlock-resuelto.png).

## ⚠️ Aviso ético

Este repositorio documenta una actividad académica/de laboratorio realizada en un entorno
autorizado (TryHackMe Sherlock). Tiene fines exclusivamente educativos. No reproduzcas
estas técnicas contra sistemas que no te pertenezcan o para los que no tengas
autorización explícita.

## 📁 Estructura del repositorio

```
.
├── README.md                                   # Este archivo
├── Informe_sshole_Giancarlo_Guarda.docx         # Informe completo en Word (entrega original)
├── EXPLICACION.md                               # Preguntas teóricas 1 y 2
├── RECOMENDACIONES.md                           # Pregunta teórica 3 (mitigación)
├── EVIDENCIA.md                                 # Análisis forense (pregunta 4) con capturas
└── evidencias/                                   # Capturas numeradas usadas como evidencia
```

## 🎯 Resumen ejecutivo

**CVE-2026-123158 ("sshole")** es un bypass de autenticación en OpenSSH `sshd` sobre
sistemas Linux que usan `musl libc` sin PAM (p. ej. Alpine Linux). Una contraseña de
entre 257 y 1024 caracteres hace que `crypt()` de `musl` devuelva el literal `*` como
código de error, el cual colisiona con el valor centinela `*` que usan las cuentas sin
contraseña real, produciendo un falso positivo de autenticación. Divulgada el 5 de
noviembre de 2026 (CVSS 3.1: 8.1), corregida en OpenSSH 10.6p2.

En el laboratorio Sherlock se investigó un host Alpine Linux (`dev-jumpbox`)
comprometido mediante esta vulnerabilidad: acceso inicial sobre `root` y luego sobre la
cuenta `adminsvc`, seguido de reconocimiento de red interna, descarga y ejecución de un
script remoto, exfiltración de datos hacia un dominio externo, y borrado antiforense de
evidencia por parte del atacante.

## 📄 Contenido

- [`EXPLICACION.md`](./EXPLICACION.md) — arquitectura/dependencias necesarias y mecánica
  técnica de la colisión en `crypt()`.
- [`RECOMENDACIONES.md`](./RECOMENDACIONES.md) — mitigación y endurecimiento más allá del
  parche oficial.
- [`EVIDENCIA.md`](./EVIDENCIA.md) — análisis forense completo del incidente simulado,
  con línea de tiempo y capturas reales del Sherlock.
- [`Informe_sshole_Giancarlo_Guarda.docx`](./Informe_sshole_Giancarlo_Guarda.docx) —
  versión Word del informe completo, lista para entrega.
