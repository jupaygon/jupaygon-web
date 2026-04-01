---
title: 'Dentro de Claude Code: 10 cosas interesantes analizando su código fuente'
date: '2026-04-01'
slug: 'claude-code-internals'
draft: false
description: 'El source code de Claude Code se filtró brevemente y alguien hizo un port público. Lo hemos analizado a fondo. Estas son algunas cosas útiles para sacarle más partido como developer.'
tags: ['ai', 'claude-code', 'productivity', 'development']
cover:
  image: '/en/posts/2026-04-01/claude-code-internals/cover.png'
  alt: 'Claude Code internals analysis'
  relative: false
ShowToc: true
TocOpen: false
---

El 31 de marzo de 2026, el código fuente de Claude Code quedó expuesto brevemente. Un developer coreano llamado <a href="https://github.com/instructkr" target="_blank">Sigrid Jin</a> hizo un port clean-room a Python y Rust antes de que Anthropic lo retirara. El repositorio <a href="https://github.com/instructkr/claw-code" target="_blank">claw-code</a> no contiene el código original, pero sí snapshots, metadatos de subsistemas, y un port parcial del runtime que revela cómo funciona Claude Code por dentro.

Entender cómo funciona tu herramienta te permite usarla mejor, así que interesaba revisarlo a fondo. Por razones obvias yo tardaría varios años en revisar todas las líneas de código, pero para eso tenemos a <a href="https://github.com/jarvis-aidev" target="_blank">Jarvis</a>, que se puso manos a la obra y me abstrajo de las primeras 50K horas de trabajo ;-)

Algunas de las cosas que hemos encontrado ya eran más o menos obvias, depende de cada uno hasta dónde haya leído o investigado, así que espero comprensión de los lectores más avanzados. Para el resto, creo que va bien incluirlas en este resumen.

## 1. CLAUDE.md es recursivo: usa una jerarquía de instrucciones

Esto es lo que más me ha cambiado el flujo de trabajo.

Claude Code no busca un único `CLAUDE.md` en la raíz de tu proyecto. Busca archivos de instrucciones **en toda la cadena de directorios ancestros**, desde tu directorio de trabajo actual hasta la raíz del filesystem. En cada directorio busca:

- `CLAUDE.md`
- `CLAUDE.local.md`
- `.claude/CLAUDE.md`
- `.claude/instructions.md`

Todos se cargan y se concatenan (con deduplicación automática si dos archivos tienen el mismo contenido). El budget total es de **12.000 caracteres**, con un máximo de 4.000 por archivo.

**Cómo aprovecharlo:** pon reglas globales en `~/CLAUDE.md` (estilo de código, convenciones de commits, reglas de seguridad), reglas de equipo o empresa en `~/Work/CLAUDE.md`, y reglas específicas en cada repo. Se aplican todas, en cascada.

Y un detalle: `CLAUDE.local.md` está pensado para instrucciones que no quieres commitear. Ponlo en `.gitignore` y úsalo para inyectar contexto personal: paths locales, variables de entorno, preferencias individuales.

Eso sí: no todo tiene que vivir en el CLAUDE.md. En mi caso, el CLAUDE.md apenas tiene unas líneas --- apunta a un repositorio dedicado con policies, protocols y skills organizados en carpetas. Si tu CLAUDE.md empieza a crecer demasiado, plantéatelo. Pero eso da para otro post.

## 2. Los hooks son guardias de seguridad reales, no sugerencias

El system prompt de Claude Code le dice cosas como "no ejecutes comandos destructivos". Pero eso es una instrucción al modelo --- una sugerencia que puede ignorar.

Los hooks son otra cosa. Son **comandos shell reales** que se ejecutan antes y después de cada uso de herramienta. Si un hook devuelve exit code 2, **la herramienta se bloquea**. No hay forma de saltárselo.

El sistema pasa al hook toda la información necesaria como variables de entorno:

- `HOOK_EVENT` --- `PreToolUse` o `PostToolUse`
- `HOOK_TOOL_NAME` --- nombre de la herramienta (ej: `Bash`)
- `HOOK_TOOL_INPUT` --- el input JSON completo
- `HOOK_TOOL_OUTPUT` --- el output (solo en PostToolUse)

