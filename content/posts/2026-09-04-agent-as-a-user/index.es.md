---
title: 'Los agentes como usuarios de la infraestructura'
date: '2026-09-04'
slug: 'agent-as-a-user'
draft: false
description: 'Notas de revisar qué podían hacer de verdad mis agentes en mis máquinas: el sudoers que daba root, la allowlist de SSH que permitía saltar a cualquier sitio, y otras tres cosas que daba por buenas.'
tags: ['ai', 'security', 'agents', 'development', 'claude-code']
cover:
  image: '/en/posts/2026-09-04/agent-as-a-user/cover.jpg'
  alt: 'Un agente y una persona entran a una sala de servidores por puertas distintas, cada uno con su credencial y su línea en el log de acceso'
  relative: false
ShowToc: true
TocOpen: false
---

Tengo agentes que escriben código, abren PRs y entran por SSH a máquinas. En agosto me senté a comprobar con qué permisos lo hacían de verdad, en lugar de con los que decían mis ficheros de policy.

Descubrí los problemas de control de acceso de toda la vida, los que aparecen en cuanto alguien que no eres tú empieza a entrar en tus servidores. ¿Por qué pasaba? Porque trataba a los agentes como herramientas y no como usuarios.

## Una cuenta por agente

El agente entraba en producción como `ubuntu`, con la misma clave de administrador que uso yo.

Podía hacer todo lo que puedo hacer yo, y el log de autenticación no distinguía sus sesiones de las mías. El «sólo lectura» de su acceso era una frase en un fichero de policy que el modelo lee al empezar la sesión; la máquina le daba root.

El arreglo es una cuenta por agente y por host, con su propia clave. Cuesta un script de aprovisionamiento y hace que «quién hizo esto» lo conteste el log y no mi memoria, que como suelo decir, no es la mejor de mis cualidades.

## El sudoers daba root

El siguiente paso fue un sudoers con verbos de diagnóstico y nada más: leer el journal, mirar el estado de una unidad. Ningún restart, ningún stop, ningún dato.

Daba uid 0 por dos caminos.

En sudoers, un comando declarado sin argumentos permite todos los argumentos. `journalctl` y `systemctl status` entraban desnudos, y los dos paginan con `less` por defecto. Desde `less`, `!sh` abre una shell, que bajo sudo es una shell de root.

El otro camino era `openssl x509`, que yo tenía clasificado como comando de inspección. Acepta `-out`, así que escribe un fichero: cualquiera, incluido el sudoers que acababa de conceder el permiso.

El «sólo lectura» resultó ser una propiedad de la invocación, no del binario. Cualquier cosa que pagine, que abra una shell o que escriba un fichero es una primitiva de escritura con nombre de verbo de lectura. Ahora las especificaciones fijan el paginador y ninguna se queda sin argumentos declarados.

## La allowlist de SSH permitía saltar a cualquier host

El siguiente control era una allowlist de hosts a los que el agente puede entrar por SSH. Convertía cada host de la lista en un punto de salto hacia cualquier otro sitio.

Tres formas de saltárselo, y las tres son el mismo error. Las claves de las opciones de ssh no distinguen mayúsculas, así que una comprobación de `ProxyCommand` no ve `proxycommand`. Las comillas rompían la comparación. Y ni `Hostname` ni `-F` contienen la cadena «proxy», mientras que `Hostname` manda la conexión a otra máquina y `-F` apunta a un fichero de configuración que puede definir cualquier cosa.

Estaba enumerando opciones peligrosas, y esa lista no se acaba nunca. Ahora está invertido: una opción pasa sólo si su clave está en la lista de las que no pueden mover la conexión, y `-F` se rechaza entero.

El mismo guardián tenía un segundo fallo que merece la pena contar: buscaba `-F` en toda la línea, así que un `ssh host 'awk -F: ...'` quedaba bloqueado. Ahora lee sólo los tokens que son de ssh, desde el `ssh` inicial hasta el host, saltándose el valor de las opciones que llevan uno, porque el `clave` de `-i clave` no es el host. La forma pegada no necesita caso especial: `-iclave` es un solo token que empieza por guion. Pasado el host es la línea de comandos de otro programa, salvo un `ssh` anidado, que abre su propia conexión y por eso también se le leen las opciones.

## Los tests estaban en verde todo el tiempo

Los dos controles anteriores tenían tests. Comparaban subcadenas del fichero de configuración renderizado, así que pasaban mientras el sudoers daba root.

Comparar texto no prueba un control. El test del sudoers ahora parsea las especificaciones de verdad, y comprobé que falla al reintroducir la forma peligrosa. El del dry-run de SSH pone un `ssh` de pega en el path que revienta si alguien lo llama.

Mutar el código para ver los dos tests en rojo fueron unos minutos muy bien invertidos.

## El veredicto del halcón era texto en el transcript

Tengo un segundo modelo que revisa los turnos del agente principal, lo que [Alejandro Vidal](https://x.com/dobleio) llama un halcón. Cuando veía algo serio escalaba, y el escalado era texto inyectado en el transcript: para y consúltalo con el humano.

O sea que el veredicto sobre el agente se le entregaba al agente, que decidía luego si hacerle caso (y eso que Alejandro lo explica bastante bien 😅). En una sesión el halcón escaló seis veces, el agente descartó las seis como falsos positivos y siguió, y acertó las seis. Una de las seis era un `git push --force`, que no lo decide el agente por mucho que después resulte que tenía razón.

Ahora un escalado escribe un fichero de bloqueo y un hook deniega cualquier herramienta mientras exista. Sin herramientas, la única acción posible es escribir texto, así que la única salida es dirigirse al humano. El agente tampoco puede quitar el bloqueo, porque la escritura y el comando que necesitaría se los deniega ese mismo hook.

La otra mitad del cambio: sólo escala lo que el código puede corroborar. Tres intentos idénticos, contados. Un comando de la lista de irreversibles, reconocido. Todo lo demás avisa sin paralizar nada, porque si una opinión de un modelo corta la sesión, lo único que has hecho es mover [la no determinación del agente a su halcón](/es/posts/2026-05-13/determinism-in-ai-workflows/).

## Los hooks colgaban de Write/Edit, no de Bash

Seis hooks de contenido estaban enganchados a las herramientas de edición de ficheros y ninguno a la shell.

Así que un heredoc a `python3`, un `cat >`, un `sed -i` o un `tee` escribían el fichero pasando por delante de los seis a la vez. No era una regla la que se saltaba: eran todas, por un camino donde nadie había puesto un guardián. Los hooks no juzgaron mal ese código, no llegaron a recibirlo.

Merece la pena contar los caminos de entrada a lo que estás protegiendo, más que las reglas que escribiste. Un agente usa más caminos que una persona.

## Por dónde empezaría

Si tienes agentes actuando sobre tu infraestructura, la comprobación más barata es el log de autenticación: mira si sabes distinguir sus sesiones de las tuyas. Todo lo demás de este post salió de ahí.

El resto es trabajo normal. Identidad antes que permisos, allowlists en vez de enumerar lo peligroso, un test que hayas visto fallar, un control en cada camino de entrada. Lo único distinto es que el usuario del otro lado trabaja a las tres de la mañana y lee tu fichero de policy como un consejo. Y cuanto más ocurre dentro de [un bucle que corre sin ti](/es/posts/2026-06-17/loop-engineering/), más tardas en enterarte.
