# 08 — Análisis de apps de inventarios y backlog de funcionalidades

> **Versión**: 1.1 · **Fecha**: 2026-10-01 · **Estado P1: IMPLEMENTADO** (api `56a3447`, web `d4fd044`, apk `aa02530`)
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

### P2 — alertas y velocidad de operación
| # | Funcionalidad | Repos | Notas |
|---|---|---|---|
| 4 | **Safety stock + punto de reorden por producto/almacén** con push | api + web + apk | Generaliza el "stock mínimo" actual a niveles configurables por ubicación (patrón BoxHero). Push vía expo-notifications + email digerido diario |
| 5 | **Modo ráfaga (multi-scan) en la APK** | apk | Tras cada escaneo exitoso, volver directo a cámara sin modal de confirmación (contador acumulado en pantalla); confirmación por lote al final. Ideal para entradas masivas de recepción |
| 6 | **Timeline por producto/etiqueta** | web (+apk) | Vista cronológica legible del historial de movimientos de un producto o etiqueta concreta (los datos ya existen: es solo presentación + filtro) |

### P3 — profundidad de catálogo (fase 2+ del plan)
| # | Funcionalidad | Repos | Notas |
|---|---|---|---|
| 7 | **Lotes y caducidad por movimiento** | api + web + apk | Ya está anunciado en 07-plan-fases; añade alertas de próximo a vencer |
| 8 | **Variantes de producto** | api + web | SKU padre + variantes (talla/color); requiere migración de catálogo y de etiquetas |
| 9 | **Ubicación interna (bin) por producto** | api + web + apk | El nomenclador de ubicaciones ya existe; falta el vínculo producto→bin por almacén y su reflejo en la ficha del escaneo |

## 5. Conclusiones

1. SACI **ya cubre el corazón profesional** del dominio (etiquetas QR reutilizables, movimientos
   auditables con traslado, stock derivado, offline-first, RBAC) — lo que lo diferencia de
   herramientas ligeras como Sortly free.
2. La brecha perceptible frente a los líderes está en **conteo cíclico**, **reportes/exportes**,
   **fotos** y **alertas proactivas con push** — de ahí el P1/P2.
3. Todo el backlog es implementable **sin romper** los invariantes actuales (movimientos
   inmutables, stock derivado, sync por lotes); el conteo cíclico es el único que añade un
   documento de dominio nuevo, y se apoya en el mecanismo de ajuste existente.

## 6. Nota de despliegue del P1

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
