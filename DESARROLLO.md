# FotosCompare · Notas de desarrollo

Para retomar el trabajo en un chat nuevo sin volver a investigar todo. Decí "seguimos con FotosCompare, <etapa>, leé DESARROLLO.md". Al cerrar cada etapa, actualizar este archivo con lo necesario para la siguiente.

## Estado

| Versión | Contenido |
|---|---|
| v0.0.1 | Takeout (.zip con ZIP64 o carpeta), Picker de Google Fotos, carpeta/archivos locales, comparación por niveles, verificación de huella, CSV |
| v0.0.2 | Comparación visual (dHash), pestaña Duplicados, HEIC/HEIF/AVIF |
| v0.1.0 | Revisar respaldos (con candados y "Apartar" reversible), Por carpeta, guardado de índices e historial (IndexedDB), alta en la caja de apps |
| v0.1.1 | Takeout desde Drive, **Parte 1**: botón "☁️ Elegir los .zip en Drive" (selector de Google, permiso `drive.file`), lectura por tramos (Range) del índice y los `.json`, clave de API en "Configurar acceso a Google", ayuda |

**Próximas etapas acordadas** (todas con **Release Patch**, ver "Versiones"):
- **Takeout desde Drive, Parte 2** (próximo chat: "seguimos con FotosCompare, Takeout desde Drive Parte 2, leé DESARROLLO.md"):
  - Prueba completa en Chrome headless con `google.picker` y Drive simulados (`page.route`), como la del Picker de Fotos. En la Parte 1 se probó el núcleo en Node (`driveFile` + `zipEntries` + `readTakeout` contra un Drive falso: mismo resultado que el .zip local, servidor que ignora Range, 401, reintento en 429) y que la página carga y avisa si falta la clave.
  - Miniaturas y huella desde Drive: ya funcionan por `entBlob` (bajan solo esa foto), pero el token dura 1 hora. Falta renovarlo sin volver a elegir los .zip (`getDriveToken` con `prompt: ""` desde un clic) y probarlo.
  - Índice guardado de un Takeout de Drive: guardar `id`, `name` y `size` de cada .zip, para reconectar sin volver a elegirlos (con `drive.file` el permiso sobre esos archivos queda).
  - README: sección "Takeout desde Drive".
  - Velocidad con Takeouts grandes: hoy es un pedido por cada `.json` (16 en paralelo). Si es lento, agrupar `.json` cercanos en un solo tramo.
  - Probar contra Google real cuando el usuario tenga la clave de API (ver "Takeout desde Drive").
- **Comparación visual de videos** (antes v0.3.0).

## Decisiones del usuario (no cambiar sin consultar)

- **Nunca borrar.** La app solo identifica, marca y "aparta": mueve los archivos a `_para_borrar` dentro de la carpeta elegida, con un registro CSV y "Deshacer". El usuario no quiere un "Liberar espacio" como el de Google Fotos.
- **Nada viene preseleccionado.** Para marcar en bloque se usa "Marcar los visibles", con confirmación.
- **Solo copias exactas son marcables:** mismo tamaño más nombre o fecha (Exacto o Idéntico), o huella SHA-256 verificada. Fuerte, Visual, Probable y las fotos elegidas con el Picker (que no traen tamaño) quedan bloqueadas.
  - Si la nube pesa menos (recompresión), queda bloqueado.
  - Opciones del usuario: cuentas que valen como respaldo, carpetas protegidas y "Exigir huella".
- **No le interesa "Recuperar lo que falta"** (descargar de la nube al dispositivo): su objetivo es liberar espacio físico con control.
- **Versiones:** estándar de la caja de apps (`home/ESTANDAR-VERSIONES.md`), `<meta name="app-version">` y `[patch]`/`[minor]`/`[major]` en el commit.
  - **Desde v0.1.1, todas las versiones salen con Release Patch** (v0.1.2, v0.1.3…) hasta que el usuario indique otra cosa. Se le puede sugerir un Minor o Major cuando convenga, pero sin aplicarlo sin su respuesta.
- **Takeout desde Drive con permiso acotado** (`drive.file` + selector de Google): la app solo ve los .zip que el usuario elige. No usar el permiso de todo Drive.

## Arquitectura

Un solo `index.html`, sin dependencias. Se arma desde tres fuentes con un script (ver "Cómo se construye"):

1. **head.html:** estilos (tema claro y oscuro con `data-tema`) y el HTML de la interfaz.
2. **core.js:** objeto `FC`, sin interfaz, que también se puede probar en Node:
   - `zipEntries`/`zipBlob`: lector de ZIP propio. Lee el directorio central y ZIP64, y descomprime con `DecompressionStream("deflate-raw")`.
   - `readTakeout`: une cada foto con su `.json` (`supplemental-metadata` y sus versiones recortadas, `(n)`, `-editado`, live photos) y unifica las copias de álbum.
   - `localMeta`: JPEG (EXIF, orientación, miniatura EXIF, SOF), PNG, HEIF/AVIF (`iinf`/`iloc`/`ipma`/`ispe`/`irot` + Exif) y MP4/MOV (`mvhd`, `tkhd`). `dateFromName` saca la fecha del nombre.
   - `compare`: índices por nombre, nombre base, tamaño y segundo de captura. Niveles en `LV`/`LV_TXT`; categorías `ambos`, `solo` y `dudoso`.
   - `vhash`: dHash de 64 bits sobre la miniatura EXIF (recortada a la proporción real) o la imagen reducida, con un margen de 48 para las zonas lisas y descarte de imágenes planas. También `visPairs` y `hamming`.
   - `dupes`: union-find por tamaño más nombre o fecha, por nombre más fecha (cuando no hay tamaño) y por pares visuales.
