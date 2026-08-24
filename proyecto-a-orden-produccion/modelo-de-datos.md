# Proyecto A — Modelo de datos dimensional: pedido de insumos de sombrerería

> Modelo estrella en Power BI sobre un pedido real de producción (17 modelos, 13,010 sombreros),
> con 13 medidas DAX. Datos anonimizados; las columnas `sim_` son simuladas y están declaradas abajo.

---

## 1. Problema de negocio

Cuando entra un pedido de N sombreros, el área de compras necesita saber como motor principal para un requerimiento, una ficha técnica de cada modelo. Que desglosa cada modelo en todos sus componentes para tener un subtotal de insumo en su unidad para satisfacer dicho pedido. 
Hoy eso se resuelve con el analista realizando la explosión de insumos individual del modelo, utilizando plantillas en excel y asistente IA para entregar a los responsables de poner pedido de requerimiento, con tablas y cantidades corroboradas.
Este modelo permite ingresar la ficha técnica y obtener de manera rápida dicho requerimiento total, así como KPIs necesarios para la sostenibilidad del negocio y futuras inversiones.

---

## 2. Origen de los datos

Hoja de pedido de insumos real de una fábrica de sombreros (León / San Francisco del Rincón, Gto.),
pedido 150: 17 modelos, 13,010 piezas.

Anonimización — script `anonimizar_y_aumentar.py`, reproducible, semilla fija `13010`:

| Qué | Cómo |
|---|---|
| Cliente | Nombre y siglas reemplazados por `CLIENTE_A` / `CA` |
| Modelos | Renombrados a `<LÍNEA> M01…M17` conservando línea de calidad y segmento |
| Hormas | Renombradas a `HORMA_A`…`HORMA_F` |
| Costos y precios | Simulados; conservan la **escala relativa** real, no los valores absolutos |

El mapeo nombre-real → nombre-anónimo **no vive en este repositorio**.

Tres familias se agregaron al archivo original porque no estaban listadas y sin ellas el análisis de
costo describe el producto equivocado:

- **CAMPANA** — implícita en el nombre del modelo (Bangora / 5 Jap / Telar / Lona). Es el ~89 % del costo de material.
- **ETIQUETA** — juego frontal + lateral, aplica a las 13,010 piezas.
- **HERRAJE** — juego de níquel; posteriormente **plegado dentro de la toquilla** (ver §8).

---

## 3. Aviso de datos simulados

Las columnas con prefijo `sim_` — consumo real, unidades producidas, fecha de registro, costo unitario
y precio de venta — son simuladas. Los modelos, hormas, insumos, códigos, cantidades, requerimientos y
la estructura de armado son reales y anonimizados. Los costos siguen la escala relativa real del producto
(la campana ≈89 % del costo de material; el resto en orden descendente: pieles, tafilete, toquilla, herraje,
sello, etiqueta) pero sus valores absolutos no son precios de proveedor. Los precios de venta por línea son
los reales del catálogo. La simulación existe para demostrar el modelo dimensional y las medidas DAX;
al incorporar la liquidación real de la orden, estas columnas se sustituyen sin cambiar el modelo.

---

## 4. Grano

Una fila de `f_consumo` representa el requerimiento y el consumo de **un insumo, para un modelo,
dentro de un pedido**.

Esa frase implica exactamente 85 filas: 17 modelos × 5 familias de insumo, porque cada modelo lleva
exactamente un insumo de cada familia.

---

## 5. Diccionario de tablas

### `f_consumo` — hechos (85 filas)

| Columna | Rol | Nota |
|---|---|---|
| `id_pedido` | dimensión degenerada | vive en los hechos, no tiene dimensión propia |
| `id_modelo` | FK → `d_modelo` | |
| `id_insumo` | FK → `d_insumo` | resuelto por **(familia, código)**, no por código solo |
| `sim_fecha_registro` | FK → `d_calendario` | simulada |
| `cantidad_requerida` | **medida** | calculada: sombreros × `factor_consumo` |
| `sim_consumo_real` | **medida** | simulada |
| `sim_unidades_producidas` | **medida** | simulada, **semi-aditiva** (ver medida 6) |

### `d_modelo` — dimensión (17 filas)

`id_modelo · modelo · horma · linea_calidad · color · segmento · cantidad_sombreros · sim_precio_venta`

`cantidad_sombreros` y `sim_precio_venta` viven **solo aquí**, no en los hechos. Es deliberado (ver §8).

### `d_insumo` — dimensión (21 filas)

`id_insumo · familia · codigo · insumo · unidad_base · factor_consumo · sim_costo_unitario · stock_inicial · pedido_pendiente · tipo_abastecimiento · composicion`

### `d_calendario` — dimensión de fecha

`Date · año · mes · nombre_mes · trimestre` — la llave es `Date`.

### Relaciones

Tres, todas `1:*` y de **dirección única**, de la dimensión hacia los hechos:

