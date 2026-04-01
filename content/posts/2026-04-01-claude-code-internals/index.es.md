---
title: 'Dentro de Claude Code: 10 cosas que descubri analizando su codigo fuente'
date: '2026-04-01'
slug: 'claude-code-internals'
draft: false
description: 'El source code de Claude Code se filtro brevemente y alguien hizo un port publico. Lo analice a fondo. Estas son las cosas mas utiles que aprendi para sacarle mas partido como developer.'
tags: ['ai', 'claude-code', 'productivity', 'development']
cover:
  image: 'cover.jpg'
  alt: 'Claude Code internals analysis'
  relative: true
ShowToc: true
TocOpen: false
---

El 31 de marzo de 2026, el codigo fuente de Claude Code quedo expuesto brevemente. Un developer coreano llamado <a href="https://github.com/instructkr" target="_blank">Sigrid Jin</a> hizo un port clean-room a Python y Rust antes de que Anthropic lo retirara. El repositorio <a href="https://github.com/instructkr/claw-code" target="_blank">claw-code</a> no contiene el codigo original, pero si snapshots, metadatos de subsistemas, y un port parcial del runtime que revela como funciona Claude Code por dentro.

Lo analice a fondo. No por curiosidad --- sino porque entender como funciona tu herramienta te permite usarla mejor. Aqui van las 10 cosas mas utiles que aprendi.

## 1. CLAUDE.md es recursivo: usa una jerarquia de instrucciones

Esto es lo que mas me ha cambiado el flujo de trabajo.

Claude Code no busca un unico `CLAUDE.md` en la raiz de tu proyecto. Busca archivos de instrucciones **en toda la cadena de directorios ancestros**, desde tu directorio de trabajo actual hasta la raiz del filesystem. En cada directorio busca:

- `CLAUDE.md`
- `CLAUDE.local.md`
- `.claude/CLAUDE.md`
- `.claude/instructions.md`

Todos se cargan y se concatenan (con deduplicacion automatica si dos archivos tienen el mismo contenido). El budget total es de **12.000 caracteres**, con un maximo de 4.000 por archivo.

**Como aprovecharlo:** pon reglas globales en `~/CLAUDE.md` (estilo de codigo, convenciones de commits, reglas de seguridad), reglas de equipo o empresa en `~/Work/CLAUDE.md`, y reglas especificas en cada repo. Se aplican todas, en cascada.

Y un detalle: `CLAUDE.local.md` esta pensado para instrucciones que no quieres commitear. Ponlo en `.gitignore` y usalo para inyectar contexto personal: paths locales, variables de entorno, preferencias individuales.

## 2. Los hooks son guardias de seguridad reales, no sugerencias

El system prompt de Claude Code le dice cosas como "no ejecutes comandos destructivos". Pero eso es una instruccion al modelo --- una sugerencia que puede ignorar.

Los hooks son otra cosa. Son **comandos shell reales** que se ejecutan antes y despues de cada uso de herramienta. Si un hook devuelve exit code 2, **la herramienta se bloquea**. No hay forma de saltarselo.

El sistema pasa al hook toda la informacion necesaria como variables de entorno:

- `HOOK_EVENT` --- `PreToolUse` o `PostToolUse`
- `HOOK_TOOL_NAME` --- nombre de la herramienta (ej: `Bash`)
- `HOOK_TOOL_INPUT` --- el input JSON completo
- `HOOK_TOOL_OUTPUT` --- el output (solo en PostToolUse)

**Ejemplo practico:** un hook que bloquea comandos peligrosos:

```bash
#!/bin/bash
# PreToolUse hook: bloquear comandos destructivos
if echo "$HOOK_TOOL_INPUT" | jq -r '.command // empty' | grep -qE 'rm -rf /|DROP TABLE|format '; then
  echo "Comando peligroso bloqueado por hook"
  exit 2
fi
```

Esto es infinitamente mas fiable que confiar en que el modelo "obedezca" las instrucciones del prompt.

