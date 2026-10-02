# 06 — Contratos de API

v1.0 · Fase 1 · Prefijo global `/api` · Respuestas estándar: `ResponseDto {id, successStatus, message}`,
listados `ListadoDto {header, key, data}`, paginación `Pagination<T> {items, meta, links}`.
Autenticación: `Authorization: Bearer <accessToken>` en todo excepto auth y health.

## Auth (heredado)

| Método | Ruta | Descripción |
|---|---|---|
| POST | `/auth/login` | `{userName, password}` → `{accessToken, refreshToken, functions, menus, almacenes, roles, userId}` |
| POST | `/auth/refresh-token` | rota refresh (opaco) |
| POST | `/auth/signup` | alta pública con rol OPERARIO (5/min) |
| POST | `/auth/logout` | invalida refresh |

## Inventario (nuevo/adaptado)

### Productos
| Método | Ruta | Notas |
|---|---|---|
| GET/POST | `/producto` · `/producto/:id` | CRUD + paginación/filtros genéricos |
| GET | `/producto/codigo/:codigo` | búsqueda por SKU (para escáner manual) |
| PATCH | `/producto/:id` | edición (categoria, unidad, stockMinimo…) |
| DELETE | `/producto/:id` | soft-delete; real en `/producto/delete/real/:id` (admin) |
| GET | `/producto/cantidad/elementos` | contador para KPIs |

### Catálogos (nomencladores dinámicos)
`/nomenclador/{almacen|categoria|unidad|ubicacion}` con el contrato genérico completo:
`/`, `/:id`, `/filtrar`, `/buscar`, `/multiple`, `/crear/select`, `/delete/real/:id`…
(agregar catálogo = 1 entrada en `NomencladorTypeEnum`; cero endpoints nuevos).

### QR
| Método | Ruta | Notas |
|---|---|---|
| POST | `/qr/generar` | `{productoId, almacenId, cantidad ≤ 1000}` → lote nuevo |
| GET | `/qr/lote` · `/qr/lote/:loteId` | listado de lotes / QRs del lote |
| GET | `/qr/lote/:loteId/pdf` | **PDF de tarjetas** (Bearer; blob para descarga) |
| GET | `/qr/validar/:codigo` | decide `puede_entrada/salida/ajuste` (consumo escáner) |
| POST | `/qr/anular/elementos/multiples` | anulación masiva |
| GET | `/qr/disponibles/:productoId` | contador para el generador |

### Movimientos de inventario
| Método | Ruta | Notas |
|---|---|---|
| POST | `/movimiento-inventario/entrada` | `{qrCodigo \| productoId, almacenId, cantidad, fecha?, lote?, fechaCaducidad?}` (P3) |
| POST | `/movimiento-inventario/salida` | valida stock suficiente (409 si excede); `lote?/fechaCaducidad?` (P3) |
| POST | `/movimiento-inventario/ajuste` | JEFE+: `{productoId, almacenId, cantidad±, observaciones!}`; `lote?/fechaCaducidad?` (P3) |
| POST | `/movimiento-inventario/traslado` | `{productoId, cantidad, almacenOrigenId, almacenDestinoId, lote?, fechaCaducidad?}` (par compensado, el lote viaja a ambas patas) |
| GET | `/movimiento-inventario` | kardex paginado; con `?productoId=&almacenId=` devuelve el **timeline** del producto (orden fecha DESC, P2); filas con `lote`/`fechaCaducidad` (P3) |
| GET | `/movimiento-inventario/stock` | `?productoId&almacenId` → stock derivado por agregación, filas con `ubicacionNombre` (bin del producto en ese almacén, P3) |
| GET | `/movimiento-inventario/bajo-minimo` | productos bajo el **punto de reorden** con umbral efectivo (`nivel_stock` > global): `{stock, stockMinimo, stockSeguridad, puntoReorden, sugerido, estado: BAJO_MINIMO\|REORDEN}` (P2) |
| GET | `/movimiento-inventario/lotes` | **stock por lote** (P3): agrega por (producto, almacén, lote, caducidad) con la misma semántica de signos; solo stock > 0; `?almacenId=&productoId=&diasProximo=30` → filas `{lote, fechaCaducidad, stock, estado: VENCIDO\|PROXIMO\|OK\|SIN_CADUCIDAD, diasParaVencer}` ordenadas vencidos→próximos→OK→sin caducidad |
| GET | `/movimiento-inventario/lotes/exportar` | CSV del stock por lote (mismos filtros) |
| POST | `/movimiento-inventario/estado` | batch `{codigos[]}` (máx 100) → estado de cada QR |

### Registro diario
`GET /registro-diario` · `GET /registro-diario/:id` (detalle con `detalleCategorias`) ·
`POST /registro-diario/cerrar/:id` (JEFE) · cron 00:00 auto-cierre.

