# FotosCompare 🖼️

Compara las fotos y videos de **Google Fotos** (la nube) con los de un **dispositivo** (PC con Windows o celular Android) y muestra qué está solo en la nube, qué está solo en el dispositivo, qué está en los dos y qué hay que revisar.

**App:** https://luchorcompartidos.github.io/fotoscompare/

Todo se procesa en el navegador: no se sube nada a ningún servidor.

## Fuentes de la nube

Se pueden sumar varias cuentas y combinar las dos formas:

- **📦 Takeout (biblioteca completa):** se eligen todas las partes `.zip` del export de [takeout.google.com](https://takeout.google.com) (solo "Google Fotos"), o la carpeta descomprimida.
  - Los `.zip` se leen sin descomprimirlos, con un lector propio que soporta ZIP64 (archivos de más de 4 GB).
  - De cada foto toma el nombre original, la fecha de captura y la URL de los `.json` del Takeout. Contempla `.supplemental-metadata.json`, los nombres recortados, los duplicados `(1)` y las copias editadas.
  - La misma foto repetida en la carpeta del año y en los álbumes se cuenta una sola vez, con sus álbumes.
- **🔑 Conectar la cuenta (Google Photos Picker API):** se abre Google Fotos, se eligen hasta 2000 fotos o videos y se compara en vivo.
  - Desde marzo de 2025 Google no permite que otras apps lean la biblioteca completa. Por eso el Takeout sigue siendo la única forma de comparar todo.
  - Las fotos elegidas no traen tamaño, así que su mejor nivel posible es "Fuerte".

### Configuración inicial del Picker (una sola vez)

Usa el ID de cliente web del proyecto de Google Cloud de `multigdrive`. Se puede cambiar en **⋯ → Configurar acceso a Google**. En ese proyecto:

1. **APIs y servicios → Biblioteca:** habilitar **Google Photos Picker API**.
2. **Credenciales → cliente web → Orígenes de JavaScript autorizados:** agregar `https://luchorcompartidos.github.io`.
3. **Pantalla de consentimiento / Acceso a los datos:** agregar el permiso `https://www.googleapis.com/auth/photospicker.mediaitems.readonly`. En modo prueba, la cuenta tiene que estar en **Usuarios de prueba**.

## Fuentes del dispositivo

- **PC:** "Elegir carpeta".
- **Celular:** "Elegir archivos" (abre la galería).

De cada archivo se lee:
- la fecha EXIF y las dimensiones (JPEG, PNG, HEIC/HEIF y AVIF);
- la fecha y las dimensiones de los videos (MP4, MOV, 3GP);
- si no hay fecha interna, la que figura en el nombre (`IMG_20231105_143000`, `PXL_…`, `Screenshot_…`, `WhatsApp Image … at …`);
- como último recurso, la fecha del archivo, que es menos confiable.

## Niveles de coincidencia

| Nivel | Criterio |
|---|---|
| Exacto | mismo nombre y mismo tamaño |
| Idéntico (otro nombre) | mismo tamaño y misma fecha y hora |
| Fuerte | mismo nombre (o la misma foto con otra extensión) y misma fecha y hora, o mismo nombre y mismas dimensiones |
| Visual (misma imagen) | se ve igual según la huella visual (después de usar 🧠) |
| Probable | coinciden algunos datos, pero falta alguno para confirmar |
| Dudoso | mismo nombre con fecha distinta, o misma hora con otro nombre |

Opciones:
- **Tolerancia de hora:** de 1 s a 1 min.
- **Otro huso horario:** acepta diferencias exactas de media hora, hasta 14 h.
- **Contar "Probable" como coincidencia.**

Otras funciones:
- **🔎 Verificar huella:** compara con SHA-256 el archivo local y el del Takeout.
- **🧠 Coincidencias visuales (v0.0.2):** analiza las fotos que quedaron sin pareja o dudosas con una huella visual (dHash de 64 bits). Encuentra la misma foto aunque Google la haya recomprimido, esté renombrada o no tenga fecha.
  - Usa la miniatura interna del JPEG cuando existe, con la orientación EXIF y el recorte de las barras negras. Si no hay miniatura, decodifica la imagen reducida.
  - Sensibilidad: estricta (4), normal (8) o amplia (12) bits de diferencia. Si las fechas no coinciden para nada, la pareja queda como Dudosa.
  - No cubre HEIC ni RAW (el navegador no los puede mostrar) ni videos.
- **🧬 Duplicados (v0.0.2):** grupos de copias dentro del dispositivo y dentro de la nube (por ejemplo, entre cuentas).
  - Son idénticos si coincide el tamaño y además el nombre o la fecha y hora; después de 🧠 también se suman los visuales.
  - Muestra cuánto espacio se liberaría y se exporta a CSV.
- **CSV:** exporta la pestaña actual o el resultado completo.
- **Miniaturas, filtros y búsqueda.**

## Próximas versiones (ideas)

- Coincidencia visual para videos (por fotogramas).
- Recuperar a una carpeta de la PC lo que está solo en la nube.
- Guardar el índice para no releer el Takeout.
- Leer el Takeout desde Google Drive con el núcleo de multigdrive.
- APK y .exe.
- Alta en Mi Panel y en la caja de apps.

## Versiones

Sigue el estándar de la caja de apps (`luchorcompartidos/home` → `ESTANDAR-VERSIONES.md`):

- El número va en `<meta name="app-version">`.
- El tipo de Release se indica en el commit con `[major]`, `[minor]`, `[patch]` o `[skip release]`.

**GitHub Pages:** *Settings → Pages* → **Deploy from a branch → main → / (root)**.
