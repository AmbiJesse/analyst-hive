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
from customers
```
Result: total "lifetime" customers is `49,478`

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

### Day 4