| De | A |
|---|---|
| `d_modelo[id_modelo]` | `f_consumo[id_modelo]` |
| `d_insumo[id_insumo]` | `f_consumo[id_insumo]` |
| `d_calendario[Date]` | `f_consumo[sim_fecha_registro]` |

---

## 6. Diagrama

![Modelo](modelo.png)

`f_consumo` al centro, las tres dimensiones alrededor. Las tres relaciones son `1:*` y de dirección
única hacia los hechos; ninguna línea se cruza, que es como se comprueba a simple vista que el
esquema es estrella y no copo de nieve.

---

## 7. Medidas

Las 13 viven en una tabla vacía `_Medidas` para no quedar dispersas entre las tablas de datos.

```dax
-- 1. Vive en d_modelo (una fila por modelo), así que SUM es seguro.
Sombreros Pedidos = SUM ( d_modelo[cantidad_sombreros] )

-- 2. Sin filtrar unidad, este número mezcla dm², metros, rollos y UN. Ver Hallazgo 1.
Requerimiento Total = SUM ( f_consumo[cantidad_requerida] )

-- 3.
Consumo Real = SUM ( f_consumo[sim_consumo_real] )

-- 4. Negativa = se consumió menos de lo requerido (la orden no ha terminado).
Variacion Consumo = [Consumo Real] - [Requerimiento Total]

-- 5. DIVIDE protege el denominador cero; "/" devolvería infinito.
% Variacion Consumo = DIVIDE ( [Variacion Consumo], [Requerimiento Total] )

-- 6. Semi-aditiva: el valor se repite en las 5 filas del modelo.
--    SUM directo daría 51,095 en vez de 10,219.
Unidades Producidas =
SUMX (
    VALUES ( d_modelo[id_modelo] ),
    CALCULATE ( MAX ( f_consumo[sim_unidades_producidas] ) )
)

-- 7.
% Avance Produccion = DIVIDE ( [Unidades Producidas], [Sombreros Pedidos] )

-- 8. Tiene que ser SUMX: el costo unitario varía por insumo, así que se multiplica
--    fila por fila ANTES de sumar. SUM(req) * SUM(costo) no significa nada.
Costo Material Requerido =
SUMX ( f_consumo,
       f_consumo[cantidad_requerida] * RELATED ( d_insumo[sim_costo_unitario] ) )

-- 9.
Venta Potencial =
SUMX ( d_modelo, d_modelo[cantidad_sombreros] * d_modelo[sim_precio_venta] )

-- 10.
Margen Material = [Venta Potencial] - [Costo Material Requerido]

-- 11.
% Margen Material = DIVIDE ( [Margen Material], [Venta Potencial] )

-- 12. ALL limpia el filtro de familia en el denominador: el porcentaje sigue
--     siendo sobre el total aunque el visual esté segmentado.
% Costo Campana =
DIVIDE (
    CALCULATE ( [Costo Material Requerido], d_insumo[familia] = "CAMPANA" ),
    CALCULATE ( [Costo Material Requerido], ALL ( d_insumo[familia] ) )
)

-- 13. Mismo patrón, sobre el eje de abastecimiento.
% Costo Armado Interno =
DIVIDE (
    CALCULATE ( [Costo Material Requerido],
                d_insumo[tipo_abastecimiento] = "Armado interno" ),
    CALCULATE ( [Costo Material Requerido], ALL ( d_insumo[tipo_abastecimiento] ) )
)
```

### Qué responde cada una

| Medida | Pregunta que contesta | Valor (sin filtros) |
|---|---|---|
| Sombreros Pedidos | ¿Cuántas piezas trae el pedido? | 13,010 |
| Requerimiento Total | ¿Cuánto material hay que tener? *(solo válido segmentado por unidad)* | 67,872.82 |
| Consumo Real | ¿Cuánto material se ha consumido? | 56,819.02 |
| Variacion Consumo | ¿Cuánto falta por consumir? | −11,053.80 |
| % Variacion Consumo | ¿Qué proporción? | −16.29 % |
| Unidades Producidas | ¿Cuántos sombreros van hechos? | 10,219 |
| % Avance Produccion | ¿Qué tan avanzada va la orden? | 78.55 % |
| Costo Material Requerido | ¿Cuánto dinero en material pide este pedido? | 3,729,822.00 |
| Venta Potencial | ¿Cuánto vale el pedido si se vende completo? | 10,400,650.00 |
| Margen Material | ¿Cuánto queda después del material? | 6,670,828.00 |
| % Margen Material | ¿Qué proporción? | 64.14 % |
| **% Costo Campana** | ¿Dónde está concentrado el costo? | **88.99 %** |
| % Costo Armado Interno | ¿Cuánto del costo pasa por armado en casa? | 1.52 % |

---

## 8. Decisiones de diseño

### 8.1 El color es atributo de `d_modelo`, no dimensión propia