## 3. La auto-compactacion tiene un umbral configurable

Cuando una sesion se hace larga, Claude Code compacta automaticamente los mensajes antiguos en un resumen estructurado. El resumen preserva: herramientas usadas, archivos clave, trabajo pendiente, y un timeline de la conversacion. Los ultimos mensajes recientes se mantienen intactos.

Lo que no es obvio es que el umbral se puede ajustar. Por defecto se activa al ~95% de la capacidad del contexto, pero puedes forzar una compactacion mas agresiva o mas conservadora.

**Tip:** para sesiones largas donde el agente "pierde el hilo", compacta manualmente con `/compact`. El resumen que genera es sorprendentemente bueno --- captura el trabajo pendiente, los archivos tocados, y lo que estabas haciendo. Es como un checkpoint de tu sesion.

## 4. El system prompt se construye dinamicamente

El system prompt no es un texto fijo. Se ensambla pieza a pieza por un builder con secciones modulares:

1. **Intro** --- define el rol del agente
2. **Output Style** --- instrucciones de formato (opcional, solo si lo configuras)
3. **System** --- reglas base (permisos, hooks, compresion)
4. **Doing Tasks** --- guia de comportamiento ("no agregues abstracciones especulativas")
5. **Actions** --- principio de precaucion para operaciones destructivas
6. **`__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`** --- marca que separa la parte estatica de la contextual
7. **Environment** --- modelo, directorio, fecha, SO
8. **Project Context** --- git status + git diff al inicio de sesion
9. **CLAUDE.md** --- tus instrucciones, recursivamente
10. **Runtime Config** --- settings cargados

Ese `DYNAMIC_BOUNDARY` es clave: todo lo que esta antes es **cacheable** por la API de Anthropic. Todo lo que esta despues cambia en cada sesion.

**Implicacion practica:** mantener tu `CLAUDE.md` estable (sin cambios frecuentes) maximiza el cache hit de prompts, lo que reduce latencia y coste. Si cambias tu CLAUDE.md constantemente, invalidas la cache.

## 5. Cada herramienta tiene un nivel de permiso requerido

El sistema de permisos no es un simple on/off. Cada herramienta tiene un `required_permission` asignado:

| Herramienta | Permiso requerido |
|---|---|
| `read_file`, `glob_search`, `grep_search` | ReadOnly |
| `write_file`, `edit_file`, `TodoWrite` | WorkspaceWrite |
| `bash`, `Agent`, `REPL` | DangerFullAccess |

Si tu modo de permisos es menor que el requerido, Claude Code te pregunta antes de ejecutar. Si es igual o mayor, ejecuta directamente.

**Tip:** para tareas de solo lectura (auditorias, exploracion de codigo, documentacion), usa el modo `ReadOnly`. El agente no podra modificar nada aunque quiera. Para automatizacion CI/CD sin intervencion, usa `Allow`.

## 6. Los subagentes admiten override de modelo

Cuando Claude Code lanza un subagente (con la herramienta `Agent`), acepta un parametro `model` que permite usar un modelo diferente al principal.

Esto ya esta integrado en el flujo normal: cuando pides algo que requiere busqueda, Claude Code puede lanzar un agente Haiku (mas rapido y barato) para la busqueda, y reservar el modelo principal para el razonamiento complejo.

**Como aprovecharlo:** si estas en una sesion con Opus y necesitas buscar algo simple, el sistema ya optimiza esto automaticamente. Pero si defines tus propios agentes (agentes custom en `.claude/agents/`), puedes especificar que modelo usar para cada uno.

## 7. Claude Code captura git status y diff al arrancar

Al inicio de cada sesion, el runtime captura un snapshot de `git status --short --branch` y `git diff` (tanto staged como unstaged). Esto se inyecta en el system prompt como contexto.

