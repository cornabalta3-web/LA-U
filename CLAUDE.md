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

**Un estado global y escrituras del árbol completo.** Todo el estado vive en un único
objeto `data` (`index.html:611`). Se persiste con `guardar()` (`index.html:1104`) →
`equipoRef.set(data)`, que **pisa el nodo `equipo-la-u` entero** en cada cambio. Una sola
suscripción `equipoRef.on("value", ...)` (`index.html:1148`) mezcla los snapshots de vuelta
en `data` y vuelve a renderizar, así todos los celus quedan sincronizados. No hay
escrituras parciales ni transacciones — gana la última escritura.

**El ciclo de render.** `render()` (`index.html:1335`) rearma de cero las stats del header,
la nav de abajo y la pestaña activa dentro de `#contenido`. Cada pestaña es una función
`v*()` que devuelve un nodo del DOM: `vSabado`, `vAmistosos`, `vCuota`, `vPlantel`,
`vStats`, `vCuentas`, `vAsado`, registradas en `TABS`/`ICONOS` (`index.html:1302`). Las
vistas construyen el DOM a mano y enganchan los handlers directo; **toda mutación sigue el
mismo patrón**:

```js
btn.onclick = async () => { /* modificar data */ ; await guardar(); render(); };
```

Agregar una pestaña = una función `v*()` + una entrada en `TABS` + una en `ICONOS` + una
línea en `render()`.

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
| `jugadores` | `[{id, nombre, exento}]` | el `id` sale de `uid()`, un string random. `exento: true` = es del grupo pero no paga cuota; lo marca un capitán desde Plantel y hace que `cuotaAlDia()` siempre dé `true` |
| `admins` | `[playerId]` | capitanes |
| `fechasCuota` / `fechaCuotaActual` | `[uid]` / `uid` | fechas de cuota; la última del array es la vigente |
| `cuota` | `{fechaId: {playerId: {declarado, confirmado}}}` | el jugador declara, el capitán confirma |
| `sabados` | `{"YYYY-MM-DD": [playerId]}` | lista de anotados; el orden es el de anotación (define la lista de espera según `CUPO_INTERNO`) |
| `sabadoConfig` | `{"YYYY-MM-DD": {hora, lugar, lugarOtro, modo, rival, resultado, golesF, golesC, votosMvp, goleadores}}` | `modo` es `"interno"` o contra un rival |
| `amistosos` | `[{id, fecha, hora, rival, anotados, ...mismos campos de resultado}]` | misma forma de resultado/MVP que `sabadoConfig` |
| `asados` | `[{id, ..., aportes}]` | quién lleva qué, según `APORTES` |
| `montoCuota`, `gastos` | string, `[{monto, ...}]` | caja del club: `totalRecaudado()` = pagos confirmados × `montoCuota` |
| `tricount` | `[{monto, pagadorId, participantes}]` | gastos divididos, se saldan con `balancesTricount()` + `liquidar()` |

Los sábados y los amistosos comparten a propósito la misma forma de resultado/MVP, así
`bloqueResultado()` (`index.html:898`), `normalizarResultado()`, `ganadorMvp()`,
`golesDe()` y `todosLosPartidos()` sirven para los dos.

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
  `pieInvitacion()`; varias vistas tienen su propio armador de mensaje.
- El estilo es CSS a mano con las variables de `:root` (`index.html:15`) — tema azul
  oscuro, pensado para el celu, ancho máximo 620px, nav fija abajo. Usá las variables que
  ya están (`--azul`, `--superficie*`, `--ok`, `--pend`, `--oro`) en vez de meter colores
  nuevos a mano.
