# 02 — Reutilización de SACP

v1.0 · Análisis de origen: `api-sacp` (NestJS 11 + TypeORM/MongoDB) y `web-sacp` (Next.js 16).

**SACP = Sistema Automatizado de Control de Parqueos**: control de parqueo de vehículos
por QR. Su dominio es **isomorfo** al de inventarios, por lo que la plataforma completa
se reutiliza y solo el dominio se reconstruye.

## Mapeo de dominio SACP → SACI

| Concepto SACP | Concepto SACI | Naturaleza del cambio |
|---|---|---|
| `Parqueo` (nomenclador + scoping) | `Almacen` | Renombrar + scoping `user.almacenIds` |
| `TipoMedio` (nomenclador) | `Categoria` | Renombrar (catálogo) |
| — | `Unidad`, `Ubicacion` | **Nuevos** catálogos (el registro dinámico los hace baratos) |
| — | `Producto` | **Nueva** (SKU, categoría, unidad, stock mínimo) |
| `Qr` (por tipo de medio, ticket) | `Qr` (por producto, etiqueta) | Adaptar payload y ciclo de vida: `disponible→usado` pasa a `disponible→asignado` **reutilizable** |
| `Movimiento` (entrada/salida con cobro congelado) | `MovimientoInventario` (ENTRADA/SALIDA/AJUSTE/TRASLADO) | Reescribir lógica: **eliminar precios/cobro**, añadir cantidad, stock y traslados |
| `Precio` (tarifa por vigencia) | — | **Eliminar** (sin cobro en inventario) |
| `RegistroDiario` (ingresos por tipo) | `RegistroDiario` (entradas/salidas por categoría) | Adaptar acumulados |
| `Sync` (pendientes del escáner offline) | `Sync` | Heredar casi íntegro |
| `BiService` (ingresos/tarifas) | `BiService` (stock, bajo mínimo, rotación) | Reescribir métricas |

## API — reutilizable directo (copiado a `api-saci`)

- **Base de entidades**: `generic.entity` (id ObjectId, soft-delete `activo`, timestamps),
  `generic-nomenclador.entity` (nombre único + descripción) y el **registro dinámico de
  nomencladores** (`GenericNomencladorRepository` + `NomencladorTypeEnum`): añadir un
  catálogo nuevo = 1 entity + 1 línea de enum.
- **Trío genérico CRUD**: `GenericRepository` (filtros, búsqueda multi-campo, paginación),
  `GenericService`, `GenericController` (`/filtrar`, `/buscar`, `/multiple`,
  `/importar/elementos`, select) — todo módulo nuevo nace con CRUD completo.
- **Admin/seguridad completa**: `user` (bcrypt+salt, refresh opaco, jerarquía jefe→trabajador),
  `rol` (3 roles semilla), `funcion` + `end-point` (permisos por endpoint **autodescubiertos
  con ts-morph** en arranque dev), `menu` (jerárquico, seeds), `log-history` (auditoría con
  valor anterior/nuevo e IP).
- **Guards**: `AuthGuard` (JWT), `RolGuard` (`@Roles`), `PermissionGuard` (función↔endpoint),
  `ThrottlerGuard` global + throttles finos en auth.
- **Infra**: `config/config.ts` (loader estricto), `main.ts` (CORS whitelist, ValidationPipe
  global, Swagger dev-only, seeds dev), Dockerfile multi-stage no-root, docker-compose con
  Mongo 7 + healthchecks, `.env.example`, healthcheck `GET /api/health`, LoggerProvider.
- **Sockets**: gateway Socket.IO con auth JWT en handshake y rooms por rol + `SocketService`
  (RxJS Subjects) — eventos `qr:estado`, `movimiento:change`, `menu:change`.
- **QR/PDF**: generación de imágenes (`qrcode.toDataURL`) e **impresión de tarjetas en PDF**
  (`pdfkit`, grilla letter) — se reutiliza para etiquetas de producto.
- **DTOs compartidos**: `ResponseDto`, `ListadoDto` (tablas/export), `Pagination` helper,
  `filtro-generico`, `buscar`, `select`.

