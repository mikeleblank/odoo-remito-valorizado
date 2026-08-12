# 📦 Odoo 19 Studio — Remito de Movimiento Interno Valorizado

Este documento explica, paso a paso, cómo personalizar el **reporte de entrega / movimiento interno** de **Odoo 19 Online (SaaS) con Odoo Studio** para agregar:

- **Precio unitario** por producto.
- **Total por línea** = cantidad entregada × precio unitario.
- **Total general del movimiento** al final del reporte.

La solución se realiza únicamente con **Odoo Studio y QWeb**, sin Odoo.sh, sin módulos externos y sin acceso al servidor.

---

## ✅ Resultado final

El reporte termina mostrando una tabla de este estilo:

| Producto | Ordenado | Entregado | Precio Unitario | Total |
|---|---:|---:|---:|---:|
| Producto de ejemplo | 9 Unidades | 9 Unidades | $ 16.900,00 | $ 152.100,00 |

Y al pie:

```text
TOTAL DEL MOVIMIENTO: $ 152.100,00
```

---

# 1. Requisitos

Antes de comenzar necesitás:

- Odoo **19 Online / SaaS**
- Plan que incluya **Odoo Studio**
- Módulo **Inventario**
- Acceso administrativo suficiente para editar reportes con Studio
- Un movimiento interno ya creado para usar como prueba

---

# 2. Identificar el reporte correcto

Abrí un movimiento interno desde:

```text
Inventario
→ Operaciones
→ Transferencias
→ Abrir un movimiento interno
```

Luego:

```text
Studio
→ Reportes
```

El reporte estándar utilizado por Odoo para estos documentos es:

```text
report_deliveryslip
(stock.report_deliveryslip)
```

que utiliza:

```text
report_delivery_document
(stock.report_delivery_document)
```

Para trabajar de forma segura, es recomendable **duplicar el reporte** y modificar la copia.

Ejemplo de nombres generados por Studio:

```text
report_deliveryslip copy(4)
report_delivery_document copy(4)
```

---

# 3. Entender la estructura del reporte

Odoo utiliza distintas plantillas según el estado y características del movimiento.

## Movimiento NO finalizado

Si:

```python
o.state != 'done'
```

se utiliza la tabla:

```text
stock_move_table
```

basada principalmente en:

```python
o.move_ids
```

## Movimiento finalizado

Si:

```python
o.state == 'done'
```

se utiliza:

```text
stock_move_line_table
```

y, para productos sin lotes o números de serie, Odoo termina llamando a:

```text
stock.stock_report_delivery_aggregated_move_lines
```

En una copia creada con Studio puede llamarse, por ejemplo:

```text
stock_report_delivery_aggregated_move_lines copy(4)
```

Esta última plantilla es la que imprime realmente cada fila del producto en un movimiento terminado.

---

# 4. Agregar la columna "Precio Unitario"

Vamos a modificar dos lugares:

1. El **encabezado** de la tabla.
2. Las **filas** de productos.

---

## 4.1. Agregar el encabezado

Abrí:

```text
report_delivery_document copy(4)
→ Editar fuentes
```

Buscá la tabla:

```xml
<table ... name="stock_move_line_table">
```

Dentro de su `<thead>` vas a encontrar algo similar a:

```xml
<th name="th_sml_quantity" class="text-end">
    Delivered
</th>
```

Después de ese `<th>`, agregá:

```xml
<th class="text-end">
    Precio Unitario
</th>
```

Queda conceptualmente así:

```xml
<th name="th_sml_quantity" class="text-end">
    Entregado
</th>

<th class="text-end">
    Precio Unitario
</th>
```

---

## 4.2. Agregar el precio en cada línea

Abrí:

```text
stock_report_delivery_aggregated_move_lines copy(4)
→ Editar fuentes
```

La plantilla tiene una estructura similar a:

