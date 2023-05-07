---
layout: post
title: MySQL Adjacency List pattern
date: 2023-03-10 22:51 +0100
---

Adjacency List pattern is a way of representing hierachical data
in a relational database. Each row (apart from the root) has a reference
that points to it's parent in the hierachy.

Let's take a look at an example.
We would like to store the following book genres tree in the database:

![image](/assets/2023-03-10-adjacency-list-pattern/adjacency_list.png)

To model this tree we define the following schema:

```sql
create table book_genres(
  id serial primary key,
  genre varchar(100),
  parent_id bigint unsigned,
  foreign key (parent_id) references book_genres(id)
);
```
When inserting data we provide the id of the parent node in the `parent_id` column:
```sql
insert into book_genres(genre, id, parent_id) values
('science fiction', 1, null),
('utopian & dystopian', 2, 1),
('dystopian', 3, 2),
('cyberpunk', 4, 3),
('biopunk', 5, 4),
('nanopunk', 6, 4),
('steampunk', 7, 4),
('utopian', 8, 2),
('comedy', 9, 1),
('hard', 10, 1),
('parallel world', 11, 10),
('climate fiction', 12, 10);
```

## Querying data
### Select root node
To retrieve the root node we look for rows which have the parent id set to null:
```sql
select * from book_genres where parent_id is null;
```
```
+----+-----------------+-----------+
| id | genre           | parent_id |
+----+-----------------+-----------+
|  1 | science fiction |      NULL |
+----+-----------------+-----------+
```

### Select direct children of a node
To find the direct children of a given node we search for all rows which have specific parent id:
```sql
select * from book_genres where parent_id = '10';
```
```
+----+-----------------+-----------+
| id | genre           | parent_id |
+----+-----------------+-----------+
| 11 | parallel world  |        10 |
| 12 | climate fiction |        10 |
+----+-----------------+-----------+
```
### Select all leaf nodes in a subtree
To find leaf nodes of a subtree we need to find nodes
that have no descendants.
```sql
select bg_1.id, bg_1.genre, bg_1.parent_id from book_genres as bg_1
left join book_genres as bg_2 on bg_1.id = bg_2.parent_id
where bg_2.id is null;
```
```
+----+-----------------+-----------+
| id | genre           | parent_id |
+----+-----------------+-----------+
|  5 | biopunk         |         4 |
|  6 | nanopunk        |         4 |
|  7 | steampunk       |         4 |
|  8 | utopian         |         2 |
|  9 | comedy          |         1 |
| 11 | parallel world  |        10 |
| 12 | climate fiction |        10 |
+----+-----------------+-----------+
```

### Select all descendants of a given node
To find all descendants of a given node we have to do a recursive query:
```sql
with recursive child_nodes as (
  select id, parent_id, genre from book_genres
  where id = '2'
  union all
  select bg.id, bg.parent_id, bg.genre from child_nodes as cn
  join book_genres as bg on cn.id = bg.parent_id
)
select * from child_nodes;
```
```
+------+-----------+---------------------+
| id   | parent_id | genre               |
+------+-----------+---------------------+
|    2 |         1 | utopian & dystopian |
|    3 |         2 | dystopian           |
|    8 |         2 | utopian             |
|    4 |         3 | cyberpunk           |
|    5 |         4 | biopunk             |
|    6 |         4 | nanopunk            |
|    7 |         4 | steampunk           |
+------+-----------+---------------------+
```

If we want to add a path to each row to show all it's ancestors
we can use the `concat` function to combine the genres:

