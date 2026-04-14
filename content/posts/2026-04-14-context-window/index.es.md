---
title: 'Context window: cuando tu agente IA empieza a olvidar'
date: '2026-04-14'
slug: 'context-window'
draft: true
description: 'El context window es el recurso más escaso del desarrollo con IA. Cuando tus políticas, protocolos y sesiones crecen, el agente empieza a ignorar instrucciones. Así lo hemos resuelto.'
tags: ['ai', 'development', 'claude-code', 'productivity']
cover:
  image: '/en/posts/2026-04-14/context-window/cover.jpg'
  alt: 'Context window limit in AI-augmented development'
  relative: false
ShowToc: true
TocOpen: false
---

Todo empieza bien.

Le das instrucciones a tu agente IA, las sigue al pie de la letra. Le defines políticas, las cumple. Le pasas protocolos de trabajo, los ejecuta. Es rápido, preciso y no se queja.

Hasta que un día deja de hacerlo.

No da error. No avisa. Simplemente ignora reglas que antes seguía. Comete errores estúpidos. Repite preguntas que ya habías respondido. ¿Qué ha cambiado?

La respuesta es el **context window**, nos metemos en faena y no le prestamos suficiente atención.

## Qué es el context window y por qué es importante

El context window es la cantidad de texto que un modelo de IA puede "tener en mente" en un momento dado. Es su memoria de trabajo. Todo lo que le inyectas — instrucciones del sistema, políticas, ficheros de código, historial de la conversación — compite por el mismo espacio.

El modelo que uso últimamente tiene un context window de un millón de tokens. Suena enorme. Pero cuando trabajas en serio con un agente de IA, descubres que un millón de tokens se llena más rápido de lo que crees.

¿Por qué? Porque no se trata solo de lo que tú escribes. En una sesión real de desarrollo, el agente:

- Lee ficheros de código completos para entender el contexto.
- Ejecuta comandos y recibe su output.
- Lee documentación, tests, configuraciones.
- Acumula todo el historial de la conversación.

Cada una de estas acciones consume tokens. Y cuando se acerca al límite, el sistema empieza a comprimir mensajes anteriores. Es decir: **el agente empieza a olvidar**.

## Lo que inyectamos antes de que el agente diga una sola palabra

En mi setup, el agente arranca cada sesión con un conjunto de ficheros inyectados en el system prompt — las instrucciones fundamentales que debe seguir siempre:

| Fichero                 | Función                                                                                |
|-------------------------|----------------------------------------------------------------------------------------|
| **IDENTITY.md**         | Quién es el agente: nombre, email, GitHub user                                         |
| **CONTACTS.md**         | Su agenda: el Owner, otros agentes, personas relevantes                                |
| **MAIN_RULES.md**       | Las reglas duras: nunca pushear a master, siempre usar worktrees, estados de ejecución |
| **BASE_POLICY.md**      | Política operacional: verdad verificable, seguridad, autonomía, credenciales           |
| **DEVELOPER_POLICY.md** | Política de rol: calidad de código, tracking de tareas, permisos                       |

Además, hay protocolos que se cargan bajo demanda: flujo de desarrollo con Git, workflow de Jira, reglas de CSS, verificación visual...

Todo esto existe porque sin instrucciones explícitas, el agente toma decisiones propias — y no siempre buenas. Las políticas son lo que convierte una IA genérica en un colaborador fiable.

El problema es que cada línea de política consume tokens del context window. Y esos tokens compiten con el código que el agente necesita leer, las herramientas que necesita usar, y la conversación que necesita recordar.

## Cuando las políticas dejaron de funcionar

Hace unos días me encontré con un problema serio: el agente había dejado de seguir reglas fundamentales. Reglas que estaban escritas en su política. Reglas que antes cumplía.

