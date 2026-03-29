---
title: 'AI-Augmented Development: mi experiencia real trabajando con un agente IA'
date: '2026-03-26'
draft: false
description: 'Llevo meses trabajando con un agente IA como colaborador de desarrollo. No es autocompletado. Es otro developer en el equipo. Esto es lo que he aprendido.'
tags: ['ai', 'development', 'claude-code', 'productivity']
cover:
  image: 'cover.jpg'
  alt: 'Developer working with an AI agent'
  relative: true
ShowToc: true
TocOpen: false
---

Hay mucho ruido sobre IA y programación. Copilots, autocompletados, generadores de código... La mayoría de herramientas se quedan en sugerir la siguiente línea. Pero se puede hacer algo diferente, algo más: trabajar con un agente IA que opera como **otro developer en el equipo**.

Mi compañero digital se llama Jarvis. Tiene su propia cuenta de GitHub, su propio acceso a Jira, y un flujo de trabajo idéntico al de cualquier desarrollador humano: branch, implementación, commit, pull request, code review. Llevo meses con este sistema en producción, y esto es lo que he aprendido.

## Qué es AI-Augmented Development (y qué no es)

No es pedirle a una IA que te escriba una función. No es autocompletado inteligente. No es copiar y pegar de ChatGPT.

AI-Augmented Development es integrar un agente IA en tu flujo de trabajo real, con las mismas herramientas, las mismas restricciones y el mismo proceso que un developer humano:

- **Branch aislado** — cada tarea en su propio branch, nunca en master.
- **Pull request** — todo el trabajo pasa por PR. Yo reviso, comento, pido cambios.
- **Permisos limitados** — el agente no puede mergear ni pushear a master. Las branch protection rules lo impiden.
- **Trazabilidad** — cada commit está asociado a su autor. El git log muestra quién hizo qué.

La diferencia con un copilot es que Jarvis entiende el contexto completo del proyecto, no solo el archivo que tienes abierto. Aporta velocidad y menores curvas de aprendizaje, siempre como un complemento que te hace mejor programador, no como un sustituto.

Y un matiz importante: Jarvis no es simplemente abrir una sesión de Claude Code o Codex y pedirle que haga cosas. Lleva meses de iteración. Políticas que definen qué puede y qué no puede hacer, procedimientos para cada tipo de tarea, herramientas conectadas (GitHub, Jira, MCP servers), reglas de seguridad, convenciones de código. Todo eso es prompt engineering estructurado, y es lo que convierte una sesión de chat en algo viable para trabajo real.

La ventaja: pídele a tu propio agente que te ayude a perfeccionarlo. Cuando algo no funcione, pregúntale: *¿cómo podemos solucionar esto? ¿Cómo podemos evitar que cometas este error de nuevo? Revisa tus políticas y dime dónde y cómo propones la modificación.* O para un nuevo procedimiento: *hazme una propuesta de procedimiento para próximas ocasiones, y cómo forzaremos a que lo sigas a rajatabla en tareas similares.* El agente mejora su propia configuración. Es un ciclo de mejora continua que se retroalimenta.

## Productividad: lo que realmente cambia

El aumento de productividad es real, pero no viene de donde la gente piensa.

No es que escriba código más rápido. Es que **yo dedico más tiempo a lo que importa**: diseñar la arquitectura, pensar los edge cases, revisar con criterio. Las tareas mecánicas --- el boilerplate, los CRUDs, las migraciones, los tests unitarios repetitivos --- las hace Jarvis. Y las hace bien, porque tiene el contexto del proyecto.

Tareas que antes llevaban horas se resuelven en minutos. Las que llevaban días, en horas. Y no hablo de código chapucero que luego hay que rehacer. El código pasa review, pasa tests, y sigue los estándares del proyecto.

Ejemplo concreto: el <a href="https://github.com/jupaygon/symfony-dashboard-skeleton" target="_blank">symfony-dashboard-skeleton</a> --- un dashboard completo con EasyAdmin 5, multi-tenant, 4 temas, jerarquía de roles, i18n --- lo montamos en horas. No semanas. Horas. Con tests, documentación, y README completo.