```sql
with recursive child_nodes as (
  select id, parent_id, concat('> ', genre) as path
  from book_genres
  where id = '2'
  union all
  select bg.id, bg.parent_id, concat(cn.path, ' > ', bg.genre) as path from child_nodes as cn
  join book_genres as bg on cn.id = bg.parent_id
)
select * from child_nodes;
```
```
+------+-----------+-----------------------------------------------------------+
| id   | parent_id | path                                                      |
+------+-----------+-----------------------------------------------------------+
|    2 |         1 | > utopian & dystopian                                     |
|    3 |         2 | > utopian & dystopian > dystopian                         |
|    8 |         2 | > utopian & dystopian > utopian                           |
|    4 |         3 | > utopian & dystopian > dystopian > cyberpunk             |
|    5 |         4 | > utopian & dystopian > dystopian > cyberpunk > biopunk   |
|    6 |         4 | > utopian & dystopian > dystopian > cyberpunk > nanopunk  |
|    7 |         4 | > utopian & dystopian > dystopian > cyberpunk > steampunk |
+------+-----------+-----------------------------------------------------------+
```

### Select a path from root to child
```sql
with recursive node_path as (
  select id, genre, parent_id from book_genres
  where id = '7'
  union all
  select bg.id, bg.genre, bg.parent_id from node_path as np
  join book_genres as bg on np.parent_id = bg.id
)
select * from node_path;
```
```
+------+---------------------+-----------+
| id   | genre               | parent_id |
+------+---------------------+-----------+
|    7 | steampunk           |         4 |
|    4 | cyberpunk           |         3 |
|    3 | dystopian           |         2 |
|    2 | utopian & dystopian |         1 |
|    1 | science fiction     |      NULL |
+------+---------------------+-----------+
```

### Calculate each nodes level
```sql
with recursive nodes_level as (
  select id, genre, parent_id, 0 as level from book_genres
  where parent_id is null
  union all
  select bg.id, bg.genre, bg.parent_id, nl.level + 1 as level from nodes_level as nl
  join book_genres as bg on nl.id = bg.parent_id
)
select * from nodes_level
order by level asc;
```
```
+------+---------------------+-----------+-------+
| id   | genre               | parent_id | level |
+------+---------------------+-----------+-------+
|    1 | science fiction     |      NULL |     0 |
|    2 | utopian & dystopian |         1 |     1 |
|    9 | comedy              |         1 |     1 |
|   10 | hard                |         1 |     1 |
|    3 | dystopian           |         2 |     2 |
|    8 | utopian             |         2 |     2 |
|   11 | parallel world      |        10 |     2 |
|   12 | climate fiction     |        10 |     2 |
|    4 | cyberpunk           |         3 |     3 |
|    5 | biopunk             |         4 |     4 |
|    6 | nanopunk            |         4 |     4 |
|    7 | steampunk           |         4 |     4 |
+------+---------------------+-----------+-------+
```
### Count number of children for each node
To count the number of direct subgenres for each node
we join each node with it's child nodes and then count
the number of rows.
```sql
select bg1.id, max(bg1.genre) as genre, count(bg2.parent_id) as subgenres
from book_genres as bg1
left join book_genres as bg2 on bg1.id = bg2.parent_id
group by bg1.id;
```
```
+----+---------------------+-----------+
| id | genre               | subgenres |
+----+---------------------+-----------+
|  1 | science fiction     |         3 |
|  2 | utopian & dystopian |         2 |
|  3 | dystopian           |         1 |
|  4 | cyberpunk           |         3 |
|  5 | biopunk             |         0 |
|  6 | nanopunk            |         0 |
|  7 | steampunk           |         0 |
|  8 | utopian             |         0 |
|  9 | comedy              |         0 |
| 10 | hard                |         2 |
| 11 | parallel world      |         0 |
| 12 | climate fiction     |         0 |
+----+---------------------+-----------+
```


### Bibliography
- [https://www.mysqltutorial.org/mysql-adjacency-list-tree](https://www.mysqltutorial.org/mysql-adjacency-list-tree)
- [https://en.wikipedia.org/wiki/List_of_writing_genres](https://en.wikipedia.org/wiki/List_of_writing_genres)
- [https://www.percona.com/blog/introduction-to-mysql-8-0-recursive-common-table-expression-part-2](https://www.percona.com/blog/introduction-to-mysql-8-0-recursive-common-table-expression-part-2)
