---
layout: post
title: "SQL part II - a little bit too late..."
date: 2019-07-22T18:01:27+02:00
keywords: "job, interview, ruby, sql"
---
Some time ago I had an interview with really cool company. I totally messed it up, despite the fact that... questions were really basic.  
There were two tasks: one in Ruby and one with SQL. Ruby took me less than 10 minutes, but SQL... It's embarrassing to talk about it.  


The question was about joining three tables in one query... With ActiveRecord I did it on the fly, but then I realized that I'm Rails developer without SQL skills. Shame. So let's start with simple joins.

## JOINS
![img](https://i.imgur.com/Hs9auHg.jpg)  

Let's say, we have 2 tables:  
- Books,  
- Authors,  

with following structure:  
```
books |   
------|   
id    |   
title |
author_id|

authors|
-------|
id     |
name   |
```

And following data:  
```
test=# select * from books;
 id |       title       | author_id 
----+-------------------+-----------
  1 | Sapiens           |         3
  2 | Homo Deus         |         3
  3 | Harry Potter      |         2
(3 rows)

test=# SELECT * FROM authors;
 id |     name     
----+--------------
  1 | Mr. Hopkins
  2 | J.K. Rowling
  3 | Yuval Noah Harari
(3 rows)
```  

You see that associations between them, right? Author has many books, book belongs to author. 
### INNER JOIN
So now, what we want is to select only authors who have any books assigned to them.
```
SELECT authors.* FROM authors
  INNER JOIN books ON books.author_id = authors.id;
```  
What we did here is `INNER JOIN` - we joined two tables and printed out records, which have matching values (`authors.id` equals `books.author_id`).
If we'd like not to repeat authors printed out, we should append `DISTINCT` keyword after `SELECT`.

### FULL OUTER JOIN / FULL JOIN

To select all data from both tables we should use `FULL OUTER JOIN`:
```
SELECT * FROM authors
  FULL OUTER JOIN books ON books.author_id = authors.id;
```
It prints out all data - all records from books table and all records from authors table. It is the same as `FULL JOIN`

### LEFT JOIN / LEFT OUTER JOIN
Left join returns all records from `authors` (the table on the left) - if we start with `books`, it would be `books` of course - and matched records from joined table - `books`.
So, let's print all authors, and their book's count!
```
SELECT authors.name, COUNT(authors.name) AS books_count FROM authors
  LEFT OUTER JOIN books
  ON authors.id = books.author_id
  GROUP BY authors.name;
```
It prints out nice, grouped data:
```
     name          | books_count 
-------------------+-------------
 Yuval Noah Harari |     2
 J.K. Rowling      |     1
 Hopkins           |     1
(3 rows)
```

### What about more than two tables?
At some point (at recruitment interview for example) you will need to join more than two tables and fetch data from them.  
Let's add another table to the previous ones.
```
authors_books
-------------
author_id
book_id
```
And remove column `books.author_id`. It gives us many to many association.  
Now, records in this table should look like this:
```
test=# SELECT * FROM authors_books;
 id | author_id | book_id 
----+-----------+---------
  1 |         3 |       1
  2 |         3 |       2
  3 |         2 |       3
(3 rows)
```

To print authors with corresponding books, we need to join `authors` with `authors_books` and then `authors_books` with `books` to match given ids.
Let's try with this SQL query:
```
SELECT authors.name, books.title FROM authors 
  JOIN authors_books ON authors.id = authors_books.author_id
    JOIN books ON authors_books.book_id = books.id;
```
```
        name        |       title       
--------------------+-------------------
 Yuval Noah Harari  | Sapiens
 Yuval Noah Harari  | Homo Deus
 J.K. Rowling       | Harry Potter
(3 rows)
```
Now, we can count their books as well:
```
test=# SELECT authors.name, COUNT(authors.name) FROM authors
         JOIN authors_books ON authors.id = authors_books.author_id
           JOIN books ON authors_books.book_id = books.id GROUP BY authors.name;

        name        | count 
--------------------+-------
 Yuval Noah Harari  |     2
 J.K. Rowling       |     1
(2 rows)
```

### Enough joins for now
As you can see, SQL queries aren't that hard and scary as they seem to be. ActiveRecord made Rails developers lazy and sql-disabled (as I am), but sometimes it is worth-knowing how to write some complex joins in raw SQL.