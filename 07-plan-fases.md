# 07 — Plan de fases

v1.0

## Fase 1 — Núcleo operativo (ESTE ENTREGABLE)

**Objetivo**: operar inventario con QR usando el panel web (escaneo por código tipeado
o escáner USB modo teclado) y la plataforma completa de administración.

| # | Épica | Incluye | Criterio de aceptación |
|---|---|---|---|
| 1 | Plataforma | API + Mongo + web + login + RBAC + menús + auditoría | login devuelve menús por rol; guard de rutas funciona |
| 2 | Catálogos | almacenes, categorías, unidades, ubicaciones (nomencladores) | CRUD genérico operativo sin código nuevo |
| 3 | Productos | SKU `PRD-000001`, categoría, unidad, stock mínimo | alta/búsqueda por código; filtro por categoría |
| 4 | Etiquetas QR | generación por lote, tarjetas, PDF, anulación, reutilización | PDF imprimible; validar-QR responde por estado |
| 5 | Movimientos | ENTRADA/SALIDA/AJUSTE/TRASLADO + stock derivado + bajo mínimo | no admite stock negativo; traslado compensado; kardex paginado |
| 6 | Corte diario | acumulados por almacén/categoría + cierre manual y cron | día cerrado rechaza movimientos |
| 7 | Dashboard | KPIs, alertas, comparativa, tendencia | todo scoping por `user.almacenIds` |
| 8 | Sync offline | endpoint ítem-por-ítem con errores y ventana 7 días | reintento del pendiente no duplica movimiento |
| 9 | Operación | Docker API+Mongo, build web, `.env.example` documentados | levantar desde README en ≤ 10 min |

## Fase 2 — Escaneo y operación de campo

- **Backlog P2 del análisis de apps — ADELANTADO E IMPLEMENTADO** (doc 08 y contratos en doc 06):
  niveles de stock por producto/almacén (safety stock + punto de reorden, módulo `/admin/niveles`
  y umbral efectivo en bajo-mínimo/BI), push de alertas (evento socket `notificacion` con campana
  en la web y notificaciones locales `expo-notifications` en la APK), modo ráfaga multi-scan en la
  APK (acumulador de sesión + confirmación por lote) y timeline por producto (filtro `productoId`
  del kardex + `show` de producto en web + pantalla `historial` en APK). Digerido diario por email
  opcional (`EMAIL_DIGEST=true`).
- **apk-saci (móvil) — NÚCLEO ENTREGADO** ([`rlarquing/apk-saci`](https://github.com/rlarquing/apk-saci),
  Expo/React Native sobre la plataforma de `apk-sacp`): escáner por cámara con
  captura de cantidad, registro manual por SKU offline, cola offline en SQLite
  (ledger + pendientes con reintentos), sync por lotes contra `POST /api/sync`
  (ítem-por-ítem con errores) y refresco de catálogos por GETs cuando no hay
  pendientes, stock cacheado para pre-validación de salidas, resumen del día con
  merge conservador y alertas de bajo mínimo, login offline con credenciales
  hash y auto-refresh de tokens, panel de administración de caché.
- Pendiente fase 2 en la APK: build firmado (EAS con URL real del API), assets
  definitivos de marca (logo) y ajustes/traslados desde el móvil (hoy son del panel
  web, el sync solo acepta `entrada|salida`).
- Reset de contraseña seguro (código con expiración + no-enumeración) y plantillas de
  correo con el adapter oficial de `@nestjs-modules/mailer`.
- Render local de QR en la web (fin de la dependencia de preview externo).
- Etiquetas con logo del cliente. *(Imágenes de producto y exportes CSV/Excel de
  kardex/stock: **ADELANTADOS — ya implementados** en el backlog P1, ver doc 08 y doc 06.)*

## Fase 3 — Inventario avanzado

- ~~Lotes por vencimiento y series individuales (número de serie por unidad).~~
  **Lotes/caducidad — IMPLEMENTADO** (backlog P3 del análisis de apps, ver doc 08 y doc 06):
  `lote` + `fechaCaducidad` opcionales e inmutables en cada movimiento, stock derivado por lote
  con estado VENCIDO/PROXIMO/OK, alertas en la web (`/admin/lotes` + menú «Lotes y vencimientos»),
  sección en el digest diario, card «Próximos a vencer» en la APK y CSV de lotes.
  *(Las series individuales por unidad siguen abiertas.)*
- **Variantes de producto — IMPLEMENTADO** (backlog P3): SKU padre + variantes con atributos
  (talla/color/modelo); cada variante es un producto completo con SKU/stock/QR propios (`productoPadreId`),
  gestión desde la ficha del producto en la web y columna «Variante» en el listado.
- **Ubicaciones internas (bin) por producto — IMPLEMENTADO** (backlog P3): vínculo producto→bin
  por almacén (`producto_ubicacion`), gestión en la ficha web, columna Bin en stock y CSV,
  y reflejo en la ficha del escáner (online por resolver + offline por cache DB v5).
- Proveedores y órdenes de compra/venta con estados.
- ~~Conteos físicos cíclicos asistidos (plan de conteo + ajustes automáticos).~~
  **ADELANTADO — ya implementado** (backlog P1 del análisis de apps, ver doc 08):
  conteo por almacén con snapshot de stock esperado, conteo a ciegas opcional,
  informe de diferencias y ajustes auditables automáticos (doc 06).
- Valorización (costo promedio ponderado / FIFO) y reportes financieros.
- BI fase 2: rotación por producto, cobertura de stock, predicción de quiebres.

## Fase 4 — Escala

- Multi-empresa (tenancy), SSO opcional.
- Métricas con Prometheus/Grafana, alertamiento operativo.
- PWA instalable del operario con notificaciones push de alertas. *(El push de alertas de stock
  ya está cubierto en el P2: socket `notificacion` en web y notificaciones locales en la APK; la
  PWA instalable y el push cuando la app está cerrada siguen abiertos.)*

## Decisiones abiertas (propietario)

1. **Marca visual definitiva** (colores, logo) — el panel usa tokens CSS; el cambio es
   barato y centralizado.
2. **SKU generado vs. código del cliente** — fase 1 genera `PRD-000001`; si el cliente
   usa códigos propios (EAN), permitir código externo único.
3. ~~**Ubicaciones**: ¿rastrear stock por ubicación física en fase 2?~~ — El vínculo
   producto→bin por almacén ya está implementado (backlog P3, `producto_ubicacion`);
   el stock como dimensión del MOVIMIENTO (bin en entrada/salida) sigue abierto para fase 4+.
