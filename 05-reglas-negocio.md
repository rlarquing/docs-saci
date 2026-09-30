# 05 — Reglas de negocio

v1.0 · Fase 1

## Ciclo de vida del QR (etiqueta de producto)

```
generado (lote) ──▶ disponible ──▶ asignado ─┐
                     │                        │  (reutilizable: cada escaneo
                     └── anulado ◀────────────┘   valida y registra movimiento)
                        (terminal, individual o masivo)
```

- **Generación por lote**: `producto + almacen + cantidad (1–1000)`; consecutivo global
  atómico vía `$max`; contenido JSON = `{productoCodigo, productoNombre, almacen}`.
- **Asignación**: al primer uso el QR pasa a `asignado` (queda pegado al producto/bin).
- **Reutilizable**: a diferencia del ticket de SACP, la etiqueta NO se agota con el uso;
  cada escaneo valida y genera un movimiento nuevo.
- **Anulación**: individual o masiva; un QR anulado no valida nunca.
- **Impresión**: PDF de tarjetas (grilla letter, 180 pt) generado por el backend.

## Validación de escaneo — `GET /qr/validar/:codigo`

El endpoint decide qué puede hacer el operario (misma mecánica que SACP):

| Estado QR | Resultado |
|---|---|
| `disponible` | `puede_entrada: true` (primera asignación) |
| `asignado` | `puede_entrada`, `puede_salida`, `puede_ajuste` según stock y almacén del usuario |
| `anulado` | inválido (409) |
| Sin datos de producto en payload | inválido (etiqueta de formato viejo) |

## Semántica de movimientos

### ENTRADA (10 pasos, hereda el esqueleto de SACP)
1. Validar acceso del usuario al `almacenId` (`user.almacenIds`).
2. QR existe y no está anulado.
3. Producto del QR existe y activo.
4. Cantidad > 0.
5. Fecha efectiva válida: no futura; si viene del sync offline, antigüedad ≤ 7 días.
6. Guard: no registrar entradas en un día **cerrado** (registro diario).
7. Crear movimiento con denormalizados (producto, usuario).
8. Marcar QR `asignado` (primera vez) — idempotente.
9. Acumular en `registro_diario` del día (`findOrCreate`).
10. Emitir `movimiento:change`. Si 8–9 fallan → `revertirEntrada()` (compensación).

### SALIDA
- Misma validación de acceso + QR asignado.
- **Regla de oro: no se puede dejar stock negativo** — se calcula el stock actual y se
  rechaza (409) si `cantidad > stock`.
- Solo se puede salir de un almacén con día abierto.
- `observaciones` recomendado (destino/consumo).

### AJUSTE
- Cantidad con signo (positiva = sobrante, negativa = faltante del conteo físico).
- `observaciones` **obligatoria** (motivo del ajuste).
- Solo JEFE_DE_ALMACEN o ADMINISTRADOR (el operario no ajusta).

### TRASLADO (entre almacenes del usuario)
- Crea **par compensado**: SALIDA en origen + ENTRADA en destino (mismo producto,
  misma cantidad, referenciados entre sí por `trasladoId`).
- Compensación manual si el segundo paso falla (`revertirTraslado`): queda el primer
  movimiento desactivado y el QR libre — patrón SACP sin transacciones multi-doc.
- Ambos almacenes con día abierto.

## Stock

- **Derivado por agregación** (nunca editable a mano): ver modelo de datos.
- `GET /movimiento-inventario/stock?productoId&almacenId` (ambos opcionales) → stock
  actual por producto/almacén.
- **Bajo mínimo**: `GET /movimiento-inventario/bajo-minimo?almacenId` → productos con
  `stock < stockMinimo` (tarjeta de alertas del dashboard).
- El `saldoResultante` de cada movimiento es informativo (congelado al registrar),
  nunca fuente de verdad.

## Corte diario (`registro_diario`)

- `findOrCreate` por (fecha, almacén) en cada movimiento del día.
- Acumulados: `totalEntradas`, `totalSalidas`, `detalleCategorias[]`.
- Cierre: manual (JEFE) o **cron 00:00** (`cierreDiarioAutomatico`) + backstop para
  días anteriores. Un día cerrado rechaza movimientos de esa fecha (409).

## Sync offline (escáner)

- `POST /api/sync` recibe `movimientosPendientes[]` (`{id, operacion, data, createdAt}`),
  procesa **ítem por ítem** con el servicio de movimientos y devuelve errores
  específicos (`SyncErrorDto`) + catálogos refrescados (productos y stock del almacén).
- Ventana de antigüedad ≤ 7 días (anti-backdating); duplicados rechazados por
  idempotencia (`id` del pendiente registrado).

## Matriz de roles

| Operación | ADMINISTRADOR | JEFE_DE_ALMACEN | OPERARIO |
|---|---|---|---|
| Escanear (entrada/salida) | ✔ | ✔ (sus almacenes) | ✔ (sus almacenes) |
| Ajuste de stock | ✔ | ✔ (sus almacenes) | ✖ |
| Traslado | ✔ | ✔ (sus almacenes) | ✔ si ambos asignados |
| Productos / QR / lotes | ✔ | ✔ (sus almacenes) | ✖ (solo consulta) |
| Cierre de día | ✔ | ✔ (sus almacenes) | ✖ |
| Usuarios / roles / funciones / menús | ✔ | ✖ | ✖ |
| Gestión de operarios | ✔ | ✔ solo OPERARIO de sus almacenes | ✖ |
| Auditoría (trazas) | ✔ | ✔ (solo lectura) | ✖ |

## Auditoría

- Toda operación de negocio escribe en `log-history`: acción (ADD/MOD/DEL), tabla,
  valores anterior/nuevo en JSON, usuario e IP. Solo producción (en dev se omite para
  no ensuciar las pruebas, igual que SACP).
