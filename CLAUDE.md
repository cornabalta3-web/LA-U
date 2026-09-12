# CLAUDE.md

Este archivo le da contexto a Claude Code (claude.ai/code) para trabajar en este repositorio.

## Qué es esto

"Urracas" / "La U" — una app de una sola página para manejar un equipo de fútbol amateur:
la lista del sábado, la cuota semanal, los amistosos, las estadísticas, un tricount de
gastos compartidos y el asado. **Toda la app es `index.html`** (~2500 líneas: el CSS en un
solo `<style>`, el JS en un solo `<script>`). No hay build, ni gestor de paquetes, ni
tests, ni framework.

Los textos de la UI, los nombres de variables y los comentarios están en castellano
rioplatense — seguí escribiéndolos así.

## Correr y publicar

- Local: abrí `index.html` directo en el navegador (`file://` anda perfecto). La config de
  Firebase está hardcodeada, así que pega contra la base real del equipo.
- En esta máquina **no hay `node` ni `python`**, pero sí Edge, que sirve para probar sin
  abrir el navegador a mano ni tocar la base real:

  ```bash
  # 1) copia de prueba sin Firebase (el try/catch cae en el ref falso, no hay red)
  cp index.html "$SP/test.html"
  sed -i 's|<script src="https://www.gstatic.com/firebasejs[^>]*></script>||' "$SP/test.html"
  # 2) agregar después de `arrancar();` un bloque que llene `data`, fije `miPerfil` y `tab`,
  #    llame a render() y vuelque lo que se quiera revisar en un <pre id="resultado-test">
  #    (para los mensajes de WhatsApp, pisar window.open y capturar el texto)
  # 3) renderizar y leer el resultado
  "/c/Program Files (x86)/Microsoft/Edge/Application/msedge.exe" --headless=new --disable-gpu \
    --virtual-time-budget=9000 --dump-dom "file:///$SP/test.html" > "$SP/dom.html"
  ```

  Como todo es global y sin módulos, desde ese bloque se puede tocar `data`, `miPerfil`,
  `tab` y llamar a `render()` directo. Si hubiera un error de sintaxis, el modal de ingreso
  no aparece en el DOM volcado.
- Deploy: GitHub Pages desde `main` / raíz. Cualquier commit a `main` es un deploy — todo
  el equipo comparte una única Realtime Database (`equipo-la-u`), o sea que los cambios
  tocan datos reales.
- `README.md` explica, para el usuario final, cómo armar el proyecto de Firebase y el
  código de capitán.

## Arquitectura

**Un estado global y tres formas de guardarlo.** Todo el estado vive en un único objeto
`data`. Una sola suscripción `equipoRef.on("value", ...)` mezcla los snapshots de vuelta en
`data` y vuelve a renderizar, así todos los celus quedan sincronizados. Para escribir hay
tres herramientas, **de menos a más segura**:

| Función | Qué manda | Cuándo usarla |
|---|---|---|
| `guardar()` | `equipoRef.set(data)`: **el árbol entero** | cosas que toca un capitán solo (configurar el partido, gastos, asado, plantel) |
| `guardarRama(ruta, valor)` | solo esa ramita | cuando varios pueden guardar a la vez pero cada uno toca **su** dato (cuota de cada jugador, voto propio) |
| `cambiarAnotado(sab, pid, entrar)` | transacción sobre `sabados/<fecha>` y `sabadoNo/<fecha>` | listas que varios modifican **al mismo tiempo**: anotarse al sábado |

La regla es por qué se pisan: `guardar()` manda una foto completa de `data`, así que si dos
personas guardan casi juntas, la segunda revierte lo que hizo la primera. `guardarRama()`
achica el daño a un solo campo; la transacción lo elimina, porque Firebase reintenta sobre
el valor del servidor. **Si agregás algo que varios puedan tocar en simultáneo, no uses
`guardar()`.**