Porque el color pertenece al atributo del modelo, Japones Panal Natural es un modelo diferente a Japones Panal Bicolor, se decidió así porque normalmente no son muchas las variables del color en cada modelo, la mayoría en un solo color (no lleva ni natural en el nombre).

### 8.2 `cantidad_sombreros` y `sim_precio_venta` viven solo en `d_modelo`

Porque mantener estas columnas en la table de hechos f_consumo (donde se suman cantidades) podría llevar al error de sumar 5 veces la verdadera cantidad de sombreros (porque existe 5 familias de insumo para cada modelo). El hecho de resolverlo con un SUMX no lo disimulaba, alguien podría sumarlos y sacarlo lo evita.

### 8.3 `sim_unidades_producidas` sí se queda en los hechos

Porque el modelo se basa en el registro de hechos por evento, su medida de unidades producidas sí se debe sumar por modelo para calcular avance o demás KPIs para mostrar a encargados.
Un evento viene a ser el requerimiento y consumo de un modelo por insumo por cinco familias, entonces un pedido de un modelo arroja cinco eventos.

### 8.4 El herraje dejó de ser familia propia

Primero se modeló herraje como una familia aparte, operación (joey) indicó que ese herraje en particular era parte de la toquilla final y era para solo pocos modelos, entonces la columna de herraje iba a tener en su mayoría null o "sin herraje". Aquí se ejemplifica el principio general, "ningún patrón técnico salva un modelo que contradice al negocio"
---

## 9. Hallazgos

### Hallazgo 1 — el requerimiento total mezcla unidades y no se le puede pedir a un proveedor

`Requerimiento Total` sin segmentar da 67,872.82, y ese número no significa nada: suma dm², metros,
rollos y unidades. Cuatro de las cinco familias reportan una sola unidad; **TAFILETE** se rompe en varias:

| Insumo | Unidad | Factor por sombrero |
|---|---|---|
| Cabra 300x | dm² | 8.3333 |
| Vinipiel | m | 0.03861 |
| Resorte | rollo | 1/60 |
| Espumín, Fino | UN | 1 |

Filtrado a `unidad_base = 'UN'`: **55,148**.

Que el visual mienta si es que no se le filtra la unidad base, es un número sin sentido porque no se pueden sumar diferentes unidades base, ese también sería el error de un analista al publicar 67,872.82 como KPI: no podría decir en qué unidad base está.

### Hallazgo 2 — la campana concentra el 89 % del costo de material

Que casi el 90 % del costo en la fabricación de un sombrero sea del insumo CAMPANA sugiere que el resto son menos influyentes, y si se quiere ahorrar en algo se apunta directamente a negociar el costo de la campana. En el inventario es igual: el costo de una campana perdida es altísimo en comparación con cualquier otro insumo. Es justamente que el margen de ganancia depende de la calidad del sombrero, que es donde hay más diferencia entre el costo de una calidad y de otra (BANGORA VS JAPONES VS TELAR)

### Hallazgo 3 — solo el 1.5 % del costo pasa por armado interno

Sólo en ese modelo de toquilla, sería 4 SKUs en vez de 1, hacer el mismo proceso 4 veces y demorar 4 veces lo que no pasa con las toquillas listas, donde el proceso se hace una sola vez. El peso económico lo tendrían que ver los encargados que tienen acceso a esos costos, pero dudo que sea menor al costo de hora/hombre del encargado del almacén más el ayudante, junto con el mayor riesgo de error por ser 4 veces la cantidad a realizar de todos los procesos; la única manera de justificar sería que el costo ahorrado así lo indique.

---

## 10. Lo que este modelo NO responde

- **"¿Cuántos sombreros llevan piel de cabra?"** — es una pregunta a un grano distinto.
  `cantidad_sombreros` no vive en los hechos y `d_insumo` no puede filtrar `d_modelo`
  (las relaciones son de dirección única hacia los hechos). Se resolvería conservando la columna
  en `f_consumo` y midiéndola con el mismo patrón semi-aditivo de `Unidades Producidas`, nunca con `SUM`.
- **Cualquier análisis por talla.** El archivo de insumos no la trae; el pedido real sí
  (curva `T_51`…`T_65`, dos escalas que conviven: niño y adulto). Meterla baja el grano a
  modelo × insumo × talla y regenera el dataset.
- **La liquidación real de material.** Cuando exista, sustituye las columnas `sim_` sin tocar
  una sola medida. Ese es el punto de haberlo modelado así.

---

## 11. Reproducir

1. `anonimizar_y_aumentar.py` genera `Pedido_Insumos_ANONIMIZADO.xlsx` (3 hojas: 17 / 21 / 85 filas).
2. `proyecto_a_modelo_de_datos.pbix` lo carga vía Power Query y construye las 4 tablas.
3. Números de control: `f_consumo` 85 filas · 0 nulos en `id_insumo` · `Sombreros Pedidos` = 13,010.
