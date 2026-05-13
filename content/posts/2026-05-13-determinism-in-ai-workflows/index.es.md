---
title: 'Lo que ya está decidido no se pregunta: determinismo en flujos de trabajo con IA'
date: '2026-05-13'
slug: 'determinism-in-ai-workflows'
draft: false
description: 'Un agente IA es no-determinista por construcción. Para que sea fiable, hay que sacar las decisiones ya tomadas del LLM y cristalizarlas en hooks, scripts y tablas. Lo que queda dentro del modelo es ejecución acotada; lo que decide es el humano.'
tags: ['ai', 'development', 'claude-code', 'workflow']
cover:
  image: '/en/posts/2026-05-13/determinism-in-ai-workflows/cover.jpg'
  alt: 'Deterministic guardrails around an AI agent workflow'
  relative: false
ShowToc: true
TocOpen: false
---

Un agente IA es no-determinista por construcción.

Le pides lo mismo dos veces y te da dos respuestas distintas. La mayoría del tiempo eso no importa: son dos formas igualmente válidas de redactar un commit, dos maneras razonables de explicar un bug. A veces incluso es deseable, porque lo creativo se mueve precisamente en ese margen.

Hasta que llegas a las decisiones que no admiten margen.

"Nunca pushees a master." "Antes de tocar nada, crea un branch + worktree." "Para hablar con GitHub usa el MCP, nunca `curl`." Estas decisiones ya están tomadas (por mí, por el equipo, por alguna herida operacional anterior). No quiero que el agente las reabra en cada sesión.

Y aquí es donde el diseño se vuelve interesante. Porque el instinto, cuando trabajas con un modelo de lenguaje, es escribir la regla con palabras y meterla en el system prompt. "Nunca pushees a master." Listo. Funciona el 95% de las veces.

El 5% restante es un push a master.

## La tentación del prompt

Mientras la regla viva sólo como texto en una policy, sigue siendo una sugerencia. Una sugerencia muy bien argumentada, escrita en mayúsculas, repetida tres veces, con ejemplos. Pero en última instancia, una sugerencia que el modelo evalúa contra todo lo demás que tiene en context.

Un día arrastra demasiados tokens, otro día el ticket dice "urgente" tres veces, otro día sólo hubo una ambigüedad pequeña en la instrucción inicial, y la regla se diluye. No falla en plan dramático, falla en silencio: el agente toma la decisión "rápida" porque, dentro de su lectura del contexto, parecía la más razonable.

Las policies sirven para muchas cosas. Para protecciones absolutas no son suficientes.

## Sacar la decisión del LLM

Lo que sí funciona es **mover la decisión fuera del runtime del modelo**. Cristalizarla en código que no se ejecuta dentro del LLM, sino antes o después. Tres capas, con ejemplos concretos de mi setup:

### 1. Hooks: bloqueo absoluto

Un hook es un script shell que el harness ejecuta antes (o después) de cada tool call. Si devuelve `exit 2`, la llamada se bloquea. El modelo ni siquiera ve la herramienta como ejecutable: el harness le devuelve un error.

```bash
# .claude/hooks/block-push-master.sh
if [ "$BRANCH" = "master" ] || [ "$BRANCH" = "main" ]; then
    echo "BLOCKED: direct push to master/main not allowed."
    exit 2
fi
```

Da igual lo que el modelo crea que es "la mejor solución": la herramienta no se ejecuta. La decisión "nunca master" ha salido del LLM y ahora vive en `bash`, que es determinista por aburrido. Ya hablé de cómo funcionan los hooks en [Claude Code por dentro](/es/posts/2026-04-01/claude-code-internals/), así que no me repito; lo que importa aquí es **dónde vive la regla**.

Y este caso lleva además doble candado: el usuario de GitHub del agente no tiene permiso de push directo a `master` en los repos protegidos. Si el hook local fallara, el remoto rechaza. Doble capa de defensa (ambas capas en dos sitios diferentes y fuera del LLM).

### 2. Scripts: receta fija

No todo es prohibir. Mucho del trabajo de un agente es **ejecutar una secuencia que ya está decidida**.

Cuando se mergea una PR, hay que: tirar de master, borrar el worktree, borrar la branch local y remota, marcar la PR como mergeada en el tracker, cerrar la tarea, marcar resueltos los issues de Sentry referenciados. Siete pasos. Siempre los mismos.

