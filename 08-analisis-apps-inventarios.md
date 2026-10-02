# 08 — Análisis de apps de inventarios y backlog de funcionalidades

> **Versión**: 1.2 · **Fecha**: 2026-10-01 · **Estado P1: IMPLEMENTADO** (api `f6e7bfa`/`56a3447`, web `d4fd044`, apk `aa02530`) ·
> **Estado P2: IMPLEMENTADO** (api `241fc9e`, web `9826911`, apk `93b6fd8`) · **Estado P3: IMPLEMENTADO** (api `e09c4e8`, web `77dac83`, apk `74f853e`)
> **Objetivo**: benchmark de apps profesionales de gestión de inventarios (con foco en
> QR/código de barras y operación móvil) para detectar funcionalidades que eleven a SACI
> a nivel profesional, y proponer un backlog priorizado y realista contra la arquitectura actual.

---

## 1. Método

Búsquedas web (2026-10) sobre apps de referencia y sobre capacidades concretas de
operación de almacén (escaneo, offline, conteo cíclico, alertas). Fuentes principales:

- **Sortly** — documentación oficial de etiquetas QR y casos de uso (help.sortly.com, sortly.com/features)
- **BoxHero** — página de features, app móvil en App Store (boxhero.io)
- **Zoho Inventory** — comparativas y web oficial (zoho.com/us/inventory, sourceforge.net)
- **Odoo Inventory** — escaneo de códigos integrado a compras/ventas (barcodestalk.com, cleverence.com)
- **Conteo cíclico móvil** — Dynamics 365 Warehouse (multi-scan), Mobile Inventory (Google Play),
  Cleverence "10 mobile inventory apps for cycle counting", Stockount, SAP/Unvired, MAS 9
- **Open source** — InvenTree (self-hosted, QR/barcodes)

## 2. Qué hace cada referencia (funcionalidades relevantes)

### Sortly — inventario visual con QR
- Generación de **etiquetas QR/código de barras imprimibles** con plantillas por tamaño de papel/etiqueta, y vínculo etiqueta↔ítem por escaneo.
- **Inventario visual**: carpetas anidadas y **fotos por ítem** para reconocer el producto sin abrirlo.
- Alertas de stock bajo configurables + **reportes exportables**.
- Escaneo directo desde el móvil sin hardware dedicado.

### BoxHero — simplicidad + alertas proactivas
- **Safety stock por ítem** con notificación *antes* de quedarse sin stock (tiempo de reorden).
- **Alertas ajustadas por ubicación** y entrega como **notificaciones push al smartphone**.
- **Multi-ubicación** con transferencias y visibilidad por almacén.
- Web + móvil con paridad de features y actualización en tiempo real.

### Zoho Inventory — profundidad de catálogo
- **Variantes de producto** (talla/color/modelo) como despliegue del mismo SKU padre.
- **Ítems serializados y lotes con caducidad** (trazabilidad por unidad).
- **Puntos de reorden (reorder points)** por producto y ubicación.
- Transferencias entre almacenes con estados y documentos.

### Odoo Inventory — integración del ciclo completo
- Escaneo conectado a **compras → recepción → ubicación interna (bins) → ventas**.
- Recepciones parciales y ubicaciones internas (rack/estante/bin) dentro del almacén.
- Doble validación y trazabilidad completa por operación.

### Conteo cíclico móvil (Dynamics, Mobile Inventory, Stockount, SAP)
- **Asignaciones de conteo**: el operario recibe una lista de ítems/zona a contar.
- **Conteo a ciegas (blind count)**: se cuenta sin ver el stock esperado → evita sesgo.
- **Conteo por equipo simultáneo** con consolidación y **informe de diferencias** → ajuste de stock documentado.
- **Modo multi-scan/ráfaga**: escanear muchos ítems seguidos sin confirmar cada uno.
- **Conteo 100 % offline** con descarga de asignaciones y subida al reconectar.

## 3. Comparativa funcional vs SACI actual

