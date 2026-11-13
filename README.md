# Analyst Hive: Analyst Week (5 SQL tickets)
Start Date: Monday, November 9, 2026

### Day 0: Onboarding

```
GoldComb Goods database includes customers, products, orders, order_items, refunds, and metrics_glossary tables.
```
### Day 1: First ticket
- **How many customers do we have?** - Followed up with clarification on whether this is everyone who's signed up or active customers. It's a part of the simulation to only show everyone in the customers table.

```SQL
select count(*) as customers
from customers;
```
Result: total "lifetime" customers is `49,478` (*not omitting test accounts circa 200-300)

### Day 2: Active Customers clarification
- The simulation prompted it meant active customers. Active customer is defined as: (1) customers in customers table that are not test accounts, (2) are between a certain timeframe, and (3) have at least one order that's not cancelled.

```SQL
select count(distinct c.customer_id) as active_members

from customers as c
join orders as o on c.customer_id = o.customer_id

where c.is_test_account = false
  and order_ts >= '2025-10-01' and order_ts < '2026-10-01'
  and o.status in ('completed', 'refunded');
```
Result: total active customers is `20,100`
- The query removes test accounts located in customers table, is customers between Oct. 1 2025 and Sept. 30, 2026, and are customers with orders that are at least completed or refunded.

### Day 3: Refunds Inquiry by Finance
- **Refunds spiked** - it was flagged that refunds spiked in September 2026 and they want to know which product is behind it before the finance review. Before blaming a product, check the data.

**Step 1: Count the refunds**
```SQL
select count(*) as refunds from refunds
where refund_ts >= '2026-09-01 00:00:00' and refund_ts < '2026-10-01 00:00:00';
```
Result: 221 refunds (while August had 89)

**Step 2: Look for duplicates**
- An order is normally refunded once, at most. **How many orders have more than one refund logged in September 2026?**
- Pre-queries show source_system has 2765 in legacy and 186 in new.

```SQL
select count(*) as refunded_twice
from (
  
  select order_id from refunds
  where refund_ts >= '2026-09-01 00:00:00' and refund_ts < '2026-10-01 00:00:00'
  group by order_id
  having count(*) > 1
  
  );
```
Result: refunded_twice is 55.

**Step 3: Find the real culprit**
- 55 orders logged twice. It's now disclosed refunds moved to new system in September and for two weeks the system logged both.
- Now the real question. With each refund counted once, which product is costing us the most in refund dollars in September? Dollars, not the number of refunds. Just send the product.

```SQL
with sept_refunds as (
  select distinct order_id, amount from refunds
  where refund_ts >= '2026-09-01 00:00:00' and refund_ts < '2026-10-01 00:00:00'
)

select p.product_name, sum(r.amount) as refund_dollars
  
from sept_refunds as r
join order_items as oi on oi.order_id = r.order_id
join products as p on p.product_id = oi.product_id
group by p.product_name
order by refund_dollars DESC
limit 1;
```
Result: the product 'Drip-Free Glass Honey Dispenser is the product costing the most in refund dollars with $3,196.00 refunded.
Consensus: EDA during part two already identified that the legacy system may be the reason for the duplicate refunds logged.

### Day 4: Sales and Finance can't agree on a number
For September 2026, Marcus says revenue was about $299.5k. Priya says about $258.7k. Both swear their number is right, and the leadership meeting is Monday.
- Responded with '**How exactly does each one of you calculate revenue?**' Both defined in the metrics_glossary: Gross Sales is Marcus's, and Net Revenue is Priya's.
- One thing that trips people up: 'order_ts' is stored in UTC but Sales reports in US Eastern Time, which is UTC-4 in September.

| metric_name | definition |
| ----------- | ----------- |
| Gross Sales | Sum of quantity * unit_price from order_items for every order placed in the period, any status. Periods are cut in US Eastern time (UTC-4 in September; order_ts is stored in UTC). Used for sales targets and commission.	 |
| Net Revenue | Sum of quantity * unit_price from order_items for orders placed in the period that were not cancelled, minus refunds issued in the period (each refund counted once). Periods are cut in UTC. This is the number reported to the board.	 |

**Step 1: Marcus's Number**
- Rebuild Gross Sales for Sept 2026 his team's way: every order placed, any status, with the month cut in Eastern time.
```SQL
select sum(quantity * unit_price) as gross_sales

from order_items as oi
join orders as o on o.order_id = oi.order_id

where o.order_ts >= '2026-09-01 04:00:00' and o.order_ts < '2026-10-01 04:00:00';
```
Result: gross_sales is 299482.00 or $299.5k

**Step 2: Priya's Sales**
- Rebuild Priya's side, starting with sales: Sept 2026 sales, leaving out cancelled orders, with the month cut in UTC (no time shift). Don't subtract refunds yet.
- At this step, simply: change back to normal UTC months, omit cancelled orders, postpone refunds.
```SQL
select sum(quantity * unit_price) as net_revenue
  
from order_items as oi
join orders as o on o.order_id = oi.order_id
  
where o.order_ts >= '2026-09-01 00:00:00' and o.order_ts < '2026-10-01 00:00:00'
and o.status in ('completed', 'refunded')
```
Result: incomplete query puts revenue at 266789.00 or $266.8k.

**Step 3: Subtract the refunds**
- Last step: subtract September's refunds, each counted once (remember Wednesday). That should land on Priya's number.
```SQL
with sales as (
  select sum(quantity * unit_price) as sales
  from orders as o
  join order_items as oi on o.order_id = oi.order_id

  where o.status <> 'cancelled'
  and o.order_ts >= '2026-09-01 00:00:00' and o.order_ts < '2026-10-01 00:00:00'
),

refunded as (
  select sum(amount) as refunds
  from (
  select distinct order_id, amount
  from refunds
  where refund_ts >= '2026-09-01 00:00:00' and refund_ts < '2026-10-01 00:00:00'
  )
)

select sales - refunds as net_revenue
from sales, refunded;
```
Result: net_revenue is 258721.00 or $258.7k

`Rebuilt both numbers from the raw data. Marcus ($299,482) and Priya ($258,721) are both right. They're $40,761 apart because they measure different things. `

### Day 5: The Leadership Summary