```xml
<t t-name="stock.stock_report_delivery_aggregated_move_lines_copy_4">
    <tr t-foreach="aggregated_lines" t-as="line">
        ...
    </tr>
</t>
```

Al final de cada `<tr>`, después de la celda de cantidad entregada, agregá:

```xml
<td class="text-end">
    <span
        t-out="aggregated_lines[line]['product'].lst_price"
        t-options='{"widget": "monetary", "display_currency": o.company_id.currency_id}'
    />
</td>
```

### ¿Qué hace?

```python
aggregated_lines[line]['product']
```

obtiene el producto asociado a la línea.

Luego:

```python
.lst_price
```

obtiene su **precio de venta actual**.

El widget:

```xml
"monetary"
```

hace que Odoo formatee el importe usando la moneda de la empresa.

Ejemplo:

```text
$ 16.900,00
```

---

# 5. Agregar la columna "Total"

El total de cada línea será:

```text
Cantidad entregada × Precio unitario
```

---

## 5.1. Agregar el encabezado

Volvé a:

```text
report_delivery_document copy(4)
```

Después de:

```xml
<th class="text-end">
    Precio Unitario
</th>
```

agregá:

```xml
<th class="text-end">
    Total
</th>
```

La cabecera final queda:

```text
Producto | Ordenado | Entregado | Precio Unitario | Total
```

---

## 5.2. Calcular el total de cada línea

Volvé a:

```text
stock_report_delivery_aggregated_move_lines copy(4)
```

Después de la celda del precio unitario agregá:

```xml
<td class="text-end">
    <span
        t-out="aggregated_lines[line]['quantity'] * aggregated_lines[line]['product'].lst_price"
        t-options='{"widget": "monetary", "display_currency": o.company_id.currency_id}'
    />
</td>
```

La parte final de cada fila puede quedar así:

```xml
<!-- PRECIO UNITARIO -->
<td class="text-end">
    <span
        t-out="aggregated_lines[line]['product'].lst_price"
        t-options='{"widget": "monetary", "display_currency": o.company_id.currency_id}'
    />
</td>

<!-- TOTAL DE LA LÍNEA -->
<td class="text-end">
    <span
        t-out="aggregated_lines[line]['quantity'] * aggregated_lines[line]['product'].lst_price"
        t-options='{"widget": "monetary", "display_currency": o.company_id.currency_id}'
    />
</td>
```

---

# 6. Agregar el TOTAL GENERAL DEL MOVIMIENTO

El total general conviene calcularlo en la plantilla principal:

```text
report_delivery_document copy(4)
```

y no dentro de `aggregated_move_lines`, porque esa subplantilla podría ejecutarse más de una vez en movimientos con paquetes.

Agregá un bloque que inserte el total después de:

```xml
<table name="stock_move_line_table">
```

Ejemplo:

```xml
<data>
    <xpath expr="//table[@name='stock_move_line_table']"
           position="after">

        <t t-set="total_valorizado"
           t-value="sum(move.quantity * move.product_id.lst_price for move in o.move_ids)"/>

        <div class="row mt-3">
            <div class="col-12 text-end">

                <strong style="font-size: 16px;">
                    TOTAL DEL MOVIMIENTO:
                </strong>

                <span
                    style="font-size: 16px; margin-left: 10px;"
                    t-out="total_valorizado"
                    t-options='{"widget": "monetary", "display_currency": o.company_id.currency_id}'
                />

            </div>
        </div>

    </xpath>
</data>
```

---

# 7. ¿Cómo se calcula el total general?

La expresión:

```python
sum(move.quantity * move.product_id.lst_price for move in o.move_ids)
```

recorre todos los movimientos del documento.

Ejemplo:

```text
Producto A
9 × $16.900 = $152.100

Producto B
3 × $5.900 = $17.700

Producto C
2 × $12.000 = $24.000
```

Entonces:

```text
TOTAL DEL MOVIMIENTO = $193.800
```