**El ciclo de render.** `render()` (`index.html:1335`) rearma de cero las stats del header,
la nav de abajo y la pestaña activa dentro de `#contenido`. Cada pestaña es una función
`v*()` que devuelve un nodo del DOM: `vSabado`, `vAmistosos`, `vCuota`, `vPlantel`,
`vStats`, `vCuentas`, `vAsado`, registradas en `TABS`/`ICONOS` (`index.html:1302`). Las
vistas construyen el DOM a mano y enganchan los handlers directo; **toda mutación sigue el
mismo patrón**:

```js
btn.onclick = async () => { /* modificar data */ ; await guardar(); render(); };
```

**La barra de abajo tiene solo 4 botones a propósito**: `TABS` son los de uso semanal
(Partido, Amistosos, Cuota) más "Más"; `SECUNDARIAS` (Plantel, Números, Asado, Cuentas)
viven detrás de `vMas()` y se abren con un "Volver" arriba. Con la app abierta en una
secundaria, el que queda encendido en la barra es "Más". La regla es que alguien que entra
por primera vez vea tres opciones, no siete — **no agregues botones a `TABS`**, sumá a
`SECUNDARIAS`. Agregar una pantalla = una función `v*()` + una entrada en `SECUNDARIAS` +
una en `ICONOS` + una línea en `render()`.

**Novedades: hay que mantenerlas a mano.** `VERSION_APP` y `NOVEDADES` están arriba del
todo en el `<script>`. Cuando entrás y la app tiene una versión más nueva que la guardada
en `localStorage` (`NOVEDADES_KEY`), salta el cartel una sola vez; el pie de página lo
vuelve a abrir. **Cada vez que subas un cambio que el equipo pueda notar, sumá una entrada
a `NOVEDADES` y subí `VERSION_APP`** — si no, el cambio pasa sin que nadie se entere.
Ojo con `mostrandoNovedades`: existe para que un snapshot de Firebase no le borre el cartel
al que lo está leyendo.

**Lo importante.** `pendientesMios()` arma la lista de cosas pendientes del jugador que
está mirando (cuota, si no contestó por el sábado, votar la figura, y para capitanes
confirmar pagos y cargar resultados) y `bloqueImportante()` la dibuja arriba de la pestaña
Sábado. Cada ítem lleva `tab` y a veces `fecha`, y al tocarlo salta a esa pestaña y a ese
sábado. Los avisos viejos siguen viviendo en su propia pestaña; esto es el resumen.

**Identidad y permisos.** No hay autenticación. `miPerfil` (`{id, nombre}`) se guarda en
`localStorage` bajo `PERSONAL_KEY` y se elige en el modal de entrada `mostrarModal()`
(`index.html:1192`) — o elegís un jugador existente, o creás uno nuevo. Tildar "soy
capitán" pide el `CODIGO_CAPITAN` (`index.html:600`), que lo único que hace es sumar el id
a `data.admins`; `soyCapitan()` habilita la UI de capitanes del lado del cliente. El código
está a la vista en el fuente y la base está abierta — es un filtro blando, no seguridad. Si
`miPerfil.id` ya no está en `data.jugadores`, se borra el perfil y vuelve a aparecer el
modal.

**Sin conexión / fallas.** El init de Firebase está envuelto en try/catch con un `equipoRef`
falso para que la pantalla igual renderice; además `arrancar()` (`index.html:1131`) arranca
un timeout de 6s que levanta la UI vacía y muestra el banner de conexión si Firebase nunca
contesta. Si tocás el arranque, mantené los tres caminos (snapshot, callback de error,
timeout) coherentes entre sí.

## Modelo de datos (`data`)

