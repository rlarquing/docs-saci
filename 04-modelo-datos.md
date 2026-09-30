# 04 — Modelo de datos (MongoDB)

v1.0 · Fase 1 · Claves: PK = ObjectId `_id`; `activo: boolean` soft-delete y
`createdAt/updatedAt` automáticos en **todas** las entidades (heredan `GenericEntity`).

## Nomencladores (catálogos dinámicos)

Heredan `GenericNomencladorEntity`: `nombre` (2–100, único), `descripcion` (≤ 500).
Se registran en `NomencladorTypeEnum` + registro dinámico de repositorios.

| Colección | Uso | Campos extra |
|---|---|---|
| `nom_almacen` | Almacenes/bodegas (scoping por usuario) | — |
| `nom_categoria` | Categorías de producto | — |
| `nom_unidad` | Unidades de medida (unidad, caja, kg…) | — |
| `nom_ubicacion` | Ubicaciones físicas (rack/estante/celda) | — |

## Administración (heredado de SACP, renombres mínimos)

| Colección | Campos clave | Relaciones |
|---|---|---|
| `user` | `userName` único (4–20), `email?`, `password`+`salt` (bcrypt), `refreshToken`+`refreshTokenExp`, `resetPasswordCode?` | `roleIds[]`, `funcionIds[]`, **`almacenIds[]`** (scoping) |
| `rol` | `nombre` único (solo ADMINISTRADOR / JEFE_DE_ALMACEN / OPERARIO) | `userIds[]`, `funcionIds[]` |
| `funcion` | permiso = grupo de endpoints + menú | `endPointIds[]`, `roleIds[]`, `menuId?` |
| `end_point` | `controller`, `servicio`, `ruta`, `nombre`, `metodo` | autodescubiertos con ts-morph |
| `menu` | `label`, `icon`, `to`, `menuId?` (jerarquía), `tipo` | seeds de SACI |
| `log-history` | `user`, `date`, `tabla`, `action` (ADD/MOD/DEL/REM), `valorNuevo`/`valorAnterior` (JSON), `registroId`, `direccionIp` | auditoría |

## Inventario

### `producto`
| Campo | Tipo | Reglas |
|---|---|---|
| `codigo` | string | **SKU único** formato `PRD-000001`, consecutivo global |
| `nombre` | string | requerido, 2–100 |
| `descripcion` | string? | ≤ 500 |
| `categoriaId` / `categoriaNombre` | ObjectId / string | nomenclador; nombre denormalizado para listados |
| `unidadId` / `unidadNombre` | ObjectId / string | nomenclador |
| `stockMinimo` | number | ≥ 0 (alerta cuando stock < mínimo) |
| `imagenUrl`? | string | fase 2 |

### `qr` (etiqueta de producto, heredada y adaptada)
| Campo | Tipo | Reglas |
|---|---|---|
| `codigo` | string | único, `QRI-000001`, consecutivo global (`$max` aggregation) |
| `numeroConsecutivo` | number | para orden de impresión |
| `productoId` + `productoNombre`, `productoCodigo` | ObjectId + strings | denormalizados (payload de la etiqueta) |
| `almacenId` + `almacenNombre` | ObjectId + string | almacén destino del lote |
| `contenido` | string JSON | payload codificado en la imagen |
| `loteId` | ObjectId | lote de impresión |
| `estado` | enum `disponible \| asignado \| anulado` | `anulado` terminal; **la etiqueta es reutilizable** |

### `movimiento_inventario` (corazón del sistema)
| Campo | Tipo | Reglas |
|---|---|---|
| `tipo` | enum `ENTRADA \| SALIDA \| AJUSTE \| TRASLADO` | TRASLADO = par compensado (salida en origen + entrada en destino) |
| `productoId` + `productoNombre`, `productoCodigo` | ObjectId + strings | denormalizados |
| `cantidad` | number | > 0, ≤ 999999.99 |
| `almacenId` | ObjectId | almacén del movimiento (origen en SALIDA, destino en ENTRADA) |
| `almacenDestinoId`? | ObjectId | solo TRASLADO |
| `qrId`? / `qrCodigo`? | ObjectId / string | etiqueta escaneada (opcional: ajuste manual sin QR) |
| `usuarioId` / `userName` | ObjectId / string | quién lo registró |
| `fecha` | Date | no futura; offline: antigüedad ≤ 7 días |
| `observaciones`? | string ≤ 300 | motivo del AJUSTE (requerido en ajustes negativos) |
| `saldoResultante`? | number | informativo, calculado al registrar |

### `registro_diario`
| Campo | Tipo |
|---|---|
| `fecha` (date), `almacenId`, `estado` (`abierto\|cerrado`) | corte por almacén |
| `totalEntradas`, `totalSalidas` | acumulados del día |
| `detalleCategorias[]` | JSON `[{categoria, entradas, salidas}]` |

## Índices (además de los únicos ya declarados)

```
movimiento_inventario: {productoId:1, almacenId:1, fecha:-1}   · stock y kardex
movimiento_inventario: {almacenId:1, fecha:-1}                  · registro diario / BI
qr: {loteId:1, numeroConsecutivo:1} · {estado:1}                · lotes y disponibles
producto: {categoriaId:1}                                       · filtro por categoría
```

## Cálculo de stock (fuente única de verdad)

```
stock(producto, almacen) = Σ ENTRADA + Σ AJUSTE(positivo)
                         − Σ SALIDA − Σ AJUSTE(negativo)
```

- Consulta por **agregación** sobre `movimiento_inventario` (no hay contador editable).
- El stock del TRASLADO queda implícito: el par compensado se anula en agregación.
- Alerta de mínimo: `stock < producto.stockMinimo` (endpoint dedicado + tarjeta en dashboard).