**Implicacion:** Claude Code sabe desde el primer mensaje que archivos has cambiado, que tienes staged, y en que branch estas. No necesitas explicarle "estoy en la branch feature/X y he modificado estos archivos" --- ya lo sabe.

**Tip:** si quieres que Claude Code parta de un estado limpio, haz commit o stash antes de iniciar la sesion. Si quieres que sepa exactamente que has tocado, deja los cambios sin commitear.

## 8. Las herramientas "deferred" se cargan bajo demanda

No todas las herramientas estan disponibles desde el inicio. Algunas son "deferred" --- aparecen listadas en los `<system-reminder>` pero no tienen schema hasta que se cargan con `ToolSearch`.

Esto es una optimizacion: cargar todas las herramientas desde el principio consumiria mucho contexto. En su lugar, el modelo ve los nombres disponibles y carga la definicion completa solo cuando la necesita.

**Tip:** si necesitas una herramienta especifica y Claude Code no parece encontrarla, mencionala por nombre. El sistema usara `ToolSearch` para cargar su schema y hacerla disponible.

## 9. Hay un sistema de skills empaquetados mas alla de los visibles

El source revela 20 modulos de skills, varios no evidentes desde la interfaz:

- **`/loop`** --- ejecuta un comando o prompt de forma ciclica con intervalo configurable. Perfecto para polling (ej: vigilar el estado de un deploy cada 5 minutos).
- **`/simplify`** --- revisa codigo cambiado buscando oportunidades de reutilizacion, calidad y eficiencia. Auto-corrige lo que encuentra.
- **`/schedule`** --- programa agentes remotos con cron. La base para tareas automatizadas recurrentes.
- **`skillify`** --- meta-skill que convierte un patron de uso frecuente en un nuevo skill reutilizable.

Estos skills no siempre aparecen en la ayuda basica, pero estan ahi. Prueba `/loop 5m /status` para monitorizar algo, o `/simplify` despues de un refactor grande.

## 10. El sandbox es mucho mas que limitar rutas

En Linux, Claude Code usa `bubblewrap` (que internamente crea namespaces de usuario, mount, IPC, PID y UTS con `unshare`). En macOS usa Seatbelt. Puede aislar la red completamente.

Ademas, detecta automaticamente si esta corriendo dentro de Docker, Podman o Kubernetes --- revisando `/.dockerenv`, `/run/.containerenv`, `/proc/1/cgroup` y variables de entorno como `KUBERNETES_SERVICE_HOST`.

**Por que importa:** si trabajas con datos sensibles o en entornos regulados, el sandbox de Claude Code no es un juguete. Es aislamiento real a nivel de kernel. Y si estas dentro de un container, lo detecta y ajusta su comportamiento en consecuencia.

## Bonus: la magnitud real de Claude Code

Los numeros del source original dan perspectiva:

- **1.902 archivos** TypeScript
- **207 comandos** slash
- **184 modulos** de herramientas
- **33 subsistemas** independientes
- **130 modulos** solo en el directorio de servicios

Esto no es una CLI que llama a una API. Es un runtime de agente completo, con sistema de plugins, coordinacion multi-agente, modo vim, modo voz, un companion animado (si, hay un buddy con sprites), y subsistemas que aun no se han activado publicamente.

## Conclusiones

Entender como funciona tu herramienta te da ventaja. Las tres cosas que mas impacto practico tienen:

1. **CLAUDE.md jerarquico** --- usa la cascada de archivos para tener reglas globales, de equipo y de proyecto. Es la forma mas efectiva de controlar el comportamiento del agente.
2. **Hooks como guardias reales** --- si necesitas seguridad real (no sugerencias), los hooks con exit code 2 son la unica barrera que el modelo no puede saltar.
3. **Estabilidad del CLAUDE.md** --- mantenerlo estable maximiza la cache de prompts. No lo cambies constantemente.

El source de Claude Code confirma algo que sospechaba: la diferencia entre usarlo bien y usarlo regular no esta en los prompts que le escribes, sino en como configuras el entorno que lo rodea.
