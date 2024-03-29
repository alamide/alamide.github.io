---
layout: post
title:  "LeetCode1795 SQL行列互转"
date:   2023-02-16
categories: LeetCode SQL
tags: LeetCode SQL 
sql_table: sql-leetcode-1795.html
---

重构 t_products 表，查询每个产品在不同商店的价格，使得输出的格式变为(product_id, store, price) 即t_trans_products。如果这一产品在商店里没有出售，则不输出这一行。
<!--more-->

```sql
select product_id, 'store1' as store, store1 as price from t_products where store1 is not null
union all
select product_id, 'store2' as store, store2 as price from t_products where store2 is not null
union all
select product_id, 'store3' as store, store3 as price from t_products where store3 is not null;
```
<table>
  <tr><th>product_id</th><th>store</th><th>price</th></tr>
  <tr><td>0</td><td>store1</td><td>95</td></tr>
  <tr><td>0</td><td>store2</td><td>100</td></tr>
  <tr><td>0</td><td>store3</td><td>105</td></tr>
  <tr><td>1</td><td>store1</td><td>70</td></tr>
  <tr><td>1</td><td>store3</td><td>80</td></tr>
</table>

将输出的结果插入 t_trans_products
```sql
insert into t_trans_products(product_id, store, price)
    select * from (
        select product_id, 'store1' as store, store1 as price from t_products where store1 is not null
        union all
        select product_id, 'store2' as store, store2 as price from t_products where store2 is not null
        union all
        select product_id, 'store3' as store, store3 as price from t_products where store3 is not null
        order by product_id, store
    ) as t_temp;
```
### 列转行
* 重构 t_trans_products 表，输出的格式变为(product_id, store1, store2, store3) 即t_products。如果这一产品在商店里没有出售，则该列为NULL。
  
```sql
select
    product_id,
    sum(if(store='store1', price, null)) store1,
    sum(if(store='store2', price, null)) store2,
    sum(if(store='store3', price, null)) store3
from t_trans_products
group by product_id;
```
<table>
  <tr><th>product_id</th><th>store1</th><th>store2</th><th>store3</th></tr>
  <tr><td>0</td><td>95</td><td>100</td><td>105</td></tr>
  <tr><td>1</td><td>70</td><td>NULL</td><td>80</td></tr>
</table>

sum 是聚合函数，将分组中的对应列数值相加。`可以理解为遍历结果集，将对应列相加`。

其它的聚合函数也可以如 `max(if(store='store1', price, null))`

对于分组 product_id=0 `sum(if(store='store1', price, null))` 即表示 price + null + null。price 为 95

对于分组 product_id=1 `sum(if(store='store2', price, null))` 即表示 null + null + null。


模拟的Sum过程
```java
public class GroupedSum {

    static class TTransProduct {
        public Integer productId;
        public String store;
        public Integer price;

        public TTransProduct(Integer productId, String store, Integer price) {
            this.productId = productId;
            this.store = store;
            this.price = price;
        }
    }

    static class Product {
        public Integer productId;
        public Integer store1;
        public Integer store2;
        public Integer store3;

        public Product() {
        }

        public Product(Integer productId, Integer store1, Integer store2, Integer store3) {
            this.productId = productId;
            this.store1 = store1;
            this.store2 = store2;
            this.store3 = store3;
        }
    }

    public static void main(String[] args) {
        Map<Integer, List<TTransProduct>> groupedResultSet = new HashMap<>();
        List<TTransProduct> id0 = new ArrayList<>();
        id0.add(new TTransProduct(0, "store1", 95));
        id0.add(new TTransProduct(0, "store2", 100));
        id0.add(new TTransProduct(0, "store3", 105));
        List<TTransProduct> id1 = new ArrayList<>();
        id1.add(new TTransProduct(1, "store1", 70));
        id1.add(new TTransProduct(1, "store3", 80));

        groupedResultSet.put(0, id0);
        groupedResultSet.put(1, id1);

        List<Product> resultSet = new ArrayList<>();
        for (Map.Entry<Integer, List<TTransProduct>> entry : groupedResultSet.entrySet()) {
            Integer productId = entry.getKey();
            Product product = new Product();
            product.productId = productId;

            //sum(if(store='store1', price, NULL)) as store1
            Integer store1Price = null;
            for (TTransProduct item : entry.getValue()) {
                //if(store='store1', price, NULL)
                if (Objects.equals("store1", item.store)) {
                    store1Price += item.price;
                }//else{
                    //sum + NULL
                //}
            }
            product.store1 = store1Price;
            //sum(if(store='store2', price, NULL)) as store2
            //sum(if(store='store3', price, NULL)) as store3

            resultSet.add(product);
        }

    }
}
```






