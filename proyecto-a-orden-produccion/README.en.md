# Project A — Production order: is there enough material for 13,010 hats?

> Dimensional model and Power BI dashboard over a real order from a hat factory:
> 17 models, 13,010 pieces, 21 supply items. Data is anonymized.

**A note on the data.** The order, the models, the supply items, the consumption factors, and the bill of materials are **real and anonymized**: customer and hat blocks renamed, models coded. Columns prefixed `sim_` — actual consumption, units produced, dates, costs, and prices — are **simulated**; they preserve the product's real relative scale, not its absolute values. The costs in this repository are **not supplier prices**. The mapping between real and anonymized names does not live in any repository.

---

## Problem

When an order for N hats comes in, Purchasing needs one thing before it can order anything: the total material requirement, item by item, each in its own unit. Today that gets done by hand — an analyst explodes the bill of materials model by model in Excel templates and hands corroborated tables to whoever places the order.

This order carries 13,010 hats across 17 models. Three questions have to be answerable:

1. How much material does the order call for, and how much has been consumed?
2. How far along is production, and which models are behind?
3. Which supply item puts the order at risk?

## Approach

A **star schema**, not a flat sheet. One fact table, `f_consumo`, with **85 rows** — 17 models × 5 supply families, since each model takes exactly one item from each family — plus three dimensions: `d_modelo` (17 rows), `d_insumo` (21), and `d_calendario`. Three `1:*` relationships, all single-direction toward the facts. **13 DAX measures.**

The full write-up — table dictionary, all 13 measures annotated, the diagram, and what the model deliberately does *not* answer — is in **[`modelo-de-datos.en.md`](modelo-de-datos.en.md)**.

The design decisions that matter, and why:

- **`cantidad_sombreros` lives only in `d_modelo`, never in the facts.** Put it in `f_consumo` and someone eventually sums 65,050 hats instead of 13,010, because every model appears five times — once per supply family. Handling it with a `SUMX` would have worked and left the trap armed; removing the column defuses it.
- **`Unidades Producidas` is semi-additive.** The value repeats across the model's five rows, so it collapses with `SUMX ( VALUES ( d_modelo[id_modelo] ), CALCULATE ( MAX (...) ) )`. A plain `SUM` returns 51,095 instead of 10,219 — and it returns a number that looks entirely believable.
- **Hardware stopped being its own family.** It was modeled separately at first; operations pointed out that on this product it is folded into the hatband, and only on some models, so the column would have been mostly null. No technical pattern rescues a model that contradicts the business.
- **Color is an attribute of `d_modelo`, not a dimension of its own.** *Japonés Panal Natural* and *Japonés Panal Bicolor* are two distinct catalog models, not one model in two colors.
- **Three families were added to the source file** because they were not listed, and without them the cost picture describes a different product: the hat body (*campana*), the label, and the hardware.

## Result

![Dashboard for order 13,010](capturas/dashboard.png)

One page covering the three blocks the business asked for: production progress (13,010 ordered · 10,219 produced · **78.55 %**), requirement versus actual consumption, and cost concentration by supply item. Seven models sit below the 75 % target; the furthest behind is LONA M13, at 55 %.

But the KPIs are not what this project has to say. Three findings are — and none of them came packaged in the data.

### Finding 1 — a correctly calculated total that means nothing

`Variacion Consumo` reads **−11,053.80**, and it is calculated perfectly. It is also adding units to rolls to meters to square decimeters: `d_insumo[unidad_base]` holds four different values. Valid arithmetic over incompatible magnitudes — the equivalent of adding kilos to liters.

There were four reasonable ways out: show it as a percentage, filter to a single unit, convert it to money, or split it by unit. **It was pulled off the headline card and split by `unidad_base`**, and that column was added to the detail table as well. The dashboard *shows* why the total can't be read, instead of hiding the flaw behind a clean KPI. A dashboard that conceals a defect in the data is worse than one that declares it.

Filtered to `unidad_base = 'UN'`, the requirement is **55,148** pieces. That one you can actually send to a supplier.

### Finding 2 — the shortage alert wasn't in the data; it had to be defined

Neither obvious definition of "item at risk" discriminates at all:

| Definition | Items flagged |
|---|---|
| `stock_inicial` < requirement | **21 of 21** |
| `stock_inicial + pedido_pendiente` < requirement | **2 of 21**, and those two by rounding hundredths |

One screams at everything, the other stays silent about everything. Implement just one and the problem is invisible; put both side by side and it is obvious that neither works.

The third definition was anchored to the **physical process, not to the data**: the hat body blocks Blocking → Dope → Drying. With no hat body there is no hat to delay; with no hatband there are thousands already waiting in drying. That argument is in no column of the file — it comes from knowing the factory floor.

And it was implemented as a **ranking by money at risk**, not as a hard filter on `familia = CAMPANA`. The hat body climbs to the top on its own because that is where the cost sits. Pinning it by hand would have produced the identical visual today and a dead alert the day the data changes.

### Finding 3 — the hat body concentrates 88.99 % of material cost

Nearly nine of every ten pesos of material sit in a single item. That reorders two things in the operation: any real saving is negotiated on the hat body and not across the other twenty items, and one hat body lost in the warehouse costs an order of magnitude more than any other shrinkage.

**Its limit, stated up front:** the costs are `sim_` and preserve the real relative scale, so the finding is about **concentration**, not about amounts. The percentage is defensible; the pesos are not.

### In one paragraph, without the jargon

The tool handed me a perfectly calculated number — "11,053 units of material missing" — that actually meant nothing, because it was adding pieces to rolls to meters to square centimeters. And when I went to build the red-light indicator for which material is short, both obvious ways of calculating it turned out to be useless: one flagged all twenty-one materials, the other flagged none. The alarm wasn't inside the data. I had to define what counts as short myself, and I defined it by what actually stops the line — the hat body, which is the base of the hat and blocks the next three processes. No tool produces that part. I know it because I know the factory floor.

---

## What's in this folder

| File | What it is |
|---|---|
| [`modelo-de-datos.en.md`](modelo-de-datos.en.md) · [`ES`](modelo-de-datos.md) | Model documentation: grain, table dictionary, all 13 DAX measures annotated, design decisions, and what the model **doesn't** answer |
| [`proyecto_a_modelo_de_datos.pbix`](proyecto_a_modelo_de_datos.pbix) | The Power BI file — model, measures, and dashboard, all openable |
| [`capturas/dashboard.png`](capturas/dashboard.png) | The dashboard page |
| [`datos/Pedido_Insumos_ANONIMIZADO.xlsx`](datos/Pedido_Insumos_ANONIMIZADO.xlsx) | The anonymized dataset: three sheets of 17, 21, and 85 rows |
| [`modelo.png`](modelo.png) | Star schema diagram |

**To reproduce:** open the `.pbix`; it loads the `.xlsx` through Power Query and builds the four tables. Control numbers: `f_consumo` at 85 rows, zero nulls in `id_insumo`, `Sombreros Pedidos` = 13,010.

*Versión en español: [README.md](README.md)*