**Ejemplo práctico:** un hook que bloquea comandos peligrosos:

```bash
#!/bin/bash
# PreToolUse hook: bloquear comandos destructivos
if echo "$HOOK_TOOL_INPUT" | jq -r '.command // empty' | grep -qE 'rm -rf /|DROP TABLE|format '; then
  echo "Comando peligroso bloqueado por hook"
  exit 2
fi
```

Esto es infinitamente más fiable que confiar en que el modelo "obedezca" las instrucciones del prompt.

## 3. La auto-compactación tiene un umbral configurable

Cuando una sesión se hace larga, Claude Code compacta automáticamente los mensajes antiguos en un resumen estructurado. El resumen preserva: herramientas usadas, archivos clave, trabajo pendiente, y un timeline de la conversación. Los últimos mensajes recientes se mantienen intactos.

Lo que no es obvio es que el umbral se puede ajustar. Por defecto se activa al ~95% de la capacidad del contexto, pero puedes forzar una compactación más agresiva o más conservadora.

**Tip:** para sesiones largas donde el agente "pierde el hilo", compacta manualmente con `/compact`. El resumen que genera es sorprendentemente bueno --- captura el trabajo pendiente, los archivos tocados, y lo que estabas haciendo. Es como un checkpoint de tu sesión.

## 4. El system prompt se construye dinámicamente

El system prompt no es un texto fijo. Se ensambla pieza a pieza por un builder con secciones modulares:

1. **Intro** --- define el rol del agente
2. **Output Style** --- instrucciones de formato (opcional, solo si lo configuras)
3. **System** --- reglas base (permisos, hooks, compresión)
4. **Doing Tasks** --- guía de comportamiento ("no agregues abstracciones especulativas")
5. **Actions** --- principio de precaución para operaciones destructivas
6. **`__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`** --- marca que separa la parte estática de la contextual
7. **Environment** --- modelo, directorio, fecha, SO
8. **Project Context** --- git status + git diff al inicio de sesión
9. **CLAUDE.md** --- tus instrucciones, recursivamente
10. **Runtime Config** --- settings cargados

Ese `DYNAMIC_BOUNDARY` es clave: todo lo que está antes es **cacheable** por la API de Anthropic. Todo lo que está después cambia en cada sesión.

**Implicación práctica:** mantener tu `CLAUDE.md` estable (sin cambios frecuentes) maximiza el cache hit de prompts, lo que reduce latencia y coste. Si cambias tu CLAUDE.md constantemente, invalidas la cache.

## 5. Cada herramienta tiene un nivel de permiso requerido

El sistema de permisos no es un simple on/off. Cada herramienta tiene un `required_permission` asignado:

| Herramienta | Permiso requerido |
|---|---|
| `read_file`, `glob_search`, `grep_search` | ReadOnly |
| `write_file`, `edit_file`, `TodoWrite` | WorkspaceWrite |
| `bash`, `Agent`, `REPL` | DangerFullAccess |

Si tu modo de permisos es menor que el requerido, Claude Code te pregunta antes de ejecutar. Si es igual o mayor, ejecuta directamente.

**Tip:** para tareas de solo lectura (auditorías, exploración de código, documentación), usa el modo `ReadOnly`. El agente no podrá modificar nada aunque quiera. Para automatización CI/CD sin intervención, usa `Allow`.

## 6. Los subagentes admiten override de modelo

Cuando Claude Code lanza un subagente (con la herramienta `Agent`), acepta un parámetro `model` que permite usar un modelo diferente al principal.

Esto ya está integrado en el flujo normal: cuando pides algo que requiere búsqueda, Claude Code puede lanzar un agente Haiku (más rápido y barato) para la búsqueda, y reservar el modelo principal para el razonamiento complejo.

**Cómo aprovecharlo:** si estás en una sesión con Opus y necesitas buscar algo simple, el sistema ya optimiza esto automáticamente. Pero si defines tus propios agentes (agentes custom en `.claude/agents/`), puedes especificar qué modelo usar para cada uno.

## 7. Claude Code captura git status y diff al arrancar

Al inicio de cada sesión, el runtime captura un snapshot de `git status --short --branch` y `git diff` (tanto staged como unstaged). Esto se inyecta en el system prompt como contexto.