## Documentación: el cambio que menos esperaba

Si hay algo que ha mejorado drásticamente y no esperaba, es la documentación.

La mayoría de developers odian documentar. Otros lo disfrutan pero no tienen tiempo. Yo estaba en el segundo grupo. Ahora, la documentación se genera como parte natural del flujo de trabajo. Jarvis documenta lo que implementa: README, comentarios en PR, descripciones de tickets Jira. Y yo reviso y ajusto el tono.

El resultado es que mis proyectos tienen mejor documentación que nunca. No como un extra que haces al final cuando te acuerdas, sino como parte del proceso. Cada PR lleva su descripción, cada ticket su resumen, cada repo su README actualizado.

Por ejemplo, he incorporado los **Project Books**: documentos HTML vivos con gráficos, colores y enlaces que reflejan la vida completa del proyecto --- desde el brainstorm inicial hasta producción y más allá. Incluyen roadmap, decisiones arquitectónicas, enlaces a los tickets de Jira, estado actual de cada pieza. Es como un dashboard del proyecto, pero narrativo. Y mantenerlos actualizados ya no es un esfuerzo extra, es parte del flujo.

## Stack: el abanico se abre

Este es quizás el cambio más inesperado.

Puedes llevar toda tu carrera en un lenguaje y frameworks concretos, en tu zona de confort. Pero programar es programar --- las buenas prácticas, SOLID, la separación de responsabilidades, la programación por capas... son universales e independientes del lenguaje.

Con este flujo de trabajo, yo aporto la experiencia en esos principios y Jarvis domina los detalles sintácticos de cada lenguaje. ¿Necesito un script en Python? ¿Un componente en Go? ¿Configuración de infraestructura? Puedo trabajar con confianza en stacks que antes me habrían exigido semanas de curva de aprendizaje.

No es que me haya convertido en experto en Python de la noche a la mañana. Es que puedo aplicar más de dos décadas de criterio de ingeniería en cualquier lenguaje, porque la parte sintáctica ya no es un cuello de botella.

## Seguridad: esto no es un juguete

Cuando le cuento a otros developers que tengo un agente IA con su propia cuenta de GitHub, algunos levantan una ceja. Suena arriesgado. Pero es exactamente lo contrario: es el enfoque más seguro.

- **Credenciales separadas** — Jarvis no tiene mis tokens ni mis contraseñas. Tiene los suyos, con permisos limitados.
- **Branch protection** — No puede pushear a master ni mergear PRs. Las reglas de protección de rama se lo impiden.
- **Permisos acotados** — En Jira puede crear y mover tickets, pero no puede borrar ni modificar configuraciones.
- **Trazabilidad total** — Cada acción queda registrada bajo su identidad. Si algo sale mal, sé exactamente qué hizo y cuándo.

La alternativa --- darle tu token personal con todos tus permisos --- es mucho más peligrosa. Y es lo que hace la mayoría de gente sin pensarlo.

## Lo que he aprendido

Después de meses con este sistema, estas son mis conclusiones:

1. **La IA no reemplaza al developer, lo amplifica.** El criterio de ingeniería sigue siendo humano. La IA ejecuta, el humano diseña y revisa.
2. **El flujo de trabajo importa más que el modelo.** Da igual que uses Claude, GPT o Gemini. Lo que marca la diferencia es cómo integras la IA en tu proceso.
3. **La documentación se soluciona sola** cuando es parte del flujo, no un extra.
4. **El stack ya no es una barrera.** Los principios de ingeniería son universales. La sintaxis es un detalle.
5. **La seguridad no es opcional.** Identidad separada, permisos limitados, branch protection. No hay atajos.

Si eres developer y aún piensas en la IA como "el autocompletado que a veces acierta", te animo a explorar esta otra forma de trabajar. No es perfecta, pero ha cambiado radicalmente cómo desarrollo software.

---

*Este blog --- incluyendo este post --- se ha construido siguiendo exactamente el flujo que describo. Puedes verlo en el <a href="https://github.com/jupaygon/jupaygon-web" target="_blank">repositorio</a>.*