Eso no es trabajo de LLM. Eso es trabajo de `scripts/post-merge-cleanup.sh`. El agente invoca el script; el script hace los siete pasos sin re-decidir nada. Si en lugar de eso le dejas al modelo "hacer la limpieza", lo que vas a obtener es una variante distinta cada vez, alguna de ellas incompleta.

Lo mismo con el bootstrap de un worktree: siete pasos fijados en un protocolo. Existe un subagente con un único trabajo: leer el protocolo, ejecutar los siete pasos, devolver el control. No piensa en alternativas. No las necesita.

### 3. Tablas de routing: qué herramienta para qué

Esta es la capa menos vistosa, y la que más bugs evita.

En el `CLAUDE.md` que carga mi agente al arrancar hay una tabla de seis filas que dice cosas como:

| Acción | Usar | No usar |
|---|---|---|
| Leer una PR privada | `mcp__github__github_pr_read` | `curl`, `gh`, `WebFetch` |
| Crear un Jira | `mcp__jira__jira_create` | `curl`, CLI |

Eso parece documentación. En realidad es **una decisión sacada del runtime del modelo**. Si dejas al agente elegir herramienta caso por caso, antes o después va a meter `curl` en algún momento porque "es más simple". Y `curl` no propaga el error de auth, no graba la operación en logs, no respeta la identidad del agente. La tabla cierra esa decisión de antemano.

## Quién decide qué

Llegados aquí conviene aclarar el reparto, porque es fácil resbalar.

**El humano decide.** La arquitectura, el enfoque ante un trade-off ambiguo, qué problema se ataca primero, cuándo descartar un planteamiento y empezar de nuevo. Eso no se delega a un modelo de lenguaje. No por desconfianza, sino por diseño: el criterio es justamente lo que un agente no debe improvisar en mi nombre.

**El humano cristaliza esas decisiones** en hooks, scripts, tablas, policies y protocolos. Cada vez que una decisión se ha tomado ya, baja del prompt al código.

**El agente ejecuta**, dentro de un perímetro estrechado por todas esas capas. Interpreta el ticket (con margen acotado), escribe el código (con políticas de estilo y tests obligatorios), redacta el commit (con plantilla), abre la PR (con cuerpo estructurado). Lo que queda no es "criterio creativo" del agente; es ejecución dentro de los huecos que el humano deliberadamente ha dejado abiertos.

Es human-in-the-loop, pero con un matiz importante: el humano **no está sólo al final**, revisando lo que el agente produjo. Está **al principio**, decidiendo qué cosas no se pueden mover. El loop empieza con el humano definiendo el perímetro determinista, sigue con el agente trabajando dentro, y se cierra con el humano revisando lo que salió.

## Bonus: el determinismo es barato

Hay un efecto colateral agradable de mover las decisiones fuera del LLM: el modelo barato puede hacer lo determinista.

Los siete pasos del bootstrap de worktree no necesitan a un modelo frontier. Son siete comandos. Un subagente con un modelo más pequeño los ejecuta perfectamente, porque no hay nada que razonar; sólo hay que correr la receta. Lo mismo con el cleanup post-merge, con un smoke test cuyos pasos están escritos, con una traducción de UI.

El modelo caro queda libre para lo que sí lo necesita: el diseño con el humano, el debug con causa desconocida, el self-review antes de pushear.

Esto sólo es posible si las tareas deterministas están de verdad cristalizadas. Si la "receta" en realidad es "más o menos esto, ya verás", ningún modelo, ni grande ni pequeño, te va a dar el mismo resultado dos veces.

## Conclusión

Un agente IA no se vuelve fiable dándole más libertad ni mejores prompts. Se vuelve fiable **achicando el espacio donde tiene que decidir**.

Cada hook que añades es una pregunta que el modelo ya no se hace. Cada script al que invoca es una secuencia que no se reinventa. Cada tabla de routing es una elección de herramienta que sale del razonamiento del momento y entra en el contrato.

Lo que queda dentro del LLM es ejecución acotada. Lo que decide lo decido yo, antes. Y lo cristalizo en cuanto la decisión está tomada, para no tener que volver a tomarla.

Eso es lo que diferencia un agente que "funciona casi siempre" de uno con el que puedes dormir tranquilo.
