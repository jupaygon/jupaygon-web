---
title: 'Los agentes como usuarios de la infraestructura'
date: '2026-09-04'
slug: 'agent-as-a-user'
draft: false
description: 'Qué hacen de verdad tus agentes en tus máquinas, si los dejas. Cuidado con el sudoers que da root, las allowlist de SSH que permiten saltar a cualquier sitio, y otros peligros por exceso de confianza.'
tags: ['ai', 'security', 'agents', 'development', 'claude-code']
cover:
  image: '/en/posts/2026-09-04/agent-as-a-user/cover.jpg'
  alt: 'Un agente y una persona entran a una sala de servidores por puertas distintas, cada uno con su credencial y su línea en el log de acceso'
  relative: false
ShowToc: true
TocOpen: false
---

Tus agentes escriben código, abren PRs y quizás entran por SSH a tus máquinas. Párate a comprobar con qué permisos lo hacen de verdad, y no te fies de ficheros de policy y similares.

Descubrirás los problemas de control de acceso de toda la vida, los que aparecen en cuanto alguien que no eres tú empieza a entrar en tus servidores. ¿Por qué pasa? Porque tratas a los agentes como herramientas y no como usuarios.

## Una cuenta por agente

El agente no debe entrar en producción como `ubuntu`, con la misma clave que usas tu.

De lo contrario podrá hacer todo lo que puedas hacer tu mismo, y el log de autenticación no distinguirá sus sesiones de las tuyas. Si el «sólo lectura» de su acceso es una frase en un fichero de policy que el modelo lee al empezar la sesión, ojo, la máquina le dara root cuando lo pida con un sudo, si tu cuenta lo tiene.

El arreglo es una cuenta por agente y por host, con su propia clave. Cuesta un script de aprovisionamiento y hace que «quién hizo esto» lo conteste el log y no tu memoria, y ya se sabe que más vale un lápiz corto que una memoria larga.

## El sudoers no debe dar root

El siguiente paso es un sudoers con verbos de diagnóstico y nada más: leer el journal, mirar el estado de una unidad. Ningún restart, ningún stop, ningún dato. No debe dar uid 0 así gratuítamente, sin necesitarlo.

En sudoers, un comando declarado sin argumentos permite todos los argumentos. `journalctl` y `systemctl status` entrarán desnudos, y los dos paginan con `less` por defecto. Desde `less`, `!sh` abre una shell, que bajo sudo es una shell de root.

Otro camino puede ser `openssl x509`, que puedes tener clasificado como comando de inspección. Acepta `-out`, así que escribe un fichero: cualquiera, incluido el sudoers que acababa de conceder el permiso.

El «sólo lectura» resultará ser una propiedad de la invocación, no del binario. Cualquier cosa que pagine, que abra una shell o que escriba un fichero es una primitiva de escritura con nombre de verbo de lectura. Las especificaciones deben fijar el paginador y que ninguna se quede sin argumentos declarados.

## Si montas una allowlist de SSH, permitirá saltar a cualquier host

Otro control puede ser una allowlist de hosts a los que el agente pueda entrar por SSH. Conviertes cada host de la lista en un punto de salto hacia cualquier otro sitio.

Él puede encontrar formas de saltárselo. Las claves de las opciones de ssh no distinguen mayúsculas, así que una comprobación de `ProxyCommand` no ve `proxycommand`. Las comillas rompen la comparación. Y ni `Hostname` ni `-F` contienen la cadena «proxy», mientras que `Hostname` manda la conexión a otra máquina y `-F` apunta a un fichero de configuración que puede definir cualquier cosa.

Para invertirlo: una opción pasa sólo si su clave está en la lista de las que no pueden mover la conexión, y se rechaza `-F`.

Otro posible fallo: buscar `-F` en toda la línea, así que un `ssh host 'awk -F: ...'` queda bloqueado. Debe leer sólo los tokens que son de ssh, desde el `ssh` inicial hasta el host, saltándose el valor de las opciones que llevan uno, porque el `clave` de `-i clave` no es el host. Si va pegada no necesita caso especial: `-iclave` es un solo token que empieza por guion. Pasado el host es la línea de comandos de otro programa, salvo un `ssh` anidado, que abre su propia conexión y por eso también se le leen las opciones.

## Los tests podrían estar en verde todo el tiempo

Los controles anteriores pueden teener tests. Pero si comparas subcadenas del fichero de configuración renderizado, estarán en verde mientras el sudoers da root.

Comparar texto no prueba un control. El test del sudoers podria mejor parsear las especificaciones de verdad, y comprobar que falla al reintroducir la forma peligrosa. El del dry-run de SSH puede tener un `ssh` de pega en el path que revienta si alguien lo llama.

Mutar el código para ver los dos tests en rojo fueron serían minutos muy bien invertidos.

## El veredicto del halcón era texto en el transcript

Tengo un segundo modelo que revisa los turnos del agente principal, lo que [Alejandro Vidal](https://x.com/dobleio) llama un halcón. Cuando veía algo serio escala, y el escalado es texto inyectado en el transcript: para y consúltalo con el humano.

O sea que el veredicto sobre el agente se le entrega al agente, que decide luego si hacerle caso (y eso que Alejandro lo explica bastante bien 😅). A mi en una sesión el halcón escaló seis veces, el agente descartó las seis como falsos positivos y siguió. Una de las seis era un `git push --force`, que no debe decidir el agente por mucho que después resulte que tenía razón.

Ahora un escalado me escribe un fichero de bloqueo y un hook deniega cualquier herramienta mientras exista. Sin herramientas, la única acción posible es escribir texto, así que la única salida es preguntarle al humano. El agente tampoco puede quitar el bloqueo, porque la escritura y el comando que necesitaría se los deniega ese mismo hook. Me lo dice a mi, me explica lo que sucede, y si todo está bien, borro el fichero que lo bloquea.

La otra mitad del cambio: sólo escala lo que el código puede corroborar. Tres intentos idénticos, contados. Un comando de la lista de irreversibles, reconocido. Todo lo demás avisa sin paralizar nada, porque si una opinión de un modelo corta la sesión, lo único que has hecho es mover [la no determinación del agente a su halcón](/es/posts/2026-05-13/determinism-in-ai-workflows/).

## Los hooks no deben colgar de Write/Edit, sino de Bash

Los hooks de contenido no deben estar enganchados a las herramientas de edición de ficheros, sino a la shell.

De lo contrario, un heredoc a `python3`, un `cat >`, un `sed -i` o un `tee` podría escribir el fichero pasando por delante de ellos. No sería una regla la que se saltarían: serían todas, pasarían como Pedro por su casa sin que nadie le moleste. Los hooks no juzgarían mal ese código, simplemente no lo recibirían.

Merece la pena contar los caminos de entrada a lo que estás protegiendo, no solo las reglas. Un agente usa más caminos que una persona, se saben todas las triquiñuelas.

## Por dónde empezaría

Si tienes agentes actuando sobre tu infraestructura, la primera comprobación podría ser el log de autenticación: mira si sabes distinguir sus sesiones de las tuyas. Todo lo demás de este post salió de ahí.

El resto es trabajo normal: identidad antes que permisos, listas blancas en lugar de listas negras, un test que hayas visto fallar, un control en cada camino de entrada. Lo único nuevo es que el usuario del otro lado trabaja a las tres de la mañana y lee tu fichero de policy como un consejo, no le da miedo que lo despidas ;-). Y cuanto más ocurre dentro de [un bucle que corre sin ti](/es/posts/2026-06-17/loop-engineering/), más tardas en enterarte.
