# E-Commerce Database Design

## 1. Introduction

This database schema is designed for an e-commerce application, where it organizes data related to products, categories, customers, orders, and order details. It leverages PostgreSQL with the "uuid-ossp" extension to generate UUIDs for unique identifiers in the `customer`, `orders`, and `order_details` tables. The schema includes a `category` table to categorize products, a `product` table containing details about each product, a `customer` table for customer information, an `orders` table for managing customer orders, and an `order_details` table that records specific products within each order. The schema enforces data integrity through foreign key relationships and various constraints, ensuring that data such as stock quantities and email formats are valid.

## 2. ERD

![ERD](/diagram/e-commerce-erd.png)

## 3. Schema DDL

``` sql
create extension if not exists "uuid-ossp";

create table category
(
    category_id serial primary key,
    category_name varchar(50)
);

create table product
(
    product_id serial primary key,
    category_id serial not null,
    name varchar(100) not null,
    description varchar(200),
    price money not null,
    stock_quantity integer not null check (stock_quantity >= 0),
    foreign key (category_id) references category(category_id)
);

create table customer
(
    customer_id uuid default uuid_generate_v4() primary key,
    first_name varchar(50) not null,
    last_name varchar(50) not null,
    email varchar(255) not null check (email ~* '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'),
    password text not null
);

create table orders
(
    order_id uuid default uuid_generate_v4() primary key,
    customer_id uuid not null,
    order_date date not null,
    total_amount money not null,
    foreign key (customer_id) references customer(customer_id)
);

create table order_details
(
    order_detail_id uuid default uuid_generate_v4() primary key,
    order_id uuid not null,
    product_id serial not null,
    quantity integer check (quantity > 0),
    unit_price money,
    foreign key (order_id) references orders (order_id),
    foreign key (product_id) references product (product_id)
);
```

## 4. Sample Queries

### 4.1. Generate a daily report of the total revenue for a specific date

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

### 4.2. Generate a monthly report of the top-selling products in a given month

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

### 4.3. Retrieve a list of customers who have placed orders totaling more than $500 in the past month

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

### 4.4. Search for all products with the word "camera" in either the product name or description

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

### 4.5. Suggest popular products in the same category for the same author, excluding the purchsed product from the recommendations

``` sql
select
    p.name,
    p.description,
    p.price,
    p.stock_quantity,
    p.sold_by,
    sum(od.quantity) as purchase_count
from
    product p
join order_details od on
    p.product_id = od.product_id
where
    p.category_id = 1
    and p.sold_by = 'Kodak'
    and p.product_id not in (
    select
        product_id
    from
        order_details
    where
        product_id = 6)
group by
    p.product_id
order by
    purchase_count desc
```