| Clave | Forma | Notas |
|---|---|---|
| `jugadores` | `[{id, nombre, exento, desde, tel}]` | el `id` sale de `uid()`, un string random. `exento: true` = es del grupo pero no paga cuota; lo marca un capitán desde Plantel y hace que `cuotaAlDia()` siempre dé `true`. `desde` es el id de fecha a partir del cual se le cuenta la deuda (`fechasQueDebe()`); si falta, cuenta desde la primera |
| `admins` | `[playerId]` | capitanes |
| `fechasCuota` | `["2026-09", ...]` | una fecha de cuota por mes; el id **es** el mes. `abrirPeriodoSiHaceFalta()` agrega el mes en curso al arrancar (idempotente, lo puede correr cualquier celu), así que nadie tiene que crear fechas a mano. Los ids random de antes de esto siguen funcionando y se etiquetan "Fecha 1", "Fecha 2". La última del array es la vigente. Para cobrar por semana en vez de por mes se cambia `periodoActual()` |
| `fechaCuotaActual` | — | **muerto**: quedó en la base pero ya no se usa. Cambiar de mes en la pestaña Cuota es local (`fechaVista`); antes se guardaba, y mirar un mes viejo se lo cambiaba a todo el equipo |
| `cuota` | `{fechaId: {playerId: {declarado, confirmado}}}` | el jugador declara, el capitán confirma. Deber **no bloquea nada**: se avisa en "Lo importante" y con una píldora, pero el que debe se anota igual (antes se le deshabilitaban los botones) |
| `sabados` | `{"YYYY-MM-DD": [playerId]}` | lista de anotados; el orden es el de anotación (define la lista de espera según `CUPO_INTERNO`). Se toca **solo** con `cambiarAnotado()` |
| `sabadoNo` | `{"YYYY-MM-DD": [playerId]}` | los que dijeron NO JUEGO. Sin esto, "dijo que no" y "no contestó" eran indistinguibles; `dijoSiJuega()` los separa |
| `sabadoConfig` | `{"YYYY-MM-DD": {hora, lugar, lugarOtro, modo, rival, resultado, golesF, golesC, votosMvp, goleadores}}` | `modo` es `"interno"` o contra un rival |
| `amistosos` | `[{id, fecha, hora, rival, anotados, ...mismos campos de resultado}]` | misma forma de resultado/MVP que `sabadoConfig` |
| `asados` | `[{id, ..., aportes}]` | quién lleva qué, según `APORTES` |
| `montoCuota`, `alias`, `gastos` | string, string, `[{monto, ...}]` | caja del club: `totalRecaudado()` = pagos confirmados × `montoCuota`. `alias` es el alias para transferir: se muestra en Cuota y se cuela en todos los mensajes de cobranza |
| `tricount` | `[{monto, pagadorId, participantes}]` | gastos divididos, se saldan con `balancesTricount()` + `liquidar()` |

Los sábados y los amistosos comparten a propósito la misma forma de resultado/MVP, así
`bloqueResultado()` (`index.html:898`), `normalizarResultado()`, `ganadorMvp()`,
`golesDe()` y `todosLosPartidos()` sirven para los dos.

## Reglas que no se pueden romper

Cada una salió de un problema real, no de una preferencia:

- **Nunca escribir sin haber leído.** `hayDatos` se prende recién cuando llega el primer
  snapshot; `guardar()`, `guardarRama()` y `cambiarAnotado()` cortan en seco si está en
  false. Sin esto, cuando Firebase no contestaba en 6s la app arrancaba con el molde vacío
  y **el primer guardado borraba la base del equipo entero**.
- **Nunca `toISOString()` para una fecha del calendario.** Devuelve UTC y Argentina va 3
  horas atrás: pasadas las 21 hs devolvía el día siguiente, y con la cuota atada al sábado
  eso abría fechas de domingo. Se usa `isoLocal(d)` / `hoyISO()`.
- **El snapshot reemplaza, no mezcla.** `data = Object.assign(estadoVacio(), v)`. Con
  `Object.assign(data, v)` una clave borrada en la base seguía viva en este celu y volvía a
  subir en el próximo guardado.