| Funcionalidad | Sortly | BoxHero | Zoho | Odoo | Conteo móvil | SACI hoy |
|---|:-:|:-:|:-:|:-:|:-:|---|
| Etiquetas QR por producto | ✅ | ✅ | ✅ | ✅ | — | ✅ reutilizables, lotes, PDF |
| Escaneo móvil para mover stock | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ APK offline-first |
| Entradas/salidas/ajustes | ✅ | ✅ | ✅ | ✅ | — | ✅ + TRASLADO entre almacenes |
| Stock derivado y por almacén | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Alertas de mínimo | ✅ | ✅ push | ✅ | ✅ | — | ⚠️ listado, sin push ni safety stock |
| Fotos por producto | ✅ | ✅ | ⚠️ | ✅ | ⚠️ | ❌ |
| Conteo cíclico con diferencias | ⚠️ | ⚠️ | ✅ | ✅ | ✅ | ❌ (solo corte diario) |
| Exportes CSV/Excel y reportes | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Variantes de producto | ⚠️ | ✅ | ✅ | ✅ | — | ❌ |
| Lotes / caducidad / series | ⚠️ | ⚠️ | ✅ | ✅ | — | ❌ (fase 2+ en plan) |
| Ubicaciones internas (bin) | ✅ | ✅ | ✅ | ✅ | ✅ | ⚠️ nomenclador existe, sin vínculo producto→bin |
| Multi-usuario con roles | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ ADMIN/JEFE/OPERARIO |
| Offline-first en móvil | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ✅ | ✅ ledger + sync por lotes |

## 4. Backlog propuesto (priorizado, contra la arquitectura actual)

> Criterios: impacto operativo real primero, esfuerzo después, y aprovechamiento de lo ya
> construido (movimientos inmutables, stock derivado, nomencladores, sync offline, roles).

### P1 — núcleo profesional (alta rotación de uso) ✅ IMPLEMENTADO
| # | Funcionalidad | Repos | Notas de implementación |
|---|---|---|---|
| 1 | **Conteo cíclico con informe de diferencias** ✅ | api + web + apk | Módulo `conteo-inventario`: abrir (snapshot de stock esperado), líneas embebidas, cierre que recalcula stock real y genera movimientos de AJUSTE auditables; a ciegas opcional; un conteo abierto por almacén; informe CSV. Web: `/admin/conteos` (+ `show/[id]`) con menú «Conteos». APK: pantallas `conteos`/`conteo-activo` con escáner y búsqueda por SKU (online-only por diseño). Ver contratos en doc 06 |
| 2 | **Exportes CSV/Excel** de stock, movimientos y conteos ✅ | web (o api) | Endpoints `movimiento-inventario/exportar`, `movimiento-inventario/stock/exportar` y `conteo-inventario/:id/exportar` (separador `;` + BOM, patrón buffer→`@Res`). Web: botones «Exportar CSV» en kardex, stock y conteos vía descarga autenticada |
| 3 | **Fotos por producto** ✅ | api + web + apk | `foto` (data URL ≤900 KB) en la entidad; `PUT /producto/:id/foto` y servidor público `GET /producto-foto/:id` (cache 24 h); `hasFoto` en listados (nunca base64). Web: subida con compresión canvas (512px JPEG) en la ficha + miniatura en el listado. APK: foto en el modal del escáner (URL pública) |

### P2 — alertas y velocidad de operación ✅ IMPLEMENTADO
| # | Funcionalidad | Repos | Notas de implementación |
|---|---|---|---|
| 4 | **Safety stock + punto de reorden por producto/almacén** con push ✅ | api + web + apk | Nueva entidad `nivel_stock` (1 nivel activo por producto+almacén, CRUD ADMIN/JEFE, menú «Niveles») + `stockSeguridad` global en producto como fallback; punto de reorden = mínimo + seguridad. `bajo-minimo` (movimientos y BI) devuelve `puntoReorden/sugerido/estado (BAJO_MINIMO\|REORDEN)` con umbral efectivo. Push: evento socket `notificacion` al cruzar el umbral → campana en el header web (alertas activas + toast en vivo) y notificaciones locales `expo-notifications` en la APK (solo productos recién caídos, DB v4 con `niveles_stock_cache`). Digerido diario por email a los admins con cron 07:00 (`EMAIL_DIGEST=true`) |
| 5 | **Modo ráfaga (multi-scan) en la APK** ✅ | apk | En el escáner: sin modal de confirmación, cada lectura suma +1 al acumulador de sesión y rearma la cámara automáticamente (guard de 2s anti-doble-lectura, vibración y aviso); panel con ajuste +/−/quitar, SKU manual y pre-validación de stock en salidas; «Registrar lote» crea un movimiento por producto con la cantidad acumulada reutilizando el mismo flujo (online u offline→pendientes) |
| 6 | **Timeline por producto/etiqueta** ✅ | web (+apk) | Nuevo filtro `productoId` (y `almacenId`) en `GET /movimiento-inventario` (paginado, fecha DESC). Web: página `show/[id]` de producto con ficha (foto, punto de reorden global) y timeline cronológico con icono/color por tipo, saldo, QR, usuario y observaciones + refresco en vivo por socket; acceso con el botón «Ver» del listado. APK: pantalla `historial` (búsqueda por SKU del cache + timeline del servidor, online-only) accesible desde el header del dashboard |