**Implicación:** Claude Code sabe desde el primer mensaje qué archivos has cambiado, qué tienes staged, y en qué branch estás. No necesitas explicarle "estoy en la branch feature/X y he modificado estos archivos" --- ya lo sabe.

**Tip:** si quieres que Claude Code parta de un estado limpio, haz commit o stash antes de iniciar la sesión. Si quieres que sepa exactamente qué has tocado, deja los cambios sin commitear.

## 8. Las herramientas "deferred" se cargan bajo demanda

No todas las herramientas están disponibles desde el inicio. Algunas son "deferred" --- aparecen listadas en los `<system-reminder>` pero no tienen schema hasta que se cargan con `ToolSearch`.

Esto es una optimización: cargar todas las herramientas desde el principio consumiría mucho contexto. En su lugar, el modelo ve los nombres disponibles y carga la definición completa solo cuando la necesita.

**Tip:** si necesitas una herramienta específica y Claude Code no parece encontrarla, menciónala por nombre. El sistema usará `ToolSearch` para cargar su schema y hacerla disponible.

## 9. Hay un sistema de skills empaquetados más allá de los visibles

El source revela 20 módulos de skills, varios no evidentes desde la interfaz:

- **`/loop`** --- ejecuta un comando o prompt de forma cíclica con intervalo configurable. Perfecto para polling (ej: vigilar una PR abierta cada 5 minutos por si recibe comentarios o cambia de estado).
- **`/simplify`** --- revisa código cambiado buscando oportunidades de reutilización, calidad y eficiencia. Auto-corrige lo que encuentra.
- **`/schedule`** --- programa agentes remotos con cron. La base para tareas automatizadas recurrentes.
- **`skillify`** --- meta-skill que convierte un patrón de uso frecuente en un nuevo skill reutilizable.

Estos skills no siempre aparecen en la ayuda básica, pero están ahí. Prueba `/loop 5m /status` para monitorizar algo, o `/simplify` después de un refactor grande.

## 10. El sandbox es mucho más que limitar rutas

En Linux, Claude Code usa `bubblewrap` (que internamente crea namespaces de usuario, mount, IPC, PID y UTS con `unshare`). En macOS usa Seatbelt. Puede aislar la red completamente.

Además, detecta automáticamente si está corriendo dentro de Docker, Podman o Kubernetes --- revisando `/.dockerenv`, `/run/.containerenv`, `/proc/1/cgroup` y variables de entorno como `KUBERNETES_SERVICE_HOST`.

**Por qué importa:** si trabajas con datos sensibles o en entornos regulados, el sandbox de Claude Code no es un juguete. Es aislamiento real a nivel de kernel. Y si estás dentro de un container, lo detecta y ajusta su comportamiento en consecuencia.

## Bonus: la magnitud real de Claude Code

Los números del source original dan perspectiva:

- **1.902 archivos** TypeScript
- **207 comandos** slash
- **184 módulos** de herramientas
- **33 subsistemas** independientes
- **130 módulos** solo en el directorio de servicios

Esto no es una CLI que llama a una API. Es un runtime de agente completo, con sistema de plugins, coordinación multi-agente, modo vim, modo voz, un companion animado (sí, hay un buddy con sprites), y subsistemas que aún no se han activado públicamente.

## Conclusiones

Entender cómo funciona tu herramienta te da ventaja. Las tres cosas que más impacto práctico tienen:

1. **CLAUDE.md jerárquico** --- usa la cascada de archivos para tener reglas globales, de equipo y de proyecto. Es la forma más efectiva de controlar el comportamiento del agente.
2. **Hooks como guardias reales** --- si necesitas seguridad real (no sugerencias), los hooks con exit code 2 son la única barrera que el modelo no puede saltar.
3. **Estabilidad del CLAUDE.md** --- mantenerlo estable maximiza la cache de prompts. No lo cambies constantemente.

El source de Claude Code confirma algo que por otra parte ya era obvio: si quieres marcar la diferencia, donde tienes margen de maniobra y mucho que mejorar en tu productividad y en resultados, es en la orquestación. Cómo configuras el entorno, cómo estructuras las instrucciones, cómo conectas las herramientas. Ahí es donde está la ventaja real.