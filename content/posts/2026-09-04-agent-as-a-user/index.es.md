---
title: 'Tu agente es un usuario de tu infraestructura'
date: '2026-09-04'
slug: 'agent-as-a-user'
draft: false
description: 'En cuanto un agente actúa en vez de sugerir, deja de ser una herramienta y pasa a ser un principal de tu modelo de amenazas. Siete cosas que me costó aprender, empezando por el día que la cuenta de sólo lectura resultó ser root.'
tags: ['ai', 'security', 'agents', 'development', 'claude-code']
ShowToc: true
TocOpen: false
---

Durante un tiempo la pregunta interesante sobre los agentes de IA era cómo de bien escriben código. La pregunta interesante ahora es qué se les deja tocar.

Un agente que sugiere es una caja de texto muy elocuente. Un agente que actúa tiene una cuenta, una clave, una sesión en una máquina y una línea en `auth.log`. Eso ya no es una herramienta. Eso es un principal: algo sobre lo que tu control de acceso tiene que tener una opinión.

El mío no la tenía, durante meses. Esto es lo que encontré cuando por fin fui a mirar, y lo que costó arreglar cada cosa.

## La cuenta que nunca estuvo

Abre el log de autenticación de tu servidor. Si tu agente entra como el usuario genérico que trae la imagen del proveedor, con la misma clave de administrador que usas tú, entonces dos cosas son ciertas a la vez: puede hacer todo lo que puedes hacer tú, y después no hay forma de distinguir lo que hizo él de lo que hiciste tú.

Ese era mi montaje. El «sólo lectura» del agente lo sostenía una frase en un fichero de policy, que el modelo lee al empezar la sesión. La máquina le daba root. Lo único que lo mantenía en sólo lectura era que había leído el párrafo pidiéndoselo.

La identidad va antes que los permisos. Una cuenta por agente y por host, con su propia clave. No por desconfianza, sino porque «quién hizo esto» debería contestarlo el log y no la memoria.

## El mínimo privilegio es más difícil de lo que parece

Así que escribí un sudoers. Sólo verbos de diagnóstico: leer el journal, mirar el estado de una unidad, leer el buffer del kernel. Ningún restart, ningún stop, ningún dato. Muy razonable.

Daba uid 0. Por dos caminos que no tienen nada que ver entre sí.

El primero: en sudoers, un comando declarado sin argumentos permite *todos* los argumentos. `journalctl` y `systemctl status` entraban desnudos, y los dos paginan su salida con `less` por defecto. Desde `less`, `!sh` abre una shell. Bajo sudo, eso es una shell de root, conseguida a través de un permiso que yo había escrito como «que pueda leer los logs».

El segundo: `openssl x509` suena a comando de inspección. También acepta `-out`, que escribe un fichero: cualquiera, incluido el sudoers que acababa de conceder el permiso.

La lección no es «escribí un mal sudoers». Es que «sólo lectura» es una propiedad de la *invocación*, no del binario. Cualquier comando que pagine, que abra una shell o que escriba un fichero es una primitiva de escritura disfrazada con el nombre de un verbo de lectura. El arreglo fue fijar el paginador en cada especificación y no dejar ninguna sin argumentos declarados.

## Enumerar lo peligroso no funciona

Segundo control: el agente sólo puede abrir SSH contra los hosts de una allowlist.

Esa allowlist convertía cada host de la lista en un pivote hacia cualquier otro sitio.

El fallo tenía tres caras y las tres son el mismo error. Las claves de las opciones de ssh no distinguen mayúsculas, así que un guardián que busca `ProxyCommand` pasa de largo ante `proxycommand`. Las comillas rompían la comparación. Y, la peor de las tres, ni `Hostname` ni `-F` contienen la palabra «proxy». Pero `Hostname` redirige la conexión a otra máquina, y `-F` apunta a un fichero de configuración que puede definir absolutamente cualquier cosa.

Estaba enumerando las opciones peligrosas. Esa lista no es finita. La única versión que aguanta es la inversa: una opción pasa sólo si su clave está en la lista de las que *no pueden mover la conexión*, y `-F` se rechaza entero.

El mismo guardián me enseñó otra cosa por su cuenta. Buscaba `-F` en cualquier parte de la línea, así que un `awk -F:` dentro del comando remoto lo disparaba. Una bandera pertenece a un programa, no a una cadena: ahora sólo lee los tokens que son de ssh, caminando desde el `ssh` inicial hasta el host y saltándose el valor de las opciones que llevan uno, porque el `clave` de `-i clave` tampoco es el host. Lo que viene después del host es la línea de comandos de otro programa. La excepción es un `ssh` anidado, que abre su propia conexión y por eso también se le leen las opciones.