### P3 — profundidad de catálogo ✅ IMPLEMENTADO
| # | Funcionalidad | Repos | Notas de implementación |
|---|---|---|---|
| 7 | **Lotes y caducidad por movimiento** ✅ | api + web + apk | `lote` (≤50) y `fechaCaducidad` opcionales e INMUTABLES en `movimiento_inventario` (se estampan en entrada/salida/ajuste/traslado; el traslado lleva el lote a ambas patas). **Stock por lote**: agregación por (producto, almacén, lote, caducidad) con la MISMA semántica de signos del stock global (Σ lotes = stock total); `GET /movimiento-inventario/lotes` con `estado (VENCIDO\|PROXIMO\|OK\|SIN_CADUCIDAD)` y `diasParaVencer` (ventana `diasProximo` default 30), + CSV. Web: página `/admin/lotes` (menú «Lotes y vencimientos», filtros almacén/ventana, badges semafóricos, exportar CSV) y lote/caducidad en el form de ENTRADA/AJUSTE, kardex y timeline. APK: captura opcional de lote/caducidad en la ficha del escáner (solo ENTRADA; el modo ráfaga no captura lote por diseño), badges en el historial y card «Próximos a vencer» (online). **Alertas**: lotes vencidos/por vencer (30 días) en el sync de la APK y sección «Lotes en alerta de caducidad» en el digest diario |
| 8 | **Variantes de producto** ✅ | api + web | Patrón Zoho: cada variante es un producto COMPLETO (SKU, stock, kardex y QR propios) con `productoPadreId` + `atributos (JSON)` + `atributosResumen` denormalizado; el nombre se deriva (`Padre (Talla: M · Color: Rojo)`). `POST /producto/:id/variantes` (hereda categoría/unidad/umbrales; 1 nivel de agrupación; unicidad de combinación), `GET /producto/:id/variantes`, `PUT /producto/:id/atributos`. Web: sección «Variantes» en la ficha (editor clave/valor dinámico, enlace al padre, badge de resumen) y columna «Variante» en el listado. Cero migración de etiquetas: un QR de variante es un QR de producto |
| 9 | **Ubicación interna (bin) por producto** ✅ | api + web + apk | Nueva entidad `producto_ubicacion` (1 bin ACTIVO por par producto+almacén, CRUD ADMIN/JEFE) con denormalizados; valida que el bin pertenezca al almacén. Endpoints `producto-ubicacion` (listado/por-producto/**resolver**/select-ubicaciones/POST/PUT/DELETE). Web: sección «Ubicaciones» en la ficha (combos dependientes almacén→bin, asignar/reasignar/quitar), columna «Bin» en stock y en su CSV. APK: bin en la ficha del escáner y en el panel de ráfaga (resolver online + cache `producto_ubicacion_cache` DB v5 vía `bins` del sync) |

## 5. Conclusiones

1. SACI **ya cubre el corazón profesional** del dominio (etiquetas QR reutilizables, movimientos
   auditables con traslado, stock derivado, offline-first, RBAC) — lo que lo diferencia de
   herramientas ligeras como Sortly free.
