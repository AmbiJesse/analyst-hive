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
