# 01 — Visión y alcance

v1.0 · Fase 1

## Problema

El control de inventario manual (planillas, conteos periódicos) produce descoordinación
entre depósito y administración: stock desactualizado, pérdidas sin trazabilidad,
quiebres de stock detectados tarde y errores de tipeo al registrar movimientos.

## Solución

Cada producto (o bin/ubicación de producto) recibe una **etiqueta QR impresa**. Todo
movimiento de inventario se registra **al escanear** la etiqueta con un escáner
(cámara del teléfono, escáner USB modo teclado o la app móvil de fase 2):

1. El operario escanea el QR del producto.
2. El sistema valida el QR y propone la operación (entrada / salida / ajuste).
3. El operario confirma la cantidad; el stock queda actualizado **al instante** en el
   panel web de administración (tiempo real vía sockets).
4. Todo queda auditado: quién, qué, cuánto, cuándo, dónde y desde qué IP.

## Objetivos medibles

| # | Objetivo | Meta |
|---|---|---|
| O1 | Registrar un movimiento en ≤ 10 s (escaneo + confirmación) | vs. ~60 s manual |
| O2 | Stock disponible siempre actualizado (sin conteos para saber el stock) | tiempo real |
| O3 | Trazabilidad 100 % de movimientos (auditoría con valor anterior/nuevo) | ≥ 99 % de movimientos auditados |
| O4 | Alertas de stock bajo mínimo antes del quiebre | notificación en panel |
| O5 | Operación offline del escáner con reconciliación posterior | sincronización ≤ 7 días de antigüedad |

## Actores

| Actor | Descripción |
|---|---|
| **Administrador** | Configura catálogos, productos, usuarios, roles y menús. Alcance total. |
| **Jefe de almacén** | Gestiona su(s) almacén(es): productos, QR, movimientos y operarios de sus almacenes. Aprueba ajustes. |
| **Operario** | Registra entradas/salidas escaneando QR en los almacenes que tiene asignados. |

## Alcance Fase 1 (este repositorio y los componentes actuales)

**Incluido**
- Admin core heredado de SACP: usuarios, roles, funciones/permisos por endpoint, menús dinámicos, auditoría (trazas con diff), nomencladores dinámicos.
- Dominio inventario: almacenes, categorías, unidades, ubicaciones, productos (SKU + stock mínimo), etiquetas QR por producto con impresión PDF, movimientos (ENTRADA/SALIDA/AJUSTE/TRASLADO), stock derivado, alertas de mínimo, corte diario, dashboard básico, sync offline de movimientos.
- Panel web: login, dashboard, CRUD de productos, registro de movimientos con escaneo manual (código), gestión de QR y lotes con PDF, catálogos, administración.

**Excluido (fases siguientes)**
- App móvil de escaneo por cámara (`apk-saci`) — la fase 1 acepta código tipeado y escáner USB.
- Lotes por vencimiento, series individuales, órdenes de compra/venta, proveedores.
- Conteos físicos cíclicos asistidos, valorización/costos, multi-empresa (tenants).
- Reportes avanzados/exportación Excel, app PWA instalable del operario.

## Suposiciones

- Una instancia = una empresa (multi-almacén sí, multi-tenant no en fase 1).
- El QR es una **etiqueta reutilizable** pegada al producto/bin (a diferencia del
  ticket de SACP que se agota): su ciclo es `disponible → asignado → anulado`.
- El stock es **derivado** de los movimientos (fuente única de verdad = movimientos),
  sin contadores mantenidos a mano que puedan desincronizarse.

## Riesgos y mitigaciones

| # | Riesgo | Mitigación |
|---|---|---|
| R1 | Etiquetas dañadas/duplicadas físicamente | Anulación de QR + regeneración por lote; código único con consecutivo global |
| R2 | Movimientos duplicados por re-escaneo | Idempotencia por QR+operación y deduplicación en sync offline |
| R3 | Escaneo sin red | Sync offline con ventana de 7 días y errores ítem por ítem (heredado de SACP) |
| R4 | Dependencia de servicio externo para render de QR en la web | El PDF oficial lo genera el backend (qrcode+pdfkit); el render web es solo preview |
| R5 | Crece el volumen de movimientos | Índices por producto+almacén+fecha; agregaciones para stock (ver modelo de datos) |

## Glosario

- **QR**: etiqueta impresa con código único (`QRI-000001`) y payload JSON del producto.
- **Movimiento**: registro atómico de entrada, salida, ajuste o traslado de stock.
- **Stock derivado**: existencias calculadas agregando movimientos, no un contador editado.
- **Nomenclador**: catálogo configurable sin código nuevo (almacén, categoría, unidad, ubicación).
