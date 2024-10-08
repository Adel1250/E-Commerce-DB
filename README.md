# E-Commerce Database Design

## 1. Introduction

This database is designed to manage an e-commerce system, organizing products, categories, customers, orders, and order details. The **`category`** table stores product categories, while the **`product`** table contains detailed information about each product, including price, stock, and its associated category. The **`customer`** table holds customer information, including their names, email addresses, and passwords. The **`orders`** table tracks customer orders with order dates and total amounts, and the **`order_details`** table records specific details about each product in an order, including the quantity and unit price. A foreign key was added in **`order_details`** to directly link each order detail to a customer, streamlining queries related to customer purchase history.

## 2. ERD

![ERD](/diagram/erd.png)

## 3. Sample Queries

### 3.1. Generate a daily report of the total revenue for a specific date

``` sql
select
    order_date,
    sum(total_amount)
from
    orders
where
    order_date = '2024-08-05'
group by
    order_date;
```

### 3.2. Generate a monthly report of the top-selling products in a given month

``` sql
select
    p.product_id,
    p.name,
    sum(od.quantity) as sold_count
from
    product as p
join order_details as od on
    p.product_id = od.product_id
join orders as o on
    od.order_id = o.order_id
where
    extract(month
from
    o.order_date) = 8
    and extract(year
from
    o.order_date) = 2024
group by
    p.product_id
order by
    sold_count desc;
```

### 3.3. Retrieve a list of customers who have placed orders totaling more than $500 in the past month

``` sql
select
    concat(c.first_name,
    ' ',
    c.last_name) as customer_name,
    sum(o.total_amount) as past_month_total_amount
from
    customer as c
join orders as o on
    o.customer_id = c.customer_id
where
    extract(month
from
    o.order_date) = (
    select
        extract(month
    from
        current_date - interval '1 month'))
group by
    customer_name
having
    sum(o.total_amount) > 500::money;
```

### 3.4. Search for all products with the word "camera" in either the product name or description

``` sql
select
    name,
    description,
    price,
    stock_quantity,
    sold_by
from
    product
where
    name like '%camera%'
    or description like '%camera%';
```

### 3.5. Suggest popular products in the same category for the same author, excluding the purchsed product from the recommendations

``` sql
select
    p.product_id,
    p."name",
    p.category_id
from
    product p
where
    p.product_id not in (
    select
        od.product_id
    from
        order_details od
    where
        od.customer_id = 4)
    and p.category_id in (
    select
        p2.category_id
    from
        product p2
    join order_details od2 on
        od2.product_id = p2.product_id
    where
        od2.customer_id = 4)
    and p.sold_by in 
    (
    select
        p2.sold_by
    from
        product p2
    join order_details od2 on
        od2.product_id = p2.product_id
    where
        od2.customer_id = 4)
```

## 4. Sample Trigger Function

The following function is triggered when a new order detail is inserted.

``` sql
create or replace
    function insert_order_history()
returns trigger as $$
begin
    insert
    into
    order_history (
        order_detail_id,
    customer_id,
    order_date,
    order_id,
    product_id,
    quantity,
    unit_price
    )
values (
        NEW.order_detail_id,
        (
select
    o.customer_id
from
    orders o
where
    o.order_id = NEW.order_id),
        (
select
    o.order_date
from
    orders o
where
    o.order_id = NEW.order_id),
        NEW.order_id,
        NEW.product_id,
        NEW.quantity,
        NEW.unit_price
    );

return new;
end;

$$ language plpgsql;

create trigger order_history_trigger
after
insert
    on
    order_details
for each row
execute function insert_order_history();
```

## 5. Applications on Locking

### 5.1. Lock on a column

It is not supported in Postgres.

### 5.2. Lock on a row

``` sql
select
    *
from
    product
where
    product_id = 1
for
update;
```

## 6. Performance Optimization Techniques

### 6.1. Retrieve the total number of products in each category

Before Optimization:

``` sql
select
    c.category_name ,
    count(p.product_id) as total_products
from
    product p
natural join category c
group by
    c.category_name;
```

After Optimization:

``` sql
select
    c.category_name ,
    count(p.product_id) as total_products
from
    product p
natural join category c
group by
    c.category_id;
```

Technique: Grouped by category_id instead of category_name as it is already indexed

### 6.2. Find the top customers by total spending

Query:

```sql
select
    o.customer_id,
    sum(o.total_amount) as customer_total_amount
from
    orders o
group by
    o.customer_id
order by
    customer_total_amount desc;
```

Optimization:

``` sql
drop index idx_customer_id;
```

Technique: Removed index on customer_id for the table orders as the query already scans the whole table, so need for an index as this could be an overhead

### 6.3. Retrieve the most recent orders with customer information with 1000 orders
