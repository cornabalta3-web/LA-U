# La U — App del equipo

App de una sola página (`index.html`) para gestionar el plantel, la cuota semanal,
el partido del sábado, los amistosos y el asado. Los datos se guardan en
**Firebase Realtime Database** (gratis) y se sincronizan solos en todos los celus.

## 1. Crear la base de datos gratis en Firebase (una sola vez)

1. Andá a **https://console.firebase.google.com** y entrá con una cuenta de Google.
2. Tocá **"Agregar proyecto"**, ponele un nombre (ej. `la-u-futbol`) y seguí los pasos
   (podés desactivar Google Analytics, no hace falta).
3. Adentro del proyecto, en el menú de la izquierda, andá a **Build > Realtime Database**.
4. Tocá **"Crear base de datos"**.
   - Elegí cualquier ubicación (la más cercana).
   - Cuando pregunte por las reglas de seguridad, elegí **"Iniciar en modo de prueba"**
     (test mode). Esto la deja abierta por 30 días; después hay que renovar las reglas
     (ver sección "Seguridad" más abajo).
5. Andá a **Configuración del proyecto** (el ícono de tuerca, arriba a la izquierda) >
   **"Tus apps"** > tocá el ícono `</>` (Web) para crear una app web.
   - Ponele un nombre (ej. `web`) y tocá "Registrar app". No hace falta Firebase Hosting.
6. Te va a mostrar un bloque de código con algo así:
   ```js
   const firebaseConfig = {
     apiKey: "AIza...",
     authDomain: "la-u-futbol.firebaseapp.com",
     databaseURL: "https://la-u-futbol-default-rtdb.firebaseio.com",
     projectId: "la-u-futbol"
   };
   ```
   Copiá esos 4 valores.

7. Abrí `index.html` (podés editarlo directo en GitHub) y buscá este bloque cerca del
   principio del `<script>`:
   ```js
   const firebaseConfig = {
     apiKey: "TU_API_KEY",
     authDomain: "TU_PROYECTO.firebaseapp.com",
     databaseURL: "https://TU_PROYECTO-default-rtdb.firebaseio.com",
     projectId: "TU_PROYECTO"
   };
   ```
   Reemplazá esos 4 valores por los tuyos y guardá.

## 2. Subir el repo a GitHub

1. En GitHub, creá un repositorio nuevo (puede ser público).
2. Subí `index.html` (y este `README.md`) — arrastrándolos desde la web de GitHub,
   o con `git add`, `git commit`, `git push` si ya usás git.
3. Andá a **Settings > Pages** del repo.
4. En "Source" elegí **"Deploy from a branch"**, rama `main`, carpeta `/ (root)`, y guardá.
5. GitHub te va a dar un link tipo `https://tu-usuario.github.io/tu-repo/` — ese es
   el que le pasás al grupo por WhatsApp.

## 3. Código de capitán

El código para registrarse como uno de los 2 capitanes está en el archivo, en esta línea:

```js
const CODIGO_CAPITAN = "LAU-CAPITAN";
```

Podés cambiarlo por el que quieras antes de subirlo. **Ojo:** como el archivo es
público en GitHub, cualquiera que sepa mirar el código fuente de la página puede
verlo — no es un secreto real, es solo un filtro para que no lo pongan por error.

## Seguridad de la base de datos (importante)

En "modo de prueba", cualquiera con el link de tu app puede leer y escribir los
datos del equipo (no hace falta ser capitán a nivel base de datos, solo a nivel
de la app). Para un grupo de amigos esto normalmente no es un problema, pero:

- El modo de prueba **expira a los 30 días** y después hay que renovarlo desde
  la consola de Firebase (Realtime Database > Reglas), o la app deja de guardar.
- Si en algún momento querés cerrarlo un poco más, avisame y te ayudo a armar
  reglas más restrictivas.

## ¿Y si no quiero pasar por todo esto?

Si te resulta mucho lío, la alternativa más simple sigue siendo usar el botón
"Publicar" del artefacto acá en Claude — no necesita cuenta de Firebase ni de
GitHub, aunque el link vive dentro de claude.ai en vez de tener tu propio dominio.