### Sync offline
`POST /api/sync` → `{movimientosPendientes: [{id, operacion, data, createdAt}]}` →
procesa ítem por ítem, devuelve `{procesados, errores: SyncErrorDto[], productos, stock, niveles, bins, lotesProximos}`.
Los productos del sync incluyen `stockSeguridad`; `niveles` trae los `nivel_stock` de los almacenes
(del usuario, DB v4); `bins` trae los vínculos producto→ubicación de sus almacenes y `lotesProximos`
los lotes vencidos o por vencer en 30 días (P3, cacheados en DB v5).

### Dashboard (BI fase 1)
| Método | Ruta | Contenido |
|---|---|---|
| GET | `/bi/dashboard` | KPIs: stock total por almacén, movimientos hoy (entradas/salidas), productos, bajo mínimo, `lotesEnAlerta` (P3), últimos movimientos |
| GET | `/bi/comparativa-almacenes` | movimientos por almacén en rango |
| GET | `/bi/tendencia` | movimientos por día (14/30 días) |
| GET | `/bi/bajo-minimo` | consolidado de alertas |

Todos filtran por `user.almacenIds` (scoping heredado de SACP).

### Conteo cíclico (backlog P1 — implementado)
Vertical completo api+web+apk. Un conteo congela el stock esperado por producto
al abrir, se cuenta (opcionalmente a ciegas) y al cerrar genera movimientos de
AJUSTE auditables por cada diferencia contra el stock real. Reglas: **un solo
conteo `ABIERTO` por almacén**, cierre solo con **todas las líneas contadas**.

| Método | Ruta | Roles | Contenido |
|---|---|---|---|
| POST | `/conteo-inventario` | ADMIN, JEFE | Abrir conteo `{almacenId, esCiego}` → snapshot de stock esperado |
| GET | `/conteo-inventario` | ADMIN, JEFE, OPERARIO | Listado paginado (`almacenId`, `estado`, `sinPaginacion`) |
| GET | `/conteo-inventario/:id` | ADMIN, JEFE, OPERARIO | Detalle con `lineas[]` embebidas |
| PUT | `/conteo-inventario/:id/linea` | ADMIN, JEFE, OPERARIO | Registrar cantidad contada `{productoId, cantidadContada}` (0 válido) |
| PATCH | `/conteo-inventario/:id/cerrar` | ADMIN, JEFE | Recalcula stock real, crea AJUSTES (±) y congela `resumen` |
| PATCH | `/conteo-inventario/:id/cancelar` | ADMIN, JEFE | Descarta el conteo sin ajustes |
| GET | `/conteo-inventario/:id/exportar` | ADMIN, JEFE | Informe CSV del conteo (esperado/contado/diferencia/ajustado) |

La línea devuelve `{id, successStatus, message}`; al cerrar, el message resume
los ajustes generados (`sobrantes`, `faltantes`, `errores` por línea).

### Fotos de producto (backlog P1 — implementado)
| Método | Ruta | Roles | Contenido |
|---|---|---|---|
| PUT | `/producto/:id/foto` | ADMIN, JEFE | Data URL `data:image/jpeg;base64,…` (cliente comprime a ≤512px, límite 900 KB) |
| GET | `/producto-foto/:id` | **público** | Binario de la foto (`Cache-Control 24 h`); 404 si no tiene |

El listado de productos nunca devuelve el base64: entrega `hasFoto` booleano y
la imagen se consume por la URL pública. `GET /producto` ahora responde el
`ListadoDto` estándar (header/key) con la columna `hasFoto`. El sync de la APK
sigue sin incluir fotos (payload ligero; la APK las pide por URL).

### Niveles de stock y push (backlog P2 — implementado)
**Safety stock por ubicación** (patrón BoxHero): el punto de reorden = `stockMinimo + stockSeguridad`.
El umbral efectivo de un producto en un almacén es el del `nivel_stock` específico si existe;
si no, los globales del producto (`stockSeguridad` añadido a `producto`, fallback 0).

| Método | Ruta | Roles | Notas |
|---|---|---|---|
| GET | `/nivel-stock` | ADMIN, JEFE, OPERARIO | Listado paginado (`almacenId`, `productoId`, `sinPaginacion`); filas con `puntoReorden` calculado |
| GET | `/nivel-stock/:id` | ADMIN, JEFE, OPERARIO | Detalle |
| POST | `/nivel-stock` | ADMIN, JEFE | `{productoId, almacenId, stockMinimo?, stockSeguridad?}`; 1 nivel activo por par producto+almacén (409 si duplica) |
| PUT | `/nivel-stock/:id` | ADMIN, JEFE | Actualiza umbrales |
| DELETE | `/nivel-stock/:id` | ADMIN, JEFE | Soft-delete |

**Push**: al registrar un movimiento (entrada/salida/ajuste/traslado), si el stock **cruza** el punto
de reorden (antes ≥ punto, después < punto) la API emite el evento socket **`notificacion`** con
`{tipo: BAJO_MINIMO\|REORDEN, producto*, almacen*, stock, stockMinimo, stockSeguridad, puntoReorden,
sugerido, timestamp}`. La web lo consume en la campana del header (toast + lista con badge de no leídas);
la APK genera **notificaciones locales** (expo-notifications) solo para productos recién caídos bajo umbral.
**Digerido diario**: cron 07:00 (`EMAIL_DIGEST=true` en el `.env`) envía a los ADMINISTRADOR activos un
email HTML con la lista completa de productos bajo punto de reorden.

