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
| POST | `/movimiento-inventario/entrada` | `{qrCodigo \| productoId, almacenId, cantidad, fecha?}` |
| POST | `/movimiento-inventario/salida` | valida stock suficiente (409 si excede) |
| POST | `/movimiento-inventario/ajuste` | JEFE+: `{productoId, almacenId, cantidad±, observaciones!}` |
| POST | `/movimiento-inventario/traslado` | `{productoId, cantidad, almacenOrigenId, almacenDestinoId}` (par compensado) |
| GET | `/movimiento-inventario` | kardex paginado con filtros (producto, almacén, tipo, rango fechas) |
| GET | `/movimiento-inventario/stock` | `?productoId&almacenId` → stock derivado por agregación |
| GET | `/movimiento-inventario/bajo-minimo` | alertas `stock < stockMinimo` |
| POST | `/movimiento-inventario/estado` | batch `{codigos[]}` (máx 100) → estado de cada QR |

### Registro diario
`GET /registro-diario` · `GET /registro-diario/:id` (detalle con `detalleCategorias`) ·
`POST /registro-diario/cerrar/:id` (JEFE) · cron 00:00 auto-cierre.

### Sync offline
`POST /api/sync` → `{movimientosPendientes: [{id, operacion, data, createdAt}]}` →
procesa ítem por ítem, devuelve `{procesados, errores: SyncErrorDto[], productos, stock}`.

### Dashboard (BI fase 1)
| Método | Ruta | Contenido |
|---|---|---|
| GET | `/bi/dashboard` | KPIs: stock total por almacén, movimientos hoy (entradas/salidas), productos, bajo mínimo, últimos movimientos |
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

### Exportes CSV (backlog P1 — implementado)
Separador `;` + BOM UTF-8 (Excel es-ES), descarga con `Content-Disposition`.

| Método | Ruta | Contenido |
|---|---|---|
| GET | `/movimiento-inventario/exportar?almacenId=&tipo=` | Kardex completo (tope 10.000 filas) |
| GET | `/movimiento-inventario/stock/exportar?almacenId=` | Stock derivado por producto/almacén |
| GET | `/conteo-inventario/:id/exportar` | Informe del conteo con diferencias |

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
