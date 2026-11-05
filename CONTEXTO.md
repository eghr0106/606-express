# Contexto para retomar este proyecto

Este archivo existe para que cualquier sesión nueva (o cualquier persona) pueda
continuar sin releer el historial. Describe qué es esto, qué decisiones ya se
tomaron, qué está verificado y qué falta.

---

## Qué es

**606 Express** — una sola página HTML que convierte el informe de transacciones
de QuickBooks (`.xlsx`) en el archivo de envío **606** de la DGII (República
Dominicana).

- **En vivo:** https://eghr0106.github.io/606-express/
- **Repo:** https://github.com/eghr0106/606-express (público)
- **Todo el código está en `index.html`** (~820 líneas: HTML + CSS + JS inline).
  No hay build, no hay dependencias que instalar, no hay backend.

### Restricciones que definieron el diseño

Estas vienen del usuario y no se cambian sin preguntarle:

1. **Es un proyecto independiente.** Vive solo en este repositorio y no se
   integra con ningún otro sistema. No lo mezcles con otros proyectos.
2. **Sin datos de la empresa.** Nada de nombres, RNC reales, ni logos. El repo es
   público. El RNC lo escribe el usuario y se guarda solo en su `localStorage`.
3. **Simple:** subir → procesar → descargar. Nada más.
4. **Los problemas se marcan en rojo** y la página explica qué elegir.
5. **Amigable y responsive**, colores tenues, tipografías legibles.
6. **Todo en el navegador.** El archivo nunca sale de la computadora del usuario
   — es su contabilidad. No agregues telemetría, analytics ni subidas.

### Única dependencia externa

SheetJS (XLSX) desde cdnjs, para leer el `.xlsx`. Se carga por `<script src>`.

---

## El formato 606, en corto

- Texto plano delimitado por `|` (**no** posicional), CRLF entre líneas y **sin
  salto después de la última**, igual que la herramienta oficial.
- Encabezado: `606|{RNC empresa}|{AAAAMM}|{cantidad de registros}`
- Detalle: **exactamente 23 campos** por línea.
- Nombre del archivo: `DGII_F_606_{RNC}_{AAAAMM}.TXT`
- **Un envío cubre un solo mes.** Por eso la herramienta se niega a mezclar.
- **Un registro por comprobante (NCF).** Esto es lo que rompía todo — ver abajo.
- La DGII rechaza el archivo **completo** si un registro es inválido; de ahí que
  la revisión previa sea el corazón de la herramienta, no un adorno.

Campos que se llenan hoy: 1 (RNC), 2 (tipo ID), 3 (tipo de gasto), 4 (NCF),
6 (fecha), 8/9 (servicios/bienes), 10 (total, sin impuestos), 11 (ITBIS),
15 (ITBIS a adelantar, siempre con cifra), 20 (ISC), 21 (otros impuestos),
22 (propina legal), 23 (forma de pago, fijo `04` = crédito). Los demás van vacíos.

### La referencia: la herramienta oficial de la DGII

La salida debe coincidir con la **Herramienta Formato 606** de la DGII (Excel
con macro, «Versión 2025»). Reglas tomadas del código de su macro
(`modServicios.GenerarArchivo` y `clsFormato606Validator`):

- Total facturado (campo 10) = servicios + bienes, **mayor que 0**. Servicios,
  bienes e impuestos **nunca negativos**: las notas de crédito (B04/E34) van en
  positivo.
- NCF: `B` + (01-04, 11-15, 17) + 8 dígitos, o `E` + (31-34, 41-45, 47) + 10
  dígitos. Cualquier otro se rechaza.
- Gastos menores (B13/E43): el RNC del proveedor debe ser el de la empresa.
- ITBIS por adelantar (15) = facturado − llevado al costo; siempre con cifra.
- ISC, otros impuestos y propina van en 20, 21 y 22, fuera del total.
- La macro escribe los montos sin formato fijo (`3000`, `1345.47`); aquí se
  dejan con 2 decimales, que el instructivo también acepta.

Instructivo oficial:
https://dgii.gov.do/publicacionesOficiales/bibliotecaVirtual/contribuyentes/formatoEnvioDatos/Documents/4-LlenadoyEnvioFormato606.pdf

---

