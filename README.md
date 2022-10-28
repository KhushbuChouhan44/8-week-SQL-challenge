# 8-week-SQL-challenge
/* --------------------
   Case Study Questions
   --------------------*/

-- 1. What is the total amount each customer spent at the restaurant?
-- 2. What was the first item from the menu purchased by each customer?
-- 3. How many days has each customer visited the restaurant?
-- 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
-- 5. Which item was the most popular for each customer?
-- 6. Which item was purchased first by the customer after they became a member?
-- 7. Which item was purchased just before the customer became a member?
-- 8. What is the total items and amount spent for each member before they became a member?
-- 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
-- 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?


**Schema (PostgreSQL v13)**

    CREATE SCHEMA dannys_diner;
    SET search_path = dannys_diner;
    
    CREATE TABLE sales (
      "customer_id" VARCHAR(1),
      "order_date" DATE,
      "product_id" INTEGER
    );
    
    INSERT INTO sales
      ("customer_id", "order_date", "product_id")
    VALUES
      ('A', '2021-01-01', '1'),
      ('A', '2021-01-01', '2'),
      ('A', '2021-01-07', '2'),
      ('A', '2021-01-10', '3'),
      ('A', '2021-01-11', '3'),
      ('A', '2021-01-11', '3'),
      ('B', '2021-01-01', '2'),
      ('B', '2021-01-02', '2'),
      ('B', '2021-01-04', '1'),
      ('B', '2021-01-11', '1'),
      ('B', '2021-01-16', '3'),
      ('B', '2021-02-01', '3'),
      ('C', '2021-01-01', '3'),
      ('C', '2021-01-01', '3'),
      ('C', '2021-01-07', '3');
     
    
    CREATE TABLE menu (
      "product_id" INTEGER,
      "product_name" VARCHAR(5),
      "price" INTEGER
    );
    
    INSERT INTO menu
      ("product_id", "product_name", "price")
    VALUES
      ('1', 'sushi', '10'),
      ('2', 'curry', '15'),
      ('3', 'ramen', '12');
      
    
    CREATE TABLE members (
      "customer_id" VARCHAR(1),
      "join_date" DATE
    );
    
    INSERT INTO members
      ("customer_id", "join_date")
    VALUES
      ('A', '2021-01-07'),
      ('B', '2021-01-09');

---

**Query #1**

    select s.customer_id, sum(m.price) as Total_amount
    from dannys_diner.sales s inner join dannys_diner.menu m on s.product_id=m.product_id group by s.customer_id order by Total_amount desc, customer_id;

| customer_id | total_amount |
| ----------- | ------------ |
| A           | 76           |
| B           | 74           |
| C           | 36           |

---
**Query #2**

    with cte1 as (
    select *, row_number() over(partition by s.customer_id order by s.order_date) as r 
    from dannys_diner.sales s inner join dannys_diner.menu m on s.product_id=m.product_id
    )
    select customer_id, product_name
    from cte1 where r=1;

| customer_id | product_name |
| ----------- | ------------ |
| A           | curry        |
| B           | curry        |
| C           | ramen        |

---
**Query #3**

    select count(DISTINCT order_date) as Noofdays, customer_id from dannys_diner.sales group by customer_id;

| noofdays | customer_id |
| -------- | ----------- |
| 4        | A           |
| 6        | B           |
| 2        | C           |

---
**Query #4**

    select m.product_name, count(m.product_name) as pcount
    from dannys_diner.sales s inner join dannys_diner.menu m on s.product_id=m.product_id group by m.product_name order by pcount desc limit 1;

| product_name | pcount |
| ------------ | ------ |
| ramen        | 8      |

---
**Query #5**

    with cte1 as
    (
    select customer_id, product_name, count(product_name) as pname
    from dannys_diner.sales s inner join dannys_diner.menu m on s.product_id=m.product_id group by customer_id, product_name
    order by pname desc
    ),
    cte2 as 
    (
      select customer_id, product_name, pname, dense_rank() over(partition by customer_id order by pname desc) as r
    from cte1
    )
    select customer_id, product_name from cte2 where r=1;

| customer_id | product_name |
| ----------- | ------------ |
| A           | ramen        |
| B           | sushi        |
| B           | curry        |
| B           | ramen        |
| C           | ramen        |

---
**Query #6**

    with cte6 as(
    select s.customer_id, m.product_name, b.join_date, s.order_date,
      dense_rank() over(partition by s.customer_id order by s.order_date) as rn
    from dannys_diner.sales s inner join dannys_diner.menu m 
    on s.product_id=m.product_id
    inner join dannys_diner.members b on b.customer_id=s.customer_id
    where s.order_date >= b.join_date)
    select customer_id, product_name from cte6 where rn=1;

| customer_id | product_name |
| ----------- | ------------ |
| A           | curry        |
| B           | sushi        |

---
**Query #7**

    with cte7 as(
    select s.customer_id, m.product_name, b.join_date, s.order_date,
      dense_rank() over(partition by s.customer_id order by s.order_date desc) as rn
    from dannys_diner.sales s inner join dannys_diner.menu m 
    on s.product_id=m.product_id
    inner join dannys_diner.members b on b.customer_id=s.customer_id
    where s.order_date < b.join_date)
    select customer_id, product_name, order_date from cte7 where rn=1;

| customer_id | product_name | order_date               |
| ----------- | ------------ | ------------------------ |
| A           | sushi        | 2021-01-01T00:00:00.000Z |
| A           | curry        | 2021-01-01T00:00:00.000Z |
| B           | sushi        | 2021-01-04T00:00:00.000Z |