## API — adaptar

- `qr.service`: lote/consecutivo/estados/PDF se conservan; payload pasa a
  `{producto, almacen}`; el estado `usado` se sustituye por `asignado` (la etiqueta no se
  agota, se reutiliza); `validarQR` pasa a decidir entradas/salidas/ajustes según stock.
- `movimiento.*`: se conserva el patrón de pasos numerados, compensación manual
  (`revertir*`), ventanas offline ≤ 7 días y verificación batch; se elimina todo el
  bloque de cobro/precio congelado y se añade cantidad + tipo + traslado compensado.
- `registro-diario`: cron 00:00 y cierre manual se conservan; acumulados por categoría.
- `sync`: estructura de pendientes/errores ítem por ítem idéntica, apuntando al nuevo
  servicio de movimientos.
- `user.*`: `parqueoIds` → `almacenIds` y textos jefe-trabajador a almacén.

## API — descartar

`precio.*` (dominio de cobro), `vercel.json` + `.gitlab-ci.yml` (obsoletos), adapter de
mail roto (Handlebars sin compilar), `TypeORMExceptionFilter` inerte en Mongo,
`GenericController.getCurrentUser()` muerto, claves de entorno sin uso (`PROVINCIA`,
`CUBE_*`), especifics de parqueo y reset de contraseña sin expiración de código.

## Web — reutilizable directo (copiado a `web-saci`)

- **Kit UI**: `components/ui/*` (24 primitivas shadcn sobre Base-UI) + `globals.css` con
  tokens de marca.
- **DataTable** (TanStack Table v9): paginación server-side, orden, selección, export CSV,
  columna de acciones — núcleo de todos los listados.
- **Capa HTTP+JWT** (`utilities/`): wrapper `fetch` con **refresh preventivo**
  (decodifica `exp` en cliente, margen 30 s), deduplicación de refrescos en vuelo,
  reintento único en 401; verbos `get/post/patch/delete` con envelope `{msg, obj}`;
  cookies `cookies-next` + espejo localStorage.
- **Shell del panel**: `app/admin/components/template` (header, sidebar colapsable, guard
  por menús leídos de IndexedDB, footer) + `components/Menu/*` (sidebar jerárquico,
  dropdown usuario, logout) + `app/providers` (Toaster, PWA).
- **Login** completo (RHF+Zod, banner de errores, guardía de sesión).
- **Módulo QR** (`app/admin/qr/`): lotes, generador (cantidad 1–1000, contador de
  disponibles), detalle de lote con grid de tarjetas, selección múltiple, acciones batch,
  **PDF blob con Bearer**, y refresco en vivo por socket — se adapta a producto/almacén.
- **Nomencladores dinámicos** (`app/admin/nomenclators/[name]`) — CRUD genérico de catálogos.
- **CRUD admin**: users/roles/functions/menus + logs-history con diff — patrón CRUD
  (page+form RHF+Zod+service+adapters) que se clona para Productos/Movimientos/Stock.
- **Infra**: Dockerfile standalone, compose, `dev.sh`/`dev.ps1`, PWA (manifest + SW +
  offline), `scripts/generate-pwa-icons.mjs`.

## Web — adaptar / descartar

- **Adaptar**: renombres de dominio (`parqueo→almacen`, `tipoMedio→producto`) en modelos,
  endpoints y textos; páginas nuevas de Productos/Movimientos/Stock clonando el patrón
  CRUD; dashboard sustituye al stub `/admin`; labels del BI diferidos a fase 2.
- **Descartar**: `precios/` (dominio de cobro), código muerto (`protected-route`,
  `components/template` legacy + barrel, `app/auth/logout.ts`), rutas fantasma
  (`/auth/signin` → `/`, `/wci/dashboard` → `/admin`), `examples/` y artefactos de scaffold.
- **Render de QR en web**: fase 1 usa el servicio externo de preview (ya en
  `remotePatterns`); el PDF oficial del backend es la vía de impresión. Fase 2: render local.
