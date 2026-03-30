---
title: 'AI Skills: cómo darle a tu agente IA conocimiento verificado que no tiene'
date: '2026-03-29'
slug: 'ai-skills'
aliases:
  - /posts/ai-skills-easyadmin5/
draft: false
description: 'Tu agente IA no sabe lo que cambió en la última versión de las tecnologías que usas. Los AI Skills resuelven eso: documentación estructurada, verificada y optimizada para máquinas. Así los creamos y así los usamos.'
tags: ['ai', 'easyadmin', 'symfony', 'claude-code', 'skills']
cover:
  image: '/en/posts/2026-03-29/ai-skills/cover.jpg'
  alt: 'AI Skills for coding agents'
  relative: false
ShowToc: true
TocOpen: false
---

Tu agente IA tiene un problema que no te cuenta: no sabe lo que no sabe. Cuando le pides que trabaje con una librería o herramienta que ha cambiado desde su fecha de entrenamiento, genera código con total confianza... usando métodos que ya no existen. No está mintiendo. Simplemente no tiene la información.

Este es el problema del **knowledge cutoff**, y afecta a todos los modelos. Para tecnologías estables y maduras, el impacto es menor. Pero para librerías, bundles y herramientas que evolucionan rápido, es un problema real que produce errores silenciosos: código que parece correcto pero falla en runtime.

La solución no es mejor prompting ni modelos más grandes. Es darle al agente **información actualizada y verificada** en un formato que pueda consumir. Eso es exactamente lo que hace un AI Skill.

## Qué es un AI Skill

Un skill es un documento Markdown estructurado que contiene conocimiento verificado sobre una tecnología concreta. No es un tutorial. No es documentación genérica. Es una **referencia optimizada para agentes IA**: firmas de API exactas, cambios entre versiones, patrones correctos, ejemplos mínimos y copiables.

Piénsalo como la diferencia entre darle a un developer nuevo un libro de 500 páginas y darle la cheat sheet verificada contra la versión exacta que estás usando. El agente no necesita entender toda la historia de una librería --- necesita saber qué métodos existen ahora, cuáles desaparecieron, y cómo se usan.

Un skill típico incluye:

- **Tabla de breaking changes** entre versiones --- lo primero que el agente consulta
- **Firmas de API** verificadas contra el código fuente, no contra documentación que puede estar desactualizada
- **Ejemplos de código** mínimos y funcionales, no tutoriales
- **Notas sobre trampas comunes** --- cosas que parecen correctas pero no lo son

## Cómo lo consume un agente

La integración es simple porque un skill es solo un fichero `.md`. No hay instalación, no hay dependencias, no hay build.

En **Claude Code**, lo colocas en `.claude/skills/` dentro de tu proyecto, o lo referencias desde tu `CLAUDE.md`:

```markdown
# CLAUDE.md
When working with EasyAdmin, read the skill at /path/to/skills/easyadmin5/SKILL.md before writing any code.
```

En **Cursor**, va en `.cursor/rules/`. En otros agentes, donde sea que coloques el contexto del proyecto.

Cuando el agente va a generar código con esa tecnología, lee el skill primero. Es como consultar la documentación antes de escribir --- pero automatizado y con información que sabemos que es correcta.

## Por qué no basta con la documentación oficial

La documentación oficial está pensada para humanos. Tiene contexto narrativo, explicaciones progresivas, ejemplos didácticos. Todo eso está bien para aprender, pero un agente IA necesita otra cosa:

- **Firmas exactas**, no párrafos explicativos
- **Cambios entre versiones** explícitos y en tabla, no enterrados en un UPGRADE.md
- **Datos verificados** contra el código fuente real, no contra documentación que a veces se desfasa del código
- **Formato consumible**: tablas, listas, bloques de código --- no prosa

Además, la documentación oficial cubre *todo*. Un skill cubre lo que el agente necesita saber para escribir código correcto: la API pública, los cambios que rompen, los patrones que funcionan.

## Cómo se crea un skill fiable

Aquí es donde se pone interesante. Crear un skill no es copiar la documentación y reformatearla. Es un proceso de verificación iterativo donde intervienen humanos y múltiples agentes IA, cada uno con un rol diferente.

El proceso que hemos seguido:

**1. Extracción inicial** --- Un agente investiga la documentación oficial, el código fuente del paquete, el UPGRADE.md, y produce un primer borrador con la API completa: métodos, firmas, parámetros, breaking changes.

**2. Primera revisión** --- Otro agente, diferente al que escribió el borrador, verifica cada afirmación contra el código fuente real. No contra la documentación --- contra el código. ¿Este método existe? ¿Esta firma es correcta? ¿Este parámetro es `object` o `mixed`?