- **Sacar un jugador es `purgarJugador(id)`**, no un filter sobre `jugadores`. Su id vive
  también en `sabados`, `sabadoNo`, `cuota`, `votosMvp`, `goleadores`, `equipos`, los
  amistosos, los aportes del asado y el tricount; si queda, ocupa lugar del cupo del sábado
  y aparece como "(?)" en los mensajes de WhatsApp.
- **La cuota se cobra por fecha y la paga todo el mundo, juegue o no.** La cancha se paga
  igual. `fechasQueDebe()` no mira si el jugador estuvo en `data.sabados[f]`: el único que
  queda afuera es el exento. (Se probó atarla a quién jugó y el dueño lo rechazó de plano.)
- **No re-renderizar encima de alguien que escribe.** El callback del snapshot se corta si
  `document.activeElement` es un input/select/textarea; si no, un cambio de otro celu le
  borraba al capitán el resultado a medio cargar. Por lo mismo el modal de ingreso no se
  rehace si ya está abierto.

## Cosas para tener en cuenta

- **Firebase borra los arrays y objetos vacíos.** Todo lo que vuelve faltando hay que
  reponerlo en `sanitizar()` (`index.html:1108`) — agregá una línea ahí por cada campo
  nuevo que sea array u objeto, o el código que lo da por existente va a romper después de
  la primera ida y vuelta.
- **Las fechas son strings `YYYY-MM-DD`** que se usan directo como claves de objeto y se
  ordenan con `localeCompare`. `proximoSabado()` calcula el sábado que viene; `sabadoVisto`
  guarda la fecha que se está mirando (`moverSabado()` se mueve ±7 días).
- **No se escapa nada.** Los nombres de jugadores, los rivales y las descripciones de
  gastos entran tal cual en `innerHTML` por template literals. Tenelo presente antes de
  abrir cualquier input nuevo.
- Compartir por WhatsApp es armar un texto y abrir `wa.me` con `compartirWsp()` /
  `pieInvitacion()`; varias vistas tienen su propio armador de mensaje. Para el chat
  privado está `wspPrivado(tel, texto)`, que normaliza el teléfono con `telWsp()` (le pone
  el 54 9 y le saca el 15) y devuelve `false` si el jugador no tiene número cargado — ahí
  se cae a `copiar()`. Para una lista de difusión no hay link posible: se copia el texto
  (`textoDifusion()`) y la persona lo pega a mano.
- **Ser capitán** se consigue con `CODIGO_CAPITAN` al crear el perfil, al entrar con un
  nombre que ya existe, o desde el botón de Plantel. Antes solo servía la primera vía, y
  quien ya estaba en la lista quedaba sin poder cargar resultados salvo que otro capitán lo
  ascendiera con el ☆.
- El estilo es CSS a mano con las variables de `:root` (`index.html:15`) — **azul** oscuro,
  el del escudo, pensado para el celu, ancho máximo 620px, nav fija abajo. Usá las variables
  que ya están (`--azul`, `--superficie*`, `--ok`, `--pend`, `--oro`) en vez de meter
  colores nuevos a mano. El violeta del buzo **no** es el color del club: el club es azul
  porque es una urraca (se probó violeta y el dueño lo corrigió).
- El escudo es un SVG dibujado a mano en `escudoSVG(ancho)` (campo azul, banda
  "URRACAS", urraca de ala azul y banderola "FC"). Está en el encabezado
  (ahí va pegado en el HTML), en el modal de ingreso y en `ESCUDO_SVG`. Para verlo mientras
  se lo retoca conviene renderizarlo con Edge y `--screenshot`, que muestra cómo queda a
  46px, que es el tamaño real.
- **Un resultado sin `resultado` es un partido invisible**: `todosLosPartidos()` filtra por
  ese campo, así que no entra ni en el historial ni en el balance ni en la racha. Pasó en
  vivo: cargaron 7-3 y el partido no contó. Por eso `normalizarResultado()` lo deduce del
  marcador y el formulario acomoda el desplegable solo mientras escribís los goles.
