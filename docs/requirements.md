# Requirements: PCBA Production Planner

## 1. Purpose

A simplified production planning model for a fictional PCB assembly company. It reproduces, on a small SQLite database, what an ERP does in the planning area: master data, bill of materials (BOM) explosion, net requirements (MRP), capacity load by work centre, and planning KPIs.

The project is a portfolio piece. It is a model, not an ERP replacement, and all data is synthetic.

## 2. Scenario

- A contract manufacturer of electronic boards.
- 6 to 8 board families, each sold in its finished, tested form.
- A pool of about 60 to 80 purchased components, partly shared between families (shared components are what make requirement aggregation meaningful).
- Three work centres: pick and place, conformal coating, test.
- Planning horizon: 26 weeks, weekly time buckets.

## 3. Item structure

Each processing step produces its own item code, so stock can be tracked at every stage.

| Item | Meaning | Made from | Work centre |
|---|---|---|---|
| `X` | Assembled board | Bare PCB + components | Pick and place |
| `X.1` | Coated board | `X` (1) + coating material | Conformal coating |
| `X.2` | Tested board (sold item) | `X.1` (1) | Test |

Notes:
- Customer orders are placed on `X.2`; requirements cascade down to purchased items.
- The test step consumes no material: it appears in the routing but the BOM of `X.2` contains only `X.1` with quantity 1.
- Coating material is a small per-piece consumption in the BOM of `X.1`.

BOM and routing are two separate concepts: the BOM says what an item consumes, the routing says which resource it occupies and for how long.

## 4. Planning logic (defaults)

- **Time buckets:** weekly. No setup or changeover times in the base model.
- **Lot sizing:**
  - `X`, `X.1`, `X.2`: lot for lot (produce what is required).
  - Purchased items (PCB, components, coating material): minimum order quantity (`min_order_qty`), rounded up to the lot.
- **Lead time:** per item, in weeks. Purchased items use supplier lead time; produced items use a standard production lead time.
- **Safety stock:** per item, subtracted from available stock in the netting.
- **Net requirement:** gross requirement minus on-hand stock minus open orders (scheduled receipts), plus safety stock.
- **Order proposals:** net requirement rounded to the lot rule, released earlier than the need date by the item lead time.
- **BOM explosion:** level by level, from finished items down to purchased items.

## 5. Modules

1. **Data generator.** Creates the SQLite database from a fixed seed (reproducible).
2. **MRP engine.** BOM explosion, netting, lot sizing, order proposals.
3. **Capacity load.** Required hours per work centre per week against available capacity; flags overloaded weeks.
4. **KPIs.** Computed from the database (see section 8).
5. **Dashboard.** Streamlit, with filters by product family and period.

Calculation split: recursive BOM explosion and aggregations in SQL (recursive CTEs); time-phased netting and lot logic in Python.

## 6. Database schema (starting point)

```sql
items(code PK, description,
      item_type CHECK IN ('purchased','assembled','coated','tested'),
      lead_time_weeks INT, min_order_qty INT, safety_stock INT)
bom(parent_item FK, component_item FK, quantity_per REAL CHECK > 0,
    PK(parent_item, component_item))
work_centers(code PK, description, weekly_capacity_hours REAL)
routings(item PK/FK, work_center FK, seconds_per_piece REAL)
stock(item PK/FK, quantity REAL)
customer_orders(id PK, item FK, quantity, due_week INT)
open_orders(id PK, item FK, quantity, receipt_week INT)
order_proposals(item FK, quantity, release_week, due_week)
```

Naming convention: English, lowercase, snake_case; plural table names, singular column names.

Constraints to enforce in the schema:
- foreign keys enabled (`PRAGMA foreign_keys = ON`);
- positive quantities;
- a parent item cannot be its own component;
- no cyclic BOMs (checked by the generator and by a test).

Routing has one row per produced item; purchased items have no routing.

## 7. Synthetic data generator (indicative parameters)

All values are starting points to be calibrated with production planning experience.

- Fixed random seed.
- 6 to 8 families, each with items `X`, `X.1`, `X.2`.
- Each board uses 20 to 60 distinct components drawn from the shared pool; one bare PCB per family; one coating material shared by all.
- Purchase lead times: 1 to 8 weeks. Minimum order quantity varies by component type (for example reels of 1,000 to 5,000 pieces for passives, small lots for connectors).
- Customer orders: 100 to 2,000 pieces per family, irregular dates over 26 weeks.
- Process times: pick and place proportional to component count; coating and test close to fixed time per piece.
- Capacity per work centre calibrated so that demand nearly saturates it in some weeks, to produce bottlenecks worth analysing.
- Initial stock and some open purchase orders, so that netting is not trivial.

## 8. KPIs

- On-time delivery (customer orders covered by the requested week).
- Stockouts (weeks and items with negative projected availability).
- Work centre utilisation by week.
- Inventory value (using a unit cost per item).
- Optional: number of order proposals released late (needed before today).

## 9. Out of scope (possible extensions)

- Finite scheduling (for example with OR-Tools).
- Demand forecasting.
- Setup and changeover times.
- Test yield, scrap and rework.
- Data entry interface.

## 10. Validation

- A minimal case (one board, two components) calculated by hand; the code must give the same result. This becomes the base of the automated tests.
- Consistency checks on the database: no orphan references, no cyclic BOMs, no negative quantities.
- Comparison with a naive baseline where useful.
- A log of AI-assisted work (what was requested, what was wrong, how it was verified) kept in `docs/ai-usage.md`.

## 11. Definition of done

- The database is created from scratch by one script.
- MRP results match the hand-calculated case.
- At least four KPIs and the capacity load are shown in the dashboard.
- README published in English, with a section mapping model concepts to ERP concepts.
- Repository made public.

Anything beyond this list is an extension and is not required to call the project finished.
