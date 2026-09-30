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