---
**Query #8**

    select s.customer_id, count(m.product_id) as totalitems, sum(m.price) as totalspent
    from dannys_diner.sales s inner join dannys_diner.menu m 
    on s.product_id=m.product_id
    inner join dannys_diner.members b on b.customer_id=s.customer_id
    where s.order_date < b.join_date group by s.customer_id order by s.customer_id;

| customer_id | totalitems | totalspent |
| ----------- | ---------- | ---------- |
| A           | 2          | 25         |
| B           | 3          | 40         |

---
**Query #9**

    with cte9 as(
    select *,
    case 
    	when m.product_id = 1 then m.price*20
        else m.price*10
    end as points
    from dannys_diner.sales s inner join dannys_diner.menu m on s.product_id=m.product_id)
    select customer_id, sum(points) as totalpoints from cte9 
    group by customer_id order by 2 desc;

| customer_id | totalpoints |
| ----------- | ----------- |
| B           | 940         |
| A           | 860         |
| C           | 360         |

---
**Query #10**

    select s.customer_id, 
        sum(case when s.order_date between b.join_date and b.join_date+INTERVAL '6 day' then m.price*2*10
        when m.product_id = 1 then m.price*2*10
        else m.price*10 end) as points
        from dannys_diner.sales s inner join dannys_diner.menu m on s.product_id=m.product_id
        inner join dannys_diner.members b on b.customer_id=s.customer_id
        where extract(month from s.order_date) = 01
        group by s.customer_id;

| customer_id | points |
| ----------- | ------ |
| B           | 820    |
| A           | 1370   |

**OR**

---
**Query #11**

    with cte10 as(
    select *, b.join_date+INTERVAL '6 day' as validdate,
    date_trunc('month', b.join_date) + interval '1 month' AS lastdate from dannys_diner.members b)
    select s.customer_id, 
    sum(case when s.order_date between join_date and validdate then m.price*2*10
    when m.product_id = 1 then m.price*2*10
    else m.price*10 end) as points                                 
    from dannys_diner.sales s inner join dannys_diner.menu m on s.product_id=m.product_id
    inner join cte10 c on c.customer_id=s.customer_id
    where s.order_date < lastdate and s.order_date >= c.join_date
    group by s.customer_id;

| customer_id | points |
| ----------- | ------ |
| B           | 320    |
| A           | 1020   |

---
**Query #12**

    select s.customer_id, s.order_date, m.product_name, m.price,
    case when s.order_date >= b.join_date then 'Y'
    else 'N' end as member
    from dannys_diner.sales s left join dannys_diner.menu m 
    on s.product_id=m.product_id
    left join dannys_diner.members b on b.customer_id=s.customer_id
    order by 1, s.order_date;

| customer_id | order_date               | product_name | price | member |
| ----------- | ------------------------ | ------------ | ----- | ------ |
| A           | 2021-01-01T00:00:00.000Z | sushi        | 10    | N      |
| A           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |
| A           | 2021-01-07T00:00:00.000Z | curry        | 15    | Y      |
| A           | 2021-01-10T00:00:00.000Z | ramen        | 12    | Y      |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      |
| B           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |
| B           | 2021-01-02T00:00:00.000Z | curry        | 15    | N      |
| B           | 2021-01-04T00:00:00.000Z | sushi        | 10    | N      |
| B           | 2021-01-11T00:00:00.000Z | sushi        | 10    | Y      |
| B           | 2021-01-16T00:00:00.000Z | ramen        | 12    | Y      |
| B           | 2021-02-01T00:00:00.000Z | ramen        | 12    | Y      |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |
| C           | 2021-01-07T00:00:00.000Z | ramen        | 12    | N      |

---
**Query #13**

    with cteb as(
    select s.customer_id, s.order_date, m.product_name, m.price,
    case when s.order_date >= b.join_date then 'Y'
    else 'N' end as member
    from dannys_diner.sales s left join dannys_diner.menu m 
    on s.product_id=m.product_id
    left join dannys_diner.members b on b.customer_id=s.customer_id
    order by 1, s.order_date)
    select *, 
    case when member = 'N' then NULL
    else dense_rank() over (partition by customer_id, member order by order_date) end as ranking from cteb;

| customer_id | order_date               | product_name | price | member | ranking |
| ----------- | ------------------------ | ------------ | ----- | ------ | ------- |
| A           | 2021-01-01T00:00:00.000Z | sushi        | 10    | N      |         |
| A           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |         |
| A           | 2021-01-07T00:00:00.000Z | curry        | 15    | Y      | 1       |
| A           | 2021-01-10T00:00:00.000Z | ramen        | 12    | Y      | 2       |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      | 3       |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      | 3       |
| B           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |         |
| B           | 2021-01-02T00:00:00.000Z | curry        | 15    | N      |         |
| B           | 2021-01-04T00:00:00.000Z | sushi        | 10    | N      |         |
| B           | 2021-01-11T00:00:00.000Z | sushi        | 10    | Y      | 1       |
| B           | 2021-01-16T00:00:00.000Z | ramen        | 12    | Y      | 2       |
| B           | 2021-02-01T00:00:00.000Z | ramen        | 12    | Y      | 3       |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |         |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |         |
| C           | 2021-01-07T00:00:00.000Z | ramen        | 12    | N      |         |

---

[View on DB Fiddle](https://www.db-fiddle.com/f/cd2C4AcAhyWg7e1hRP6GpZ/1)