3. **app.js:** la interfaz.
   - Fuentes: Takeout, Picker y local. La carpeta local usa File System Access si existe; si no, `webkitdirectory`.
   - Compara y arma las filas (`buildRows`, `applyVisual`), con pestañas, miniaturas y CSV.
   - Revisar respaldos: `estado()`, selección, `apartar()` y `verApartados()`. "Por carpeta" con `carpetas()`.
   - IndexedDB `fotoscompare` v2, con los stores `fuentes`, `historial`, `apartados` y `raices`.
4. **Bloque de versión:** copiado de `jobpost/index.html` (estándar de la caja de apps).

**Detalle importante:** los accesos a carpetas (`FileSystemDirectoryHandle`) se guardan aparte, en `raices`, y **solo se leen al reconectar o deshacer**. Así, si un navegador falla al leerlos, la app igual abre. Leer de IndexedDB un handle de OPFS cuelga el Chrome headless de pruebas; con carpetas reales no debería pasar, pero quedó aislado igual.

**Takeout desde Drive:**
- `FC.driveFile(meta, token, fetch?)` imita un Blob: `size`, `slice(a, b).arrayBuffer()` y `read(a, b)` con encabezado `Range` sobre `files/{id}?alt=media`. Cuenta `bytes` y `pedidos`. Reintenta en errores de red, 429 y 5xx; con 401 avisa que el acceso venció.
- `zipEntries` funciona igual sobre ese objeto. `zipBlob` detecta `zip.remote` y usa `zipBlobRemote`: un solo pedido con encabezado local más datos (estima el largo con `nlen`/`xlen` del directorio central).
- En la app: `loadPicker` (`apis.google.com/js/api.js` → `gapi.load("picker")`), `getDriveToken` (token aparte con `drive.file`), `elegirEnDrive` (DocsView de .zip con carpetas, multiselección, `setAppId` = número de proyecto, el prefijo del ID de cliente) y `driveMeta` (`files.get` con `size`). `leerZips(files, acc, how)` es común a los .zip locales y de Drive.
- Requiere en Google Cloud: **Google Picker API** habilitada, una **clave de API** (restringida al sitio y a Picker API) cargada en el menú → "Configurar acceso a Google" (`cfg.apiKey`), y el permiso `drive.file` en la pantalla de consentimiento. Drive API ya está habilitada por multigdrive. **No se probó contra Google real.**
- El núcleo de `multigdrive` es Dart (Flutter), así que no se reutiliza código: solo el cliente OAuth y el proyecto.

**Picker:**
- Cliente OAuth web del proyecto de `multigdrive` (`414410842048-ac53…`), con Google Identity Services (token en ventana emergente).
- Permisos `photospicker.mediaitems.readonly` y `userinfo.email`. El pedido de `maxItemCount` es 2000.
- Las miniaturas y la huella visual usan `baseUrl` con el encabezado Authorization; **no se probó contra Google real** si lo permite (CORS).

## Cómo se construye y se prueba

Las fuentes de trabajo no están en el repo: el `index.html` publicado es la fuente de verdad. Para editar:
- Separá las partes (`<script>` del núcleo, `<script>` de la app) o editá el `index.html` directo.
- Mantené `core.js` sin referencias a la interfaz.

Pruebas que se usaron, con Chrome headless (Playwright) y archivos generados con Python (PIL y ffmpeg):
- **Takeout de ejemplo:** `.json` con `supplemental-metadata`, nombres recortados, `(1)`, `-editado`, la misma foto en un álbum y una versión ZIP64.
- **Carpeta local:** EXIF con huso horario, foto girada (orientación 6) con miniatura con barras, un video MP4 y un HEIC sintético.
- **Picker:** respuestas simuladas con `page.route` y un objeto `google.accounts.oauth2` falso.
- **Apartar y deshacer:** sobre OPFS, reemplazando `window.showDirectoryPicker` por `navigator.storage.getDirectory()`. Servir con `python3 -m http.server`, porque OPFS e IndexedDB no funcionan en `file://`.

## Pendientes y limitaciones conocidas

- La comparación visual no cubre HEIC ni RAW (el navegador no los decodifica) ni videos.
- En el celular (web) no se pueden mover archivos: solo se revisa y se exporta el CSV. Para apartar en el celular haría falta la APK.
- Takeout no exporta la "Carpeta bloqueada".
- Un índice guardado de Takeout no tiene los archivos: para la huella y las miniaturas hay que volver a cargar los .zip.