### Exportes CSV (backlog P1 — implementado)
Separador `;` + BOM UTF-8 (Excel es-ES), descarga con `Content-Disposition`.

| Método | Ruta | Contenido |
|---|---|---|
| GET | `/movimiento-inventario/exportar?almacenId=&tipo=` | Kardex completo (tope 10.000 filas) |
| GET | `/movimiento-inventario/stock/exportar?almacenId=` | Stock derivado por producto/almacén (+ columna Bin, P3) |
| GET | `/movimiento-inventario/lotes/exportar?almacenId=&productoId=&diasProximo=` | Stock por lote con estado de caducidad (P3) |
| GET | `/conteo-inventario/:id/exportar` | Informe del conteo con diferencias |

### Variantes de producto (backlog P3 — implementado)
**Patrón Zoho/BoxHero**: cada variante es un producto COMPLETO (SKU propio PRD-XXXXXX,
stock, kardex y etiquetas QR propios) enlazado a su SKU padre con `productoPadreId`;
un solo nivel de agrupación (el padre nunca tiene padre) y unicidad de combinación de
atributos bajo el mismo padre. El `nombre` de la variante se deriva (`Padre (Talla: M · Color: Rojo)`)
y el listado de productos muestra la columna `Variante` (`atributosResumen`).

| Método | Ruta | Roles | Notas |
|---|---|---|---|
| POST | `/producto/:id/variantes` | ADMIN, JEFE | `{atributos: [{clave, valor}]}` → crea la variante heredando categoría/unidad/umbrales del padre |
| GET | `/producto/:id/variantes` | ADMIN, JEFE, OPERARIO | Variantes activas del padre (ReadProductoDto) |
| PUT | `/producto/:id/atributos` | ADMIN, JEFE | `{atributos: [{clave, valor}]}` → recalcula resumen y nombre (solo variantes) |

Las etiquetas QR no cambian: una variante es un producto y sus QR referencian su SKU.

### Ubicaciones internas — bins por producto/almacén (backlog P3 — implementado)
**Patrón Sortly/Odoo**: el nomenclador `nom_ubicacion` describe los racks/estantes;
`producto_ubicacion` es el vínculo producto→bin por almacén. Regla: **una ubicación
activa por par producto+almacén** (reasignar = editar o borrar y recrear).

| Método | Ruta | Roles | Notas |
|---|---|---|---|
| GET | `/producto-ubicacion` | ADMIN, JEFE, OPERARIO | Listado paginado (`almacenId`, `productoId`, `sinPaginacion`) |
| GET | `/producto-ubicacion/producto/:productoId` | ADMIN, JEFE, OPERARIO | Bins de un producto en todos sus almacenes (ficha web) |
| GET | `/producto-ubicacion/resolver?productoId=&almacenId=` | ADMIN, JEFE, OPERARIO | Bin para la ficha del escáner → `{ubicacionNombre\|null}` |
| GET | `/producto-ubicacion/select-ubicaciones?almacenId=` | ADMIN, JEFE, OPERARIO | Combo de ubicaciones activas (filtra por almacén dueño) |
| POST | `/producto-ubicacion` | ADMIN, JEFE | `{productoId, almacenId, ubicacionId}` (409 si ya existe bin para el par) |
| PUT | `/producto-ubicacion/:id` | ADMIN, JEFE | `{ubicacionId}` reasignar |
| DELETE | `/producto-ubicacion/:id` | ADMIN, JEFE | Soft-delete |

El stock (`/movimiento-inventario/stock`), su CSV y la ficha del escáner muestran el bin;
la APK lo cachea en `producto_ubicacion_cache` (DB v5) vía `bins` del sync.

## Admin (heredado, contratos idénticos a SACP)

- `/user`, `/rol`, `/funcion`, `/menu`, `/log-history` con el CRUD genérico completo
  (`/filtrar`, `/buscar`, `/multiple`, `/importar/elementos`, `/delete/real/:id`,
  `/cantidad/elementos`).
- Roles semilla: `ADMINISTRADOR`, `JEFE_DE_ALMACEN`, `OPERARIO`.
- Menús semilla: Dashboard, Productos, Movimientos, Stock, Alertas, QR, Almacenes,
  Catálogos (categorías/unidades/ubicaciones), Registro diario, Trazas, Administración
  (usuarios/roles/funciones/menús).

## Convenciones

- Errores: códigos HTTP estándar + mensaje en español; validaciones con class-validator
  (pipe global `whitelist + forbidNonWhitelisted`).
- Soft-delete en todas las entidades (`activo`); `remove` = soft, `/delete/real` = físico.
- Auditoría automática en producción vía `log-history` en cada service.
- Swagger en `/api/docs` solo fuera de producción.
