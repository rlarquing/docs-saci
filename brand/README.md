# Marca SACI — Kit de logo y guía de uso

> Identidad visual oficial de **SACI — Sistema Automatizado de Control de Inventarios usando QR**.
> Concepto: **brackets de escaneo QR rodeando una caja de inventario** — el gesto núcleo
> del producto (escanear → mover stock) convertido en emblema.

![Logo horizontal](saci-logo-horizontal.png)

## Archivos del kit

| Archivo | Descripción | Uso principal |
|---|---|---|
| `saci-logo-master.png` | Emblema teal sobre blanco, 1024×1024 | Master de referencia sobre fondos claros |
| `saci-app-icon.png` | Icono teal full-bleed con glifo blanco, 1024×1024 | Icono de app (apk-saci `assets/icon.png`) |
| `saci-emblem-teal.png` | Emblema teal con fondo transparente | Fondos claros, documentos, web |
| `saci-emblem-white.png` | Emblema blanco con fondo transparente | Fondos teal, splash, headers oscuros |
| `saci-logo-horizontal.png` | Wordmark horizontal (emblema + "SACI"), teal | README, cabeceras, presentaciones |
| `saci-logo-horizontal-white.png` | Wordmark horizontal en blanco | Fondos teal u oscuros |
| `saci-favicon.ico` | Favicon multi-tamaño (16/32/48) | Navegador (web-saci) |
| `saci-splash.png` | Glifo blanco transparente 720×1080 | Splash de la APK sobre fondo teal |

Variantes integradas en los productos: favicon-32, PWA 192/512, apple-touch 180,
icono maskable (web-saci `public/`) e icono adaptativo Android (apk-saci `assets/adaptive-icon.png`).

## Paleta

| Rol | Color | Hex | Notas |
|---|---|---|---|
| **Primario** | Teal SACI | `#0F766E` | Fondo de icono/splash, acentos, botones primarios. Único color de marca |
| Superficie | Blanco | `#FFFFFF` | Fondos claros; el glifo blanco vive sobre teal |
| Texto oscuro | Teal-900 | `#134E4A` / `#115E59` | Subtítulos del wordmark en fondo claro |

Regla simple: **todo el color de marca es `#0F766E`**; el emblema es monocromo
(teal sobre claro o blanco sobre teal). No combinar con otros colores de acento.

## Reglas de uso

1. **Zona de respiro**: dejar alrededor del emblema un margen mínimo del 25 % de su lado.
2. **Tamaños mínimos**: 24 px (favicon/inline), 48 px (iconos de UI), 120 px (impresión).
3. **No** rotar, sesgar, añadir sombras externas ni recolorear el emblema.
4. **Fondos**: sobre claro usar `saci-emblem-teal` / master; sobre teal u oscuro usar `saci-emblem-white`.
5. **Icono de app**: siempre la variante full-bleed teal (`saci-app-icon`); nunca el emblema suelto.
6. **Icono adaptativo Android**: el foreground es el glifo blanco al 50 % del lienzo;
   el fondo lo aporta `backgroundColor: #0F766E` (ya configurado en apk-saci).
7. **Maskable (PWA)**: glifo contenido en la zona segura del 66 % central (ya generado).

## Tipografía del wordmark

- El wordmark usa una sans bold de amplio disponible (DejaVu Sans Bold en el kit).
- En producto (web/apk) el nombre se renderiza con la tipografía del sistema junto al
  emblema — no usar el PNG del wordmark dentro de las apps.
