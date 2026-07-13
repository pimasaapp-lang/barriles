# Control de Barriles — Pimasa

App para digitalizar los 3 partes (1ª Revisión, 1ª Reparación, 2ª Reparación/Enderezado),
con datos en tiempo real vía **Firebase Firestore**, publicada en **GitHub Pages**, y con
una **Cloud Function** que vuelca los datos en un **Google Sheets** en vivo.

```
control-barriles-firebase/
├── index.html            ← la aplicación (esto es lo único que se publica en GitHub Pages)
├── firestore.rules       ← reglas de seguridad de la base de datos
├── firebase.json         ← configuración para el Firebase CLI
├── functions/
│   ├── index.js          ← Cloud Function: Firestore → Google Sheets
│   └── package.json
└── README.md              ← esta guía
```

Como ya tienes cuentas de GitHub y de Firebase/Google Cloud, aquí solo están los pasos
específicos de este proyecto.

---

## 1. Firebase: base de datos y usuarios

1. En la [consola de Firebase](https://console.firebase.google.com), crea un proyecto (o usa uno existente).
2. **Firestore Database** → *Crear base de datos* → modo producción → elige región (p. ej. `eur3` / Bélgica, la más cercana a España).
3. **Authentication** → *Sign-in method* → activa **Correo electrónico/contraseña**.
4. **Authentication** → pestaña *Users* → *Add user*:
   - Email: `admin@pimasa.local`
   - Contraseña: `1234`
   - (En la app se entra escribiendo usuario `Admin` y clave `1234` — la app traduce
     internamente el usuario a ese correo. Si más adelante quieres una cuenta por empleado,
     repite este paso con `empleado1@pimasa.local`, etc., y añade un email real como opción
     de recuperación si algún día quieres poder resetear contraseñas).
5. **Configuración del proyecto** (rueda dentada) → *Tus apps* → añade una app **Web** (icono `</>`).
   Copia el objeto `firebaseConfig` que te da Firebase.
6. Abre `index.html`, busca el bloque `const firebaseConfig = {...}` casi al principio del
   `<script>`, y pega ahí tus valores reales.

## 2. Desplegar las reglas de seguridad

Con el [Firebase CLI](https://firebase.google.com/docs/cli) instalado (`npm install -g firebase-tools`):

```bash
cd control-barriles-firebase
firebase login
firebase use --add        # elige tu proyecto de Firebase
firebase deploy --only firestore:rules
```

Esto sube `firestore.rules`, que exige estar autenticado para leer/escribir — así, aunque
alguien tenga la URL pública de GitHub Pages, no puede tocar los datos sin entrar con
usuario y clave válidos.

## 3. Publicar la app en GitHub Pages

1. Crea un repositorio en GitHub (puede ser privado) y sube el contenido de esta carpeta
   (sin el `.git` de ejemplo si lo hubiera).
2. En el repo: **Settings → Pages → Source**: rama `main`, carpeta `/ (root)`.
3. GitHub te dará una URL tipo `https://tu-usuario.github.io/control-barriles/`. Esa es la
   que compartes con tu equipo y con Heineken/DHL para ver el panel en vivo.

> El repositorio puede ser privado sin problema — GitHub Pages funciona igual, y de hecho
> es más prudente si prefieres que el código no sea público.

## 4. Google Sheets en tiempo real

1. Crea un Google Sheets nuevo (en la cuenta de Drive que uses con tu cliente).
2. Crea 3 pestañas con estos nombres **exactos**: `Revision1`, `Reparacion1`, `Reparacion2`.
3. Comparte ese Sheets con quien quieras que lo vea (por ejemplo, tu contacto en Heineken),
   como ya harías con cualquier documento de Drive.
4. Copia el **ID del Sheets** de la URL:
   `https://docs.google.com/spreadsheets/d/`**`ESTE_TROZO_ES_EL_ID`**`/edit`

### Cuenta de servicio (para que la Cloud Function pueda escribir en el Sheets)

1. En [Google Cloud Console](https://console.cloud.google.com), selecciona el mismo
   proyecto que tu Firebase (Firebase usa un proyecto de GCP por debajo).
2. **APIs y servicios → Biblioteca** → busca "Google Sheets API" → *Habilitar*.
3. **IAM y administración → Cuentas de servicio** → *Crear cuenta de servicio*
   (por ejemplo `sheets-sync`). No hace falta darle ningún rol de proyecto.
4. Entra en esa cuenta de servicio → pestaña **Claves** → *Agregar clave* → JSON.
   Se descarga un archivo `.json`: **guárdalo, no lo subas nunca a GitHub**.
5. Abre ese JSON y copia el valor de `"client_email"` (algo como
   `sheets-sync@tu-proyecto.iam.gserviceaccount.com`).
6. Vuelve a tu Google Sheets → botón **Compartir** → pega ese email → dale permiso de
   **Editor**. Sin este paso, la función no podrá escribir aunque tenga la clave.

### Configurar y desplegar la Cloud Function

Los datos sensibles (la clave de la cuenta de servicio y el ID del Sheets) se guardan como
**secretos** de Firebase Functions, nunca en el código:

```bash
cd control-barriles-firebase

# Pega aquí el CONTENIDO COMPLETO del .json de la cuenta de servicio cuando te lo pida:
firebase functions:secrets:set GOOGLE_SERVICE_ACCOUNT_JSON

# Pega aquí solo el ID del Sheets:
firebase functions:secrets:set SHEET_ID

firebase deploy --only functions
```

A partir de aquí, cada vez que se cree, edite o borre un parte en la app, la función
correspondiente regenera esa pestaña del Google Sheets en cuestión de segundos.

## 5. Probar que todo encaja

1. Abre la URL de GitHub Pages, entra con `Admin` / `1234`.
2. Crea un parte de prueba y rellena un par de casillas.
3. En la consola de Firebase → Firestore, comprueba que aparece el documento.
4. Abre el Google Sheets: la pestaña correspondiente debería tener los datos al cabo de
   pocos segundos (revisa **Firebase Console → Functions → Registros** si algo no llega,
   ahí se ven los errores exactos — el motivo más habitual es no haber compartido el
   Sheets con el email de la cuenta de servicio).

## Notas

- El usuario/clave de la app pide autenticación real contra Firebase (no es solo una
  cortina visual como en la primera versión) — sin una cuenta creada en el paso 1, nadie
  puede entrar.
- La Cloud Function reescribe toda la pestaña en cada cambio en lugar de tocar una fila
  concreta: es una estrategia simple y robusta, de sobra para el volumen de partes de un
  taller. Si en el futuro son miles de partes por pestaña, se podría optimizar para tocar
  solo la fila que cambia.
- Si algún día quieres cuentas individuales por empleado en vez de un único `Admin`, basta
  con crear más usuarios en Firebase Authentication — el resto de la app no cambia.