---

# 8. Ejemplo de resultado

El reporte final queda aproximadamente así:

```text
Movimiento interno LOC21/INT/00107

Producto                              Ordenado   Entregado   Precio Unitario        Total

cort baño tela rayada celeste/rosa
oferta                                9 Unid.     9 Unid.     $ 16.900,00       $ 152.100,00


                                                    TOTAL DEL MOVIMIENTO:
                                                    $ 152.100,00
```

---

# 9. Precio utilizado

En esta implementación usamos:

```python
product.lst_price
```

que corresponde al:

```text
Precio de venta
```

del producto.

## Importante

Este valor **no queda congelado históricamente** en el movimiento.

Ejemplo:

- El 10/04/2026 el producto vale `$16.900`.
- Más adelante cambiás el precio a `$20.000`.
- Si reimprimís un movimiento anterior, el reporte puede mostrar el **precio actual** del producto.

Si necesitás conservar el precio histórico, la solución recomendada es crear un campo con Studio y almacenar el valor del precio en el momento en que se realiza la transferencia.

---

# 10. Usar costo en lugar de precio de venta

Si el objetivo del reporte es valorizar internamente el inventario, se puede reemplazar:

```python
.lst_price
```

por:

```python
.standard_price
```

Entonces:

### Precio unitario

```xml
aggregated_lines[line]['product'].standard_price
```

### Total línea

```xml
aggregated_lines[line]['quantity']
*
aggregated_lines[line]['product'].standard_price
```

### Total general

```python
sum(move.quantity * move.product_id.standard_price for move in o.move_ids)
```

---

# 11. Problemas frecuentes

## La columna aparece vacía

Verificar que la plantilla utilizada sea:

```text
stock_report_delivery_aggregated_move_lines copy(...)
```

cuando el movimiento está en estado:

```text
Hecho / done
```

---

## El precio aparece pero no como moneda

Verificar que el `<span>` tenga:

```xml
t-options='{"widget": "monetary", "display_currency": o.company_id.currency_id}'
```

---

## Error al guardar el reporte

Revisar:

- Cierre correcto de `<td>`, `<span>`, `<xpath>` y `<data>`.
- Comillas simples y dobles.
- Que el `xpath` encuentre realmente el elemento.
- No reemplazar accidentalmente la plantilla original.

---

## Movimiento con lotes o números de serie

Esta guía modifica principalmente:

```text
stock_report_delivery_aggregated_move_lines
```

Los movimientos con lotes/series utilizan otra plantilla:

```text
stock_report_delivery_has_serial_move_line
```

Si se utilizan lotes o números de serie, hay que replicar las columnas Precio Unitario y Total también en esa plantilla.

---

# 12. Buenas prácticas

- Trabajar siempre sobre una **copia del reporte estándar**.
- Probar primero con un movimiento de prueba.
- Agregar las modificaciones de forma incremental:
  1. Precio unitario.
  2. Total línea.
  3. Total general.
- Guardar una copia del XML antes de cada cambio importante.
- No modificar `external_layout` salvo que se quiera cambiar el encabezado general de todos los documentos.
- Documentar las personalizaciones de Studio antes de actualizaciones importantes de Odoo.

---

# 13. Resumen técnico

### Producto

```python
aggregated_lines[line]['product']
```

### Cantidad entregada

```python
aggregated_lines[line]['quantity']
```

### Precio de venta

```python
aggregated_lines[line]['product'].lst_price
```

### Total línea

```python
aggregated_lines[line]['quantity']
*
aggregated_lines[line]['product'].lst_price
```

### Total general

```python
sum(
    move.quantity * move.product_id.lst_price
    for move in o.move_ids
)
```

---

## Entorno utilizado

- **Odoo 19**
- **Odoo Online / SaaS**
- **Odoo Studio**
- **Inventario**
- **Reportes QWeb**

---

