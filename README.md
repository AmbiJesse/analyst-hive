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

### Day 3: 