2. La brecha perceptible frente a los líderes está en **conteo cíclico**, **reportes/exportes**,
   **fotos** y **alertas proactivas con push** — de ahí el P1/P2.
3. Todo el backlog es implementable **sin romper** los invariantes actuales (movimientos
   inmutables, stock derivado, sync por lotes); el conteo cíclico es el único que añade un
   documento de dominio nuevo, y se apoya en el mecanismo de ajuste existente.
4. Con el P3, el backlog del benchmark queda **cerrado al 100 %**: lotes/caducidad, variantes
   y bins ya son parte del producto y los tres invariantes siguen intactos (lote/caducidad son
   campos del movimiento inmutable; el stock por lote es derivado; el bin viaja en el sync).

## 5b. Nota de despliegue del P3

- **Menú web**: la entrada «Lotes y vencimientos» se siembra en `crearMenuInventario()` apuntando
  a `/admin/lotes`; en producción hay que resear los menús (o insertarla manualmente) para que
  supere el guard.
- **Endpoints/permisos**: `/movimiento-inventario/lotes*` y `/producto-ubicacion/*` se autorregistran
  en dev vía `parseController`; revisar funciones/roles de usuarios no-admin tras actualizar.
- **Base de datos**: no requiere migración en MongoDB (colección `producto_ubicacion`, campos
  `lote`/`fechaCaducidad` en `movimiento_inventario` y `productoPadreId`/`atributos*` en `producto`
  se crean al vuelo; los lectores tratan ausentes como null/0).
- **APK**: la base local sube a **versión 5** (tablas cache purgadas y re-descargadas en la primera
  sync; sesión preservada). Sin módulos nativos nuevos: no hace falta regenerar binario.
- **Semántica de lotes**: Σ stock por lote = stock total por producto/almacén (misma agregación que
  el stock global). Los movimientos sin lote forman el grupo «(sin lote)». Solo se listan lotes con
  stock vivo > 0.
- **Modo ráfaga**: deliberadamente SIN captura de lote (velocidad); para recepción con lote usar el
  modo normal del escáner o la web.
- **Digest**: la sección de caducidad viaja en el mismo email del P2 (`EMAIL_DIGEST=true`); no hay
  flag separado.

## 6. Nota de despliegue del P2

- **Menú web**: la entrada «Niveles» se siembra en `crearMenuInventario()`; en producción hay que
  resear los menús (o insertar la entrada manualmente) para que `/admin/niveles` supere el guard.
- **APK**: la base local sube a **versión 4** (tablas cache purgadas y re-descargadas en la primera
  sync; sesión preservada). `expo-notifications` añade un módulo nativo → hay que regenerar el
  binario (prebuild/EAS); en Expo Go funciona sin cambios.
- **Email digerido**: desactivado por defecto. Activar con `EMAIL_DIGEST=true` en el `.env` de la
  API (requiere el SMTP ya configurado para la recuperación de contraseña).
- **Semántica de alertas**: `bajo-minimo` ahora dispara desde el **punto de reorden** (mínimo +
  seguridad). Con seguridad 0 el comportamiento coincide con el anterior (solo `stock < mínimo`).
- **Base de datos**: no requiere migración en MongoDB (nueva colección `nivel_stock` y campo
  `stockSeguridad` en `producto` se crean al vuelo; Mongo rellena `stockSeguridad=0` al vuelo en
  los lectores con `?? 0`).

## 7. Nota de despliegue del P1
- **Menú web**: la entrada «Conteos» se siembra en `menuService.crearMenuInventario()` al
  arrancar la API en dev. En producción hay que resear los menús (o insertar la entrada
  manualmente) para que la ruta `/admin/conteos` supere el guard de menús del panel.
- **Endpoints/permisos**: el guard de permisos de los nuevos endpoints se autorrega
  en dev vía `parseController`; revisar las funciones/roles asignados a los usuarios
  no-admin tras actualizar.
- **APK**: el conteo es deliberadamente ONLINE-ONLY (compara contra el stock real del
  API; no se encola ni se cachea). Sin conexión la operación se bloquea con aviso.
- **Base de datos**: no requiere migración en MongoDB (nueva colección `conteo_inventario`
  y campo `foto` en `producto` se crean al vuelo).
