# 03 — Arquitectura

v1.0 · Fase 1

## Vista de contenedores

```
┌────────────────────┐        ┌─────────────────────────────────────┐
│  web-saci          │  HTTPS │  api-saci (NestJS 11)               │
│  Next.js 16 panel  │──────▶ │  ┌─────────┐ ┌───────────────────┐  │
│  admin + stock     │  REST  │  │ api/    │ │ core/             │  │
│  (fase 2: apk-saci │  JSON  │  │ guards  │ │ services negocio  │  │
│  escáner móvil)    │◀──WS──▶│  │ ctrl    │ │ QR + PDF + Sync   │  │
└────────────────────┘ socket │  └─────────┘ └───────────────────┘  │
                               │  persistence/  shared/              │
                               └───────────────┬─────────────────────┘
                                               │ TypeORM (MongoRepository)
                                       ┌───────▼───────┐
                                       │  MongoDB 7    │
                                       └───────────────┘
```

- **web-saci**: panel de administración (Next.js App Router). Consumo REST con JWT en
  cookies + refresh preventivo; tiempo real con socket.io; menús cacheados en IndexedDB.
- **api-saci**: monolito por capas. Autodescubrimiento de endpoints para permisos.
- **MongoDB**: documento por entidad; sin migraciones (sincronización en dev, índices
  declarados en entidades).
- **apk-saci** (fase 2): reutiliza `POST /api/sync` y `GET /qr/validar` ya diseñados.

## Capas de api-saci (convención heredada de SACP)

```
src/
├── api/          controllers + guards + decorators (HTTP)
│   ├── controller/   X.controller.ts (rutas, Swagger)
│   ├── guard/        Auth, Rol, Permission, Throttler
│   └── decorator/    GetUser, Roles, Servicio, IpAddress, Public
├── core/         servicios de negocio + mappers + estrategias JWT
│   ├── service/      X.service.ts (+ qr.service, sync.service, bi.service)
│   ├── mapper/       X.mapper.ts (entity ↔ read-dto)
│   └── strategy/     jwt.strategy, refresh.strategy
├── persistence/  entities + repositories (TypeORM/Mongo)
├── shared/       dto, enums, filtros, paginación, interfaces
├── database/     database.service (registro de entidades), seeds
├── socket/       gateway Socket.IO (auth JWT, rooms por rol)
└── mail/         (fase 2: reset de contraseña con expiración)
```

Reglas de dependencia: `api → core → persistence`; `shared` transversal;
`core` nunca importa de `api`.

## Módulos

| Bloque | Módulos | Origen |
|---|---|---|
| Admin | auth, user, rol, funcion, end-point, menu, log-history | Heredado SACP |
| Catálogos (nomencladores dinámicos) | almacen, categoria, unidad, ubicacion | Heredado + nuevo |
| Inventario | producto, qr, movimiento-inventario, registro-diario, stock (agregación en movimiento) | Adaptado/nuevo |
| Soporte | sync (offline), bi (dashboard), health, socket | Heredado/adaptado |

## Patrones transversales

1. **CRUD genérico**: cada módulo extiende `GenericRepository/GenericService/GenericController`
   → filtros, búsqueda, paginación, múltiples, importación, select y soft-delete gratis.
2. **Permisos por endpoint autodescubiertos**: en dev, `lib/parse-controller` (ts-morph)
   escanea los controladores y puebla `end_point`; `PermissionGuard` compara la ruta
   invocada con las funciones del usuario. Los menús del frontend derivan del login.
3. **Auditoría**: cada service de negocio escribe `log-history` (acción, tabla,
   valorAnterior/valorNuevo, IP, usuario) en producción.
4. **Tiempo real**: los cambios de estado emiten eventos por `SocketService`:
   `movimiento:change` (dashboard/stock en vivo), `qr:estado` (gestión de lotes),
   `menu:change` (refresco de menús).
5. **Compensación manual**: Mongo (fase 1) no usa transacciones multi-documento; los
   flujos de 2 pasos (traslado, sync) siguen el patrón SACP: operación + `revertir*`
   si el segundo paso falla, con el error reportado ítem por ítem.
6. **Soft-delete** universal (`activo`); borrado real solo en rutas `/delete/real` admin.

## Seguridad

- JWT HS256 (1 h) + refresh opaco (rand-token) con expiración; throttle 60 req/min global,
  10/min login, 5/min signup.
- Contraseñas bcrypt con salt por usuario; semilla de dev `admin/Admin1234*` **solo en dev**.
- CORS whitelist en producción (fail-closed recomendado configurando `CORS_ORIGINS`).
- Pendiente fase 2: reset de contraseña con código con expiración y sin enumeración
  (el heredado de SACP se descarta deliberadamente).

## Despliegue

- `docker compose`: `mongo:7` (healthcheck `mongosh ping`) + `api` (depends_on healthy,
  healthcheck `/api/health`). Web: build standalone con `API_URL` embebida en build.
- Variables obligatorias del API: `SECRET`, `URL`, `LOGGER_LEVELS`, `NODE_ENV`, `DB_NAME`,
  `EMAIL_*` (fase 2); opcionales: `PORT`, `CORS_ORIGINS`, `SOCKET_ENABLED`, `DB_HOST/PORT`,
  `DB_SYNC`.