**3. Corrección y segunda revisión** --- Se corrigen los errores encontrados y un tercer agente vuelve a revisar. El principio es el mismo que en code review: ojos frescos encuentran cosas que el autor no ve.

**4. Verificación humana** --- El humano revisa el resultado, aporta criterio sobre qué incluir y qué no, valida las correcciones, y a veces descubre que un revisor estaba equivocado y otro tenía razón.

**5. Iteración** --- Se repite hasta que la tasa de errores converge a cero.

¿Por qué múltiples agentes y no uno solo? Por la misma razón que no haces code review de tu propio código: el sesgo de confirmación. Un agente que escribió algo tenderá a validarlo. Un agente diferente lo cuestiona.

## Ejemplo real: EasyAdmin 5

Todo esto suena abstracto, así que voy con un caso concreto.

EasyAdmin es uno de los bundles más populares de Symfony. La versión 5 se publicó como nueva versión estable, eliminando todo lo deprecado de la serie 4.x. Funcionalmente es similar, pero la API cambió en sitios críticos:

- `linkToCrud()` → `linkTo()`
- `displayAsLink()` → `renderAsLink()`
- `addPanel()` → `addFieldset()`
- `{{ ea.property }}` en Twig → `{{ ea().property }}`
- Pretty URLs ahora obligatorias
- PHP 8.2+ y Symfony 6.4+ requeridos

Son cambios que parecen menores, pero producen errores en runtime que no son obvios. Y lo peor: tu agente está **convencido** de que su código es correcto, porque lo era... en la versión 4.

El detonante fue práctico: estaba migrando uno de mis proyectos de EA4 a EA5 con Jarvis, y el resultado era frustrante. Usaba métodos de la versión 4 con total confianza, daba por hecho que cosas de EA4 seguían funcionando en EA5, y generaba código que fallaba en runtime una y otra vez. No era un problema de capacidad del agente --- era un problema de información.

<a href="https://github.com/javiereguiluz" target="_blank">Javier Eguíluz</a> (creador de EasyAdmin) ha escrito sobre <a href="https://dev.to/javiereguiluz/claude-code-for-symfony-and-php-the-setup-that-actually-works-1che" target="_blank">cómo usar Claude Code con Symfony</a>, pero no existe un skill oficial de EasyAdmin 5 para agentes de IA, una referencia verificada y estructurada específica para EA5 que los agentes puedan consumir directamente.

Así que decidí crearla y compartirla con la comunidad Symfony, esperando que más programadores le vean la utilidad que yo le veo y la puedan aprovechar.

### Lo que encontraron los revisores

En la primera revisión, un agente verificó el borrador contra el código fuente de EasyAdmin 5 y encontró **8 errores factuales**.

En la segunda revisión, otro agente encontró **5 errores más**: un atributo con parámetros incorrectos (`routePath` en lugar de `path`), un formato de número engañoso, tokens de ImageField incompletos.

En la tercera revisión: 2 errores más. Y en la cuarta: 1 error y 1 imprecisión.

Cada ronda convergía: 8 → 5 → 2 → 1. Hasta llegar a un documento con más de 900 líneas verificadas donde cada firma, cada parámetro y cada ejemplo está contrastado contra el código fuente real del bundle.

### El resultado

Antes del skill, pedirle a un agente que creara un menú en EasyAdmin 5 producía código con `linkToCrud()` (EA4). Después del skill, genera `linkTo()` (EA5) a la primera. Lo mismo con campos, actions, filtros, eventos, templates Twig.

El skill cubre la API pública completa: dashboard, menús, CRUD controllers, 30 tipos de campo con documentación detallada, actions y action groups, batch actions, filtros, seguridad por niveles, eventos de ciclo de vida, pretty URLs, personalización de diseño, y generación de URLs.

## Skills como inversión

Crear un skill lleva tiempo. Pero es una inversión que se amortiza cada vez que tú --- o cualquier otro developer --- le pide a un agente que trabaje con esa tecnología.

Es open source. Verificado. Mantenido. Un fichero Markdown que cualquiera puede copiar a su proyecto y empezar a usar.

Si trabajas con EasyAdmin y un agente IA, pruébalo. Y si trabajas con otra tecnología que cambia entre versiones, considera crear tu propio skill. El proceso que he descrito funciona, y el resultado marca la diferencia entre un agente que alucina y uno que escribe código correcto a la primera.

---

*El skill de <a href="https://github.com/EasyCorp/EasyAdminBundle" target="_blank">EasyAdmin 5</a> está disponible en <a href="https://github.com/jupaygon/ai-skills" target="_blank">jupaygon/ai-skills</a>. Si te resulta útil, una estrella en el repo ayuda a que otros developers lo encuentren.*
