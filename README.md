# 606 Express

Convierte el informe de transacciones de **QuickBooks** en el archivo de envío
**606** que exige la DGII de República Dominicana.

**→ [Abrir la herramienta](https://eghr0106.github.io/606-express/)**

## Cómo funciona

1. Sube el informe de transacciones (`.xlsx`) exportado de QuickBooks
2. Revisa los registros — la herramienta marca los que la DGII rechazaría
3. Descarga el `DGII_F_606_{RNC}_{AAAAMM}.TXT`

Todo ocurre en el navegador. **El archivo nunca sale de tu computadora**: no hay
servidor, no se sube nada a ninguna parte.

## Qué revisa antes de generar

La DGII rechaza el archivo **completo** si un solo registro es inválido, así que
la herramienta los detecta primero y explica cada caso:

| Problema | Qué significa |
|---|---|
| Sin RNC | El campo 1 es obligatorio; en QuickBooks viene en «Descripción» |
| RNC con dígitos inválidos | Un RNC tiene 9 dígitos, una cédula 11 |
| Sin NCF | Sin comprobante fiscal no se puede reportar |
| NCF con formato inusual | No sigue el patrón B/E + 10-12 dígitos |
| No es factura de proveedor | Cheques y transferencias no van en el 606 |

Cada grupo ofrece incluir o excluir en bloque, y el RNC se puede corregir a mano
en la tabla.

## Lo que la herramienta deduce

El 606 pide datos que QuickBooks no exporta. Se deducen de la cuenta contable y
**son editables** antes de generar:

- **Tipo de gasto** (campo 3) — p. ej. `512220 Combustible` → `09 Costo de venta`
- **Bien o servicio** (campos 8 y 9) — Teléfonos → servicio, Combustible → bien

## Antes de remitir

Valida el `.TXT` con la
[herramienta de pre-validación de la DGII](https://dgii.gov.do/herramientas/formularios/pre-validacion/Paginas/default.aspx)
antes de subirlo a la Oficina Virtual.

## Especificación

Implementa el [instructivo oficial del formato 606](https://dgii.gov.do/publicacionesOficiales/bibliotecaVirtual/contribuyentes/formatoEnvioDatos/Documents/4-LlenadoyEnvioFormato606.pdf):
23 campos delimitados por `|`, encabezado `606|RNC|AAAAMM|cantidad`, máximo
10,000 registros.
