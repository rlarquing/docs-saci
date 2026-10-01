# SACI — Sistema Automatizado de Control de Inventarios usando QR

![SACI](brand/saci-logo-horizontal.png)

> Plataforma para el control automatizado de inventarios mediante etiquetas QR:
> productos etiquetados, movimientos de entrada/salida/ajuste/traslado registrados
> al escanear, stock derivado de movimientos, alertas de mínimo y trazabilidad total.

**Autor**: Reynelbis Larquin Guerra · **Licencia**: propietaria

## Qué es

SACI es el gemelo de dominio de SACP (*Sistema Automatizado de Control de Parqueos*):
reutiliza su plataforma madura (RBAC de 3 niveles, auditoría, nomencladores dinámicos,
sync offline, sockets, Docker) y sustituye el dominio de parqueos por el de inventarios.

| Componente | Repositorio | Stack |
|---|---|---|
| Documentación | **docs-saci** (este repo) | Markdown |
| API backend | **api-saci** | NestJS 11 + TypeORM (MongoDB) + JWT + QR/PDF |
| Panel web | **web-saci** | Next.js 16 (App Router) + React 19 + Tailwind 4 + shadcn/Base-UI |
| App móvil | **apk-saci** | Expo SDK 57 + React Native 0.86 + SQLite (offline-first) + escáner QR |

## Documentos

| # | Documento | Contenido |
|---|---|---|
| 01 | [Visión y alcance](01-vision-alcance.md) | Problema, objetivos, actores, alcance por fases, riesgos |
| 02 | [Reutilización SACP](02-reutilizacion-sacp.md) | Análisis de api-sacp/web-sacp: qué se reutiliza, qué se adapta, qué se descarta |
| 03 | [Arquitectura](03-arquitectura.md) | Vista de contenedores, capas de la API, patrones transversales, tiempo real |
| 04 | [Modelo de datos](04-modelo-datos.md) | Entidades, campos, relaciones e índices (MongoDB) |
| 05 | [Reglas de negocio](05-reglas-negocio.md) | Ciclo de vida del QR, semántica de movimientos, cálculo de stock, matriz de roles |
| 06 | [Contratos de API](06-contratos-api.md) | Endpoints del dominio de inventario + contratos admin heredados |
| 07 | [Plan de fases](07-plan-fases.md) | Fase 1 (este repo), fase 2+ (escáner móvil, lotes/vencimientos, reportes) |
| 08 | [Análisis de apps de inventarios](08-analisis-apps-inventarios.md) | Benchmark (Sortly, BoxHero, Zoho, Odoo, conteo cíclico) + backlog priorizado — **P1 y P2 ya implementados** |

## Marca

El kit oficial del logo (master, icono de app, emblemas, wordmark, favicon, splash) y la
guía de uso están en [`brand/`](brand/README.md). Color de marca único: teal `#0F766E`.

## Arranque rápido

```bash
# API (requiere Docker para MongoDB)
git clone https://github.com/rlarquing/api-saci && cd api-saci
cp .env.example .env && docker compose up -d
npm ci && npm run start:dev        # seeds de dev crean admin/Admin1234*

# Web
git clone https://github.com/rlarquing/web-saci && cd web-saci
cp .env.example .env.local         # API_URL apuntando al API
npm ci && npm run dev              # http://localhost:3000
```

## Fase 1 — alcance incluido

- Autenticación JWT + refresh, RBAC por funciones/endpoints, menús dinámicos
- Catálogos dinámicos: almacenes, categorías, unidades, ubicaciones
- Productos con SKU, categoría, unidad, stock mínimo
- Etiquetas QR por producto (lotes, consecutivo, impresión PDF, anulación)
- Movimientos ENTRADA / SALIDA / AJUSTE / TRASLADO con stock derivado
- Alertas de stock bajo mínimo · Corte diario por almacén · Auditoría completa
- Dashboard básico (stock por almacén, bajo mínimo, movimientos del día)