La causa: los ficheros de política habían crecido demasiado. Lo que empezó como documentos concisos se había ido inflando con explicaciones, ejemplos, secciones de contexto histórico, y duplicaciones entre ficheros. El system prompt se había convertido en un muro de texto que, combinado con una sesión larga de trabajo, empujaba el contenido importante fuera de la zona de atención del modelo.

El agente no "decidía" ignorar las reglas. Simplemente no las veía — estaban enterradas bajo tanto texto que el modelo las descartaba al priorizar.

**El síntoma más traidor: no hay error.** El agente sigue respondiendo, sigue trabajando, sigue pareciendo competente. Pero sus decisiones ya no respetan las restricciones que le definiste. Y no te das cuenta hasta que empiezas a desesperarte con los errores.

## La gran reducción

La solución fue drástica pero necesaria: reescribir todas las políticas desde cero con un criterio claro — **cada token debe ganarse su sitio en el context window**.

El resultado en los ficheros que se inyectan en cada sesión:

| Fichero             | Antes            | Después        | Reducción |
|---------------------|------------------|----------------|-----------|
| IDENTITY.md         | 18 líneas        | 6 líneas       | **67%**   |
| CONTACTS.md         | ~27 líneas       | 19 líneas      | **30%**   |
| MAIN_RULES.md       | —                | 41 líneas      | nuevo     |
| BASE_POLICY.md      | 353 líneas       | 50 líneas      | **86%**   |
| DEVELOPER_POLICY.md | 391 líneas       | 47 líneas      | **88%**   |
| **Total**           | **~789 líneas**  | **163 líneas** | **79%**   |

A esto hay que sumar los protocolos que se cargan bajo demanda durante las sesiones de trabajo (Jira, Symfony, verificación visual, etc.), que pasaron de 651 líneas a 302.

En total, el contexto inyectable pasó de ~1.440 líneas a ~465. Ojo, luego he tenido que hacer reajustes, porque en algunas cosas había recortado demasiado.

### Ejemplo concreto: IDENTITY.md

Antes:

```markdown
## Agent

- Display name: Jarvis
- Role: developer
- Execution environment: Claude Code on macOS
- Timezone: Europe/Madrid
- Operational email: jarvis@example.com
- GitHub user: jarvis-aidev
- Code language: English

## Jira Instances

- zeronet
  - URL: xxxxxxxxxxxxxxxxx
  - Key: ZN
  - Credentials: credentials/jira-zeronet.conf
```

Después:

```markdown
# Identity

- Display name: Jarvis
- Role: developer
- Email: jarvis@example.com
- GitHub user: jarvis-aidev
- Code language: English
```

El execution environment, el timezone, las instancias de Jira — nada de eso necesitaba estar en el prompt permanente. El agente ya sabe que corre en macOS. El timezone no afecta sus decisiones. Y las credenciales de Jira se resuelven por otro mecanismo.

### Ejemplo concreto: BASE_POLICY.md

Antes — la sección de control de cambios en políticas:

```markdown
## HARD RULE — Policy file change control

- No automated agent may edit policy files directly on `master`.
- Exception: direct updates to `master` are allowed only when explicitly instructed by the Owner.

Default rule for any policy change (unless the exception above is explicitly invoked):

- Any change to a policy must be made only:
  1) by the Owner manually, OR
  2) via a dedicated branch + Pull Request explicitly requested by the Owner.
- Agents must never push policy changes to `master` (no direct commits, no force-push) unless the Owner explicitly invokes the exception above.
- If the Owner requests a policy change via PR, the agent must:
  - propose the diff,
  - create a PR,
  - and wait for the Owner to review/merge.
- If in doubt, do not modify. Ask.
```

Después — la misma regla en MAIN_RULES.md:

```markdown
## Hard Rules

1. ALL code changes → branch + worktree. No exceptions.
2. Never push master/main.
3. Never merge PRs.
```

Tres líneas. La misma protección. Cero ambigüedad.

## El estilo telegráfico: menos prosa, más instrucción