## El test que nunca ejecutó la forma peligrosa

Los dos controles tenían tests en verde. Los tests comparaban subcadenas del fichero de configuración renderizado. Estuvieron verdes todo el tiempo que el sudoers dio root.

Un test que compara texto no prueba un control. Lo que prueba un control es la forma que se supone que rechaza. El test del sudoers ahora parsea las especificaciones de verdad, y comprobé que falla si se reintroduce la forma peligrosa. El del dry-run de SSH pone un `ssh` de pega en el path que revienta a gritos si alguien lo invoca.

Si nunca has visto tu test de seguridad en rojo, no tienes un test de seguridad. Tienes un comentario escrito dentro de un framework de tests.

## Una instrucción no es un control

Este es el que me quedaría si sólo pudiera quedarme con uno.

Tengo un segundo modelo que vigila los turnos del agente principal y emite un veredicto. Cuando veía algo serio, escalaba, y el escalado era *texto inyectado en el transcript*: para, y llévaselo al humano.

Léelo otra vez. El veredicto sobre el agente se le entregaba al agente, que decidía después si obedecerlo. El acusado recibía el escrito del fiscal y resolvía él.

En una sesión el juez escaló seis veces. El agente descartó las seis como falsos positivos y siguió. Acertó las seis. Y eso no salva el mecanismo, porque una de las seis era un `git push --force`, y eso no lo decide el agente.

Ahora un escalado escribe un fichero de bloqueo, y un hook deniega *cualquier* herramienta mientras ese fichero exista. Sin herramientas, lo único que queda es escribir texto, así que la única salida es dirigirse al humano. El agente tampoco puede levantarlo: la escritura y el comando que necesitaría se los deniega ese mismo hook. Lo quita una acción humana, y nada más.

El corolario importa igual. Sólo escala lo que el código puede corroborar: tres intentos idénticos, contados; un comando de la lista de irreversibles, reconocido. Todo lo demás avisa sin paralizar nada. Lo que corta una sesión es siempre un hecho verificado, nunca la opinión de un modelo. Si no, has movido la no determinación del agente a su juez, que es [justo lo contrario de para lo que sirven las barandillas deterministas](/es/posts/2026-05-13/determinism-in-ai-workflows/).

## Un control que cubre un camino no es un control

Seis de mis hooks colgaban de las herramientas de edición de ficheros. Ninguno de la shell.

Así que un heredoc a `python3`, un `cat >`, un `sed -i`, un `tee`: cada uno de ellos escribía el fichero pasando por delante de los seis a la vez. No es que se saltara una regla: se saltaban todas, simultáneamente, por una puerta donde nadie había puesto un guardián. Los hooks no fallaron al juzgar ese código. No llegaron a verlo.

Cuando cuentes tus controles, cuenta los *caminos* de entrada a lo que estás protegiendo, no las reglas que escribiste. Un agente tiene muchas más formas de escribir un fichero de las que un humano se molesta en usar.

## Las credenciales vuelven solas

Este es corto, y es mi favorito. Encontré mis propias claves privadas en una máquina que usa el agente, y las borré. Un par de semanas después estaban otra vez ahí.

La receta de aprovisionamiento las copiaba, por diseño, y se ejecuta a menudo. Borrar un secreto de una máquina te compra exactamente el tiempo que falte hasta la siguiente pasada de lo que lo puso ahí. El arreglo nunca está en la máquina; está en lo que construye la máquina.

## Cierre

Nada de esto es seguridad exótica. Es la de siempre (identidad, mínimo privilegio, allowlists en vez de denylists, tests que pueden fallar, un control en cada camino), aplicada a un compañero que resulta ser un modelo de lenguaje, que trabaja a las tres de la mañana y que lee tu fichero de policy como un consejo.

Lo incómodo es la cronología. Todo lo anterior ya era cierto el día que le di al agente su primera credencial. Simplemente no había mirado. Y cuanto más trabajo le pasas a [un bucle que corre sin ti](/es/posts/2026-06-17/loop-engineering/), más largo es el hueco entre el error y el momento en que alguien se entera.

Así que empieza por el log. Si tu infraestructura sabe distinguir a tu agente de ti es una pregunta de sí o no, y la respuesta se tarda treinta segundos en encontrar.
