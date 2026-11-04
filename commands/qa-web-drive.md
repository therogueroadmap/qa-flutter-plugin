---
description: Ejecuta tests E2E de Playwright en una app Flutter web vía el HTML renderer. Genera el spec si no existe, lo corre como regresión si existe. Opcionalmente consume un plan vía --plan=<path>.
argument-hint: "<feature-name> [--auto] [--plan=<path>] [--headed] [--no-bootstrap] [--skip-preflight-probe]"
---

Invocar el skill `qa-flutter:qa-flutter-web-driver-runner` con argumentos: $ARGUMENTS

Seguir SKILL.md exactamente — empezar por pre-flight checks (Step 3), sin atajos.

Si `$ARGUMENTS` está vacío o vale `regresion`, correr la suite completa de regresión. Si no, tratar el argumento como nombre de feature (buscar en `qa-plugin-config/web-tests/`; generar si falta).

Si `--plan=<path>` está presente, el skill lee el plan (generado por `qa-flutter-test-planner` / `/qa-plan`) y usa sus secciones **Pantallas** y **Flujos** en lugar de hacer resolución semántica. Ver Section C.0 del SKILL.md.

**Flags v1.5+**:
- `--no-bootstrap` — el driver NO invoca `qa-flutter-bootstrap` automáticamente (asume server ya levantado a mano). Útil para debug iterativo donde levantas el server una vez y corres varios runs del driver.
- `--skip-preflight-probe` — saltea el a11y probe (~5-8s) del Step 3e. Recomendado solo si la app ya está validada y quieres ahorrar tiempo. El primer test del usuario va a fallar con timeout genérico si la app no llama `ensureSemantics()`.