## Lo que hay que entender del informe de QuickBooks

Aquí estuvieron todos los errores. El informe **no** trae una fila por factura:

### 1. Una factura se parte en varias líneas

Claro y Altice facturan el servicio y luego los recargos como líneas separadas;
PriceSmart reparte una compra entre dos cuentas contables. Todas comparten el
mismo NCF. **La DGII exige un registro por NCF**, así que hay que sumarlas
(`consolidar()`). Sin esto el archivo sale con NCF duplicados y lo rechazan.

### 2. El RNC solo viene en la PRIMERA línea de cada factura

Las siguientes traen la columna «Descripción» vacía. Si se validan por separado,
se marcan "sin RNC" y se excluyen — y entonces **la factura se declara por menos
de lo que es**. Por eso el RNC se hereda entre líneas del mismo NCF (el mapa
`rncPorNcf` en `procesar()`). Lo mismo con el nombre del proveedor.

### 3. «Descripción» es texto libre, no un campo numérico

Trae cosas como `101001577-Servicio Claro AGOSTO 2026` o `2% Otros Impuestos`.
Juntar todos los dígitos produce RNC inventados (13 dígitos, o "2"). Hay que
extraer un número **delimitado** de 9 u 11 dígitos → `extraerRnc()`.

### 4. QuickBooks exporta fechas en MM/DD/YYYY

No DD/MM. Ver `aFecha()`.

### 5. El 606 pide datos que QuickBooks no exporta

El **tipo de gasto** (campo 3) y si es **bien o servicio** (campos 8/9) se
deducen de la cuenta contable con el arreglo `MAPEO` (18 reglas, por número de
cuenta o por nombre). El default cuando nada calza es `02` / servicio.

> ⚠️ `MAPEO` está calibrado con el plan de cuentas de **un** contribuyente. Otra
> empresa necesitará ajustarlo. Es lo primero a revisar si los tipos salen mal.
> Por eso ambos campos son **editables en la tabla** antes de generar.

---

## Estado actual: verificado contra el informe real de agosto 2026

Archivo de prueba: un informe real de agosto 2026 exportado de QuickBooks.
**No está en el repo y no debe subirse** — son datos contables reales. Quien
retome esto necesita pedirlo o exportar uno equivalente.

| Dato | Valor |
|---|---|
| Transacciones leídas | 107 |
| Comprobantes al 606 | 87 |
| Con problemas (en rojo) | 8 |
| Monto | RD$ 325,566.34 |
| ITBIS | RD$ 39,908.17 |
| Encabezado | `606|131234567|202608|87` |

Comprobado en el sitio publicado: 88 líneas (1 encabezado + 87 detalles), 23
campos en cada detalle, cero NCF duplicados, CRLF, ASCII, y el total consolidado
cuadra exactamente con la suma de las líneas de origen.

Los 8 en rojo son errores reales del contribuyente en QuickBooks, no fallos de
la herramienta: Sirena con RNC de 10 dígitos (3 facturas), Altice con un NCF de
14 dígitos (3 líneas), un cheque sin NCF, y una compra de PriceSmart sin
comprobante.

### Lo que se arregló (commit `04544fd`)

La primera versión producía un archivo que la DGII rechaza **y** que declaraba
de menos. Cuatro fallos:

1. NCF duplicados — 7 casos, uno repetido 6 veces (no consolidaba).
2. Montos incompletos — PriceSmart declaraba 843.22 en vez de 1,769.95; Roger
   Navarro 5,150 en vez de 5,900. **Faltaban RD$ 8,761 en total.**
3. RNC inventados desde el texto libre de «Descripción».
4. El período salía de la primera fila: un informe a caballo entre dos meses
   generaba un encabezado equivocado **en silencio**.

Extras del mismo commit: el campo 10 siempre lleva cifra (aunque la factura se
neutralice en 0.00), se avisa si un comprobante entra con líneas excluidas, y
dos proveedores distintos con el mismo NCF no se mezclan (la clave de
consolidación es NCF + RNC, porque cada emisor lleva su propia secuencia).

---

## Casos borde ya probados

Con el archivo real y con datos sintéticos. Todos pasan hoy:

- Factura partida en 2, 3 y 6 líneas → un solo registro, montos sumados
- RNC heredado entre líneas del mismo NCF
- RNC embebido en texto libre → se extrae el correcto
- Notas de crédito (B04/E34) → montos en positivo, como la herramienta de la DGII
- Bienes y servicios en un mismo NCF → campos 8 y 9 separados, 8+9 = 10
- Factura que se neutraliza a 0.00 → se bloquea al generar (la DGII exige total > 0)
- Líneas «10% ISC», «Otros Impuestos», «10% DE LEY» → campos 20, 21 y 22, fuera del total
- Meses mezclados → se detiene y ofrece elegir el mes
- Dos proveedores con el mismo NCF → no se mezclan
- Suma de decimales con arrastre binario (0.1+0.2+0.3) → `0.60` en el TXT
- Archivo que no es el informe esperado → mensaje claro, no se rompe

### Cómo re-verificar

```bash
# 1. servir index.html y el .xlsx desde una misma carpeta temporal
#    (el .xlsx tiene datos reales: NO lo copies al repo)
cd <carpeta temporal>
python -m http.server 8901

# 2. abrir http://localhost:8901/index.html, subir el archivo y revisar

# 3. chequeo de sintaxis del JS inline
#    extraer el contenido de <script> a un .js y: node --check ese.js
```

Para automatizar con Playwright: en `browser_evaluate` las funciones
(`procesar`, `consolidar`, `periodoDe`, `filas`) son globales y se pueden llamar
directamente.

> ⚠️ **Trampa que ya costó una revisión completa:** un helper de prueba escrito
> como `fn() ? "OK" : "FALLA"` da "OK" cuando `fn()` devuelve un string
> descriptivo, porque en JS todo string no vacío es verdadero. Toda la primera
> tanda de pruebas pasó sin probar nada. **Compara valores concretos.**

---

## Publicación

GitHub Pages sirve la rama `main` desde la raíz. Para actualizar:

```bash
git add -A && git commit -m "..." && git push origin main
# el build tarda ~30-40s; verificar:
gh api repos/eghr0106/606-express/pages --jq '.status'   # -> "built"
curl -s -o /dev/null -w "%{http_code}" https://eghr0106.github.io/606-express/
```

Después de publicar, **probar en la URL real**, no solo en local.

---

## Pendientes / ideas

Nada de esto está comprometido con el usuario — son opciones, no tareas.

- [ ] **Correrlo por el pre-validador oficial de la DGII.** Es lo único que
      falta para cerrar la validación. Se verificó estructura, cantidad de
      campos, unicidad de NCF y CRLF contra la especificación, pero la
      confirmación final la da esa herramienta. El usuario la corre.
- [ ] Formatos **607** (ventas) y **608** (anulados), si los pide.
- [x] Retenciones de ITBIS/ISR (campos 7, 12, 17, 18). QuickBooks no las trae;
      se **calculan** (objeto `RETENCION` y función `retencion()`). Se
      identifican por comprobante E41/B11 o proveedor con cédula (11 dígitos),
      criterio de la contadora. Tasas por defecto: ITBIS 100% (con cédula y NCF
      normal, solo sobre servicios). ISR según el tipo de servicio, criterio de la
      contadora: **profesional 15%** (tipo 02) y **no profesional 2%** (tipo 03);
      la clase «Servicio profesional» sale de la cuenta (honorarios, legal,
      contable…) y se cambia en la tabla. Alquiler 10% (tipo 01) y el código
      03 para el 2% son supuestos, sin confirmar. Fecha de pago: por defecto
      la del comprobante (supuesto), **editable** en la tabla en las filas con
      retención; no puede ser anterior a la factura. Tasas en `RETENCION`. Ojo: la DGII
      exige que **el tipo y el monto de retención vayan juntos**, y que si se
      informa retención haya fecha de pago (campo 7).
- [ ] Forma de pago (campo 23) está fija en `04` = compra a crédito. QuickBooks
      no la exporta en este informe. Ojo: el `04` del campo 23 (forma de pago) y
      el `04` del campo 3 (tipo de gasto, "gastos de activos fijos") son
      catálogos distintos — no los confundas.
- [ ] `MAPEO` configurable desde la interfaz, para que sirva a otra empresa sin
      tocar código.
