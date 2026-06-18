---
title: 'Loop engineering: diseñar el sistema que promptea al agente'
date: '2026-06-17'
slug: 'loop-engineering'
draft: false
description: 'Addy Osmani y Boris Cherny le han puesto nombre: dejas de promptear al agente y empiezas a diseñar el bucle que lo promptea. Aquí están sus seis piezas, que llevo meses probando en mi propio setup, y también algo de factura crítica.'
tags: ['ai', 'development', 'claude-code', 'workflow', 'agents']
cover:
  image: '/en/posts/2026-06-17/loop-engineering/cover.jpg'
  alt: 'Un bucle autónomo que descubre, ejecuta y verifica trabajo alrededor de un punto de control humano'
  relative: false
ShowToc: true
TocOpen: false
---

En junio de 2026, Addy Osmani y Boris Cherny le han puesto nombre a algo que no es nuevo (considerando las velocidades actuales de novedades): **loop engineering**. Osmani es ingeniero en Google Chrome y escribe mucho sobre desarrollo; Cherny es uno de los creadores de Claude Code. Le dieron forma en [un hilo en X](https://x.com/addyosmani/status/2064127981161959567) y un par de ensayos.

La frase que lo resume es de Cherny: "Ya no prompteo a Claude. Tengo bucles corriendo que promptean a Claude." El cambio se dice en una línea y se vive en grande. Dejas de ser quien sostiene la herramienta turno a turno y empiezas a diseñar el sistema que encuentra el trabajo, lo reparte, lo revisa, anota lo que está hecho y decide lo siguiente.

Ellos hicieron esa parte difícil: ver la forma con claridad suficiente para nombrarla. Llevo meses probando esas seis piezas en mi propio setup. Así que este post es dos cosas a la vez: un mapa de las piezas que describen, cada una conectada a mi setup real, y una nota honesta sobre la factura.

## Un piso por encima del harness

Ya [escribí antes](/es/posts/2026-05-13/determinism-in-ai-workflows/) sobre sacar decisiones del modelo de lenguaje y cristalizarlas en hooks, scripts y tablas de routing. Eso es el harness: el perímetro determinista dentro del que corre un agente.

Loop engineering vive un piso por encima. El harness responde a "cómo se comporta un agente de forma segura". El bucle responde a "quién arranca al agente, cuántos corren a la vez, qué pasa cuando uno termina y cómo se elige el siguiente trabajo". El harness son las barreras de seguridad. El bucle es lo que conduce por la carretera.

Osmani parte el bucle en seis piezas. Veámoslas una por una.

## 1. Automatizaciones: el latido

Un bucle necesita algo que dispare sin ti. Tareas programadas y vigilantes que descubren trabajo y lo sacan a la superficie en vez de esperar a que tú escribas.

En mi caso son dos capas. Los hooks reaccionan a eventos (una PR abierta lanza un watcher que sondea hasta que CI está en verde o llega una review). Las ejecuciones programadas van a buscar trabajo con una cadencia: coger la siguiente fila sin marcar del roadmap, abrir la tarea trackeada, arrancar la rama. La clave es que el primer prompt de una sesión no siempre es mío. Muchas veces lo escribió el sistema.

Esta es la pieza que parece magia la primera semana y un pasivo la tercera, por razones a las que llego enseguida.

## 2. Worktrees: paralelo sin colisiones

En cuanto hay más de un agente vivo, se pelean por los ficheros. Los worktrees de Git lo resuelven limpio: cada agente tiene su propio checkout en su propia rama, compartiendo historia, sin pisarse el árbol de trabajo.

Debe ser regla dura en el setup: nada se toca en `master`, toda tarea arranca con rama y worktree, y el bootstrap que lo prepara es [una receta fija](/es/posts/2026-05-13/determinism-in-ai-workflows/) que ejecuta un subagente barato. El paralelismo deja de dar miedo cuando el aislamiento es el valor por defecto y no una ocurrencia posterior.

## 3. Skills: la intención que sobrevive al reset

Una skill es una carpeta con un `SKILL.md` que captura una convención, un paso de build, una pieza de conocimiento de dominio que el agente, si no, vuelve a deducir (mal) en cada ciclo. Osmani llama a lo que evitan "intent debt": el coste de que el modelo redescubra, una y otra vez, lo que tú ya decidiste.

Tengo un conjunto versionado de ellas. Una fija la superficie de API de un framework de administración que rompió la mitad de sus métodos entre versiones mayores, para que el agente deje de adivinar. Otra codifica un checklist de SEO técnico. Viven en el repo del agente, revisadas como código, porque [eso es lo que son](/es/posts/2026-03-29/ai-skills-easyadmin5/): intención ejecutable, no documentación que se pudre en una wiki.

## 4. Conectores: el agente actúa, no sugiere

Sin conectores, un bucle es un becario muy elocuente que sólo sabe hablar. Con ellos lee una PR, la comenta, transiciona un ticket, resuelve una alerta, abre una incidencia. Los servidores MCP son el cableado: GitHub, el gestor de tareas, el monitor de errores, el tablero de proyecto.

Es la diferencia entre "esto es lo que haría" y "hecho, aquí tienes el enlace". Y es donde una [tabla de routing](/es/posts/2026-05-13/determinism-in-ai-workflows/) aporta más valor: el bucle tiene que llegar a una PR privada por el conector de GitHub, nunca por `curl`, o pierde identidad, logging y manejo de auth en un solo atajo.

## 5. Sub-agentes: separación de responsabilidades

Un agente explora, otro implementa, otro verifica. Corren con instrucciones distintas y, a menudo, con niveles de modelo distintos. El modelo barato ejecuta las recetas deterministas; el caro se reserva para el diseño y para la review antes de un push.

La razón más afilada para separarlos es de Osmani, y merece citarla literal: "el modelo que escribió el código es demasiado blando corrigiendo sus propios deberes." Un agente nuevo, sin nada invertido en la respuesta anterior, es mucho mejor crítico que el que la defiende. Por eso el auditor no es el autor. El verificador lee el diff en frío.

## 6. Estado externo: memoria en disco

El modelo olvida todo entre ejecuciones. Así que la memoria no puede vivir en la ventana de contexto: tiene que vivir en disco. Ficheros markdown, un tablero de tareas, un tracker. El bucle lee su estado al empezar una ejecución y lo vuelve a escribir al terminar.

Esta es la columna vertebral silenciosa. El roadmap que dice qué está hecho y qué viene, la tarea trackeada que sobrevive a un crash, la auto-memoria que recuerda un quirk que expliqué hace tres semanas. Me [importa mucho qué entra en la ventana de contexto](/es/posts/2026-04-14/context-window/); el corolario es que me importa igual qué vive a salvo fuera de ella.

## La factura

Aquí está la parte que no cabe en un hilo de lanzamiento.

Un bucle es un amplificador, y a los amplificadores les da igual el signo de la señal. Apúntalo a una buena práctica y compone buena práctica. Apúntalo a una suposición endeble y compone el error, en paralelo, mientras duermes. La misma maquinaria que entrega diez PRs correctas por la noche entregará diez equivocadas con la misma confianza y los mismos checks en verde.

El coste más sutil es lo que Osmani llama comprehension debt. Un bucle te deja correr en trabajo que entiendes a fondo, y te deja correr exactamente igual en trabajo que has dejado de entender por completo. Desde fuera los dos casos son idénticos: los tickets se cierran, el tablero se pone verde. La diferencia sólo aparece el día que algo se rompe y descubres que no sabes razonar sobre un sistema que tus bucles construyeron sin ti.

Por eso el humano no sale del bucle. El humano se mueve a sus dos extremos. Al principio, decidiendo lo que no se mueve: la arquitectura, el perímetro, qué problema merece siquiera un bucle. Al final, la verificación: no "¿pasó CI?" sino "¿entiendo lo que cambió y estoy dispuesto a ponerle mi nombre?". Ese segundo punto de control es el cuello de botella real de todo esto, y la única pieza que no debes automatizar, porque en el momento en que lo haces, el bucle ya no te hace más rápido. Te convierte en espectador de tu propio código.

## Cierre

Loop engineering es real, y es un salto genuino respecto a promptear turno a turno. Lo tengo corriendo desde hace meses y no volvería atrás.

Pero la palanca corta por los dos lados. Un bucle bien diseñado multiplica a un programador cuidadoso. También multiplica a uno descuidado, más rápido. El trabajo de diseño, la parte que de verdad es ingeniería, no es cablear las seis piezas. Es decidir qué se le permite hacer al bucle por su cuenta, y conservar el único punto de control que sigue siendo tuyo: entregar código que verificaste que funciona, y que entiendes.