La clave de la reducción no fue eliminar reglas, sino cambiar el formato. Pasamos de **prosa explicativa** a **estilo telegráfico**: frases cortas, sin artículos innecesarios, sin justificaciones largas.

Una política no es un documento para humanos. Es una instrucción para una máquina. No necesita convencer; necesita ser clara y breve.

Esto es contraintuitivo. Cuando escribes políticas para un agente IA, el instinto natural es ser exhaustivo — explicar el por qué, dar ejemplos, cubrir todos los edge cases. Pero cuanto más texto añades, más diluyes las instrucciones importantes entre ruido.

Hay una variante de esto que se está poniendo de moda, que es el estilo "cavernícola" - sí sí, en serio - que parece que también funciona muy bien, y con una reducción tremenda de tokens. No he tenido tiempo ni ganas de probarlo aún, pero seguro que lo hago.

**Menos texto, más cumplimiento.**

## Sesiones largas: el otro enemigo

Las políticas optimizadas resuelven la mitad del problema. La otra mitad son las sesiones largas.

En una sesión de trabajo intensa, el agente puede leer decenas de ficheros, ejecutar comandos, recibir outputs largos, y acumular cientos de intercambios de mensajes. Todo eso se apila en el context window. Y cuando se acerca al límite, el sistema comprime automáticamente los mensajes más antiguos.

Esa compresión es necesaria, pero tiene un coste: el agente pierde matices. Decisiones que tomasteis juntos al principio de la sesión, acuerdos sobre cómo abordar un problema, contexto específico del task — todo eso se difumina.

Los síntomas:

- **Repite preguntas** que ya habías respondido.
- **Vuelve a leer ficheros** que ya había leído.
- **Contradice decisiones** tomadas antes en la misma sesión.
- **Pierde el tono** — se vuelve más genérico, menos preciso.

La solución no es técnica — es operativa: **aprende a cortar sesiones a tiempo.** Una sesión fresca con contexto limpio rinde más que una sesión larga donde el agente arrastra tokens comprimidos de hace una hora. Pídele que haga un resumen con lo imprescindible para continuar el trabajo sin terminar en una próxima sesión, y que lo deje en un sitio donde pueda recuperarlo.

## Lecciones aprendidas

Después de semanas optimizando el context window de mi agente, estas son las lecciones que me llevo:

**1. El context window es un presupuesto, no un espacio ilimitado.**
Cada token que gastas en políticas es un token menos para código, herramientas y conversación. Gestiona ese presupuesto con la misma disciplina que hace muuuuchos años se gestionaba la memoria RAM al programar.

**2. Si el agente no sigue una regla, el problema puede no ser el agente.**
Antes de culpar al modelo, revisa cuánto texto le estás inyectando. A veces la solución no es repetir la instrucción — es quitar todo lo demás.

**3. Escribe políticas como código, no como documentación.**
Concisas, sin ambigüedad, sin prosa innecesaria. Si una regla necesita un párrafo de explicación, probablemente no es una buena regla.

**4. Las sesiones tienen vida útil.**
No intentes hacer todo en una sola sesión. Corta cuando notes degradación. Una sesión nueva con instrucciones claras es más productiva que una sesión vieja con contexto comprimido.

**5. Mide antes de optimizar.**
Pídele al propio agente que mida el coste en tokens de todos los ficheros inyectados y te liste los mayores consumidores. Es algo que puede hacer él mismo en segundos.

## El context window como restricción de diseño

Trabajar con un agente IA no es solo escribir buenos prompts. Es diseñar un sistema donde la información correcta llega al modelo en el momento correcto, sin saturar el espacio disponible.

El context window no es un detalle técnico que puedas ignorar. Es la restricción fundamental que determina si tu agente es un colaborador fiable o una máquina de generar respuestas genéricas.

Cuanto antes lo optimices, antes dejarás de preguntarte por qué tu agente "ha dejado de funcionar." No ha dejado de funcionar. Ha perdido información relevante que no le das (sin saberlo).
