---
layout: post
title: Modeling comments in MySQL
date: 2023-05-07 18:00 +0100
---
Let’s say we want to model a simplified version of youtube comments.\
Specifically, we want to:
- add comments
- reply to comments
- like/dislike comments and replies
- sort comments by the number of likes and/or creation date
- load more comments as the user scrolls down (infinite scroll)

To support these requirements we will store both comments and replies in the same table.\
We will use the adjacency list pattern to link replies to their comment using the `parent_id` field.

Lets have a look at the comments table schema:
```sql
create table comments (
  id serial primary key,
  parent_id bigint unsigned default null,
  video_id bigint unsigned,
  content varchar(10000) not null,
  likes int unsigned default 0,
  created_at timestamp not null default current_timestamp,
  author_name varchar(100) not null,
  foreign key (parent_id) references comments(id),
  foreign key (video_id) references videos(id),
  index (parent_id)
);
```
Replies will have `parent_id` set to the id of the comment. Comments will have the `parent_id` set to `null`. `author_name` refers to the user who created the comment. Usually, we would store this information in a separate table and link it with a foreign key but in this example, for simplicity we store it directly in the comments table. `video_id` refers to the video the comment is posted on.

#### Inserting comments
```sql
insert into comments(video_id, content, author_name)
values (1, 'This is cool!', 'John');
```
We don't pass parent id as this is a top-level comment.
#### Inserting replies
```sql
insert into comments(video_id, content, author_id, parent_id)
values (1, 'It is cool indeed', 'Anne', 1);
```
For reply, we pass the id of the comment as the parent id.
#### Liking a comment
```sql
update comments
set likes = likes + 1
where id = '1';
```

Before moving forward let's insert some data to be able to see the result of the queries.

```sql
insert into comments(id, parent_id, video_id, content, author_name, likes, created_at)
values
(1,  null, 1, 'comment 1', 'user 1',   3, '2023-03-01 13:30:00'),
(2,  1,    1, 'reply 1.1', 'user 1.1', 1, '2023-03-01 13:30:10'),
(3,  1,    1, 'reply 1.2', 'user 1.2', 2, '2023-03-01 13:30:20'),
(4,  1,    1, 'reply 1.3', 'user 1.3', 3, '2023-03-01 13:30:30'),
(5,  1,    1, 'reply 1.4', 'user 1.4', 1, '2023-03-01 13:30:40'),
(6,  null, 1, 'comment 2', 'user 2',   1, '2023-03-01 13:31:00'),
(7,  6,    1, 'reply 2.1', 'user 2.1', 1, '2023-03-01 14:30:10'),
(8,  6,    1, 'reply 2.2', 'user 2.2', 2, '2023-03-01 14:30:20'),
(9,  6,    1, 'reply 2.3', 'user 2.3', 3, '2023-03-01 14:30:30'),
(10, 6,    1, 'reply 2.4', 'user 2.4', 1, '2023-03-01 14:30:40'),
(11, null, 1, 'comment 3', 'user 3',   1, '2023-03-01 13:32:00'),
(12, null, 2, 'comment 4', 'user 4',   4, '2023-03-01 14:01:00'),
(13, null, 2, 'comment 5', 'user 5',   2, '2023-03-01 15:23:00'),
(14, null, 2, 'comment 6', 'user 6',   1, '2023-03-01 16:38:00');
```
#### Sorting comments
There are two ways in which we allow users to sort comments.\
First is "Top comments" which sorts comments by the number of likes in descending order.
If comments have the same number of likes we sort by date in descending order.
We may at first write this query in the following manner:
```sql
select author_name, content, likes, created_at from comments
where video_id = '1' and parent_id is null
order by likes desc, created_at desc;
```
```
+-------------+-----------+-------+---------------------+
| author_name | content   | likes | created_at          |
+-------------+-----------+-------+---------------------+
| user 1      | comment 1 |     3 | 2023-03-01 13:30:00 |
| user 3      | comment 3 |     1 | 2023-03-01 13:32:00 |
| user 2      | comment 2 |     1 | 2023-03-01 13:31:00 |
+-------------+-----------+-------+---------------------+
```
This query selects top levels comments (comments with parent id set to null)
for given video and sorts them by likes and date.
Let's have a look at the `explain` output for this query:
```
+----+-------------+----------+------------+------+--------------------+-----------+---------+-------+------+----------+----------------------------------------------------+
| id | select_type | table    | partitions | type | possible_keys      | key       | key_len | ref   | rows | filtered | Extra                                              |
+----+-------------+----------+------------+------+--------------------+-----------+---------+-------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | comments | NULL       | ref  | video_id,parent_id | parent_id | 9       | const |    6 |    78.57 | Using index condition; Using where; Using filesort |
+----+-------------+----------+------------+------+--------------------+-----------+---------+-------+------+----------+----------------------------------------------------+
```
The `key` column tells us that the `parent_id` index has been used and that 6 rows have been selected
and then filtered down using the `where` cluase.
The problem is that we are selecting all the top level comments for both
video id 1 and 2. This means that as our table grows and we store more comments for more videos
the query will become slower. We can also see in the `Extra` column the `Using filesort` info which means
that mysql is ordering the output.
We can get rid of both these issues by creating an index that directly supports our query:
```sql
create index top_comments on comments (video_id, parent_id, likes desc, created_at desc);
```
Now if we `explain` the query again:
```
+----+-------------+----------+------------+------+------------------------+--------------+---------+-------------+------+----------+-----------------------+
| id | select_type | table    | partitions | type | possible_keys          | key          | key_len | ref         | rows | filtered | Extra                 |
+----+-------------+----------+------------+------+------------------------+--------------+---------+-------------+------+----------+-----------------------+
|  1 | SIMPLE      | comments | NULL       | ref  | parent_id,top_comments | top_comments | 18      | const,const |    3 |   100.00 | Using index condition |
+----+-------------+----------+------------+------+------------------------+--------------+---------+-------------+------+----------+-----------------------+
```
We can see that not only we examined 3 rows (down from 6) which is exactly the number of top level comments
for video id 1. We also got rid of sorting since the index stores the rows in the order that we need.

To support retrieving "Newest first" comments we can create a similar index.

#### Query replies
To retrieve replies for a comment we select all the comments with specific `parent_id` and order it
by date and likes:
```sql
select author_name, content, likes, created_at from comments
where parent_id = '1'
order by created_at desc, likes desc;
```
```
+-------------+-----------+-------+---------------------+
| author_name | content   | likes | created_at          |
+-------------+-----------+-------+---------------------+
| user 1.4    | reply 1.4 |     1 | 2023-03-01 13:30:40 |
| user 1.3    | reply 1.3 |     3 | 2023-03-01 13:30:30 |
| user 1.2    | reply 1.2 |     2 | 2023-03-01 13:30:20 |
| user 1.1    | reply 1.1 |     1 | 2023-03-01 13:30:10 |
+-------------+-----------+-------+---------------------+
```
We don't need to provide the video id since the parent id uniquely identifies comments.\
If we `explain` the query:
```
+----+-------------+----------+------------+------+---------------+-----------+---------+-------+------+----------+---------------------------------------+
| id | select_type | table    | partitions | type | possible_keys | key       | key_len | ref   | rows | filtered | Extra                                 |
+----+-------------+----------+------------+------+---------------+-----------+---------+-------+------+----------+---------------------------------------+
|  1 | SIMPLE      | comments | NULL       | ref  | parent_id     | parent_id | 9       | const |    4 |   100.00 | Using index condition; Using filesort |
+----+-------------+----------+------------+------+---------------+-----------+---------+-------+------+----------+---------------------------------------+
```
We can see that the `parent_id` index is being used to select the replies and then MySQL sorts them.

#### Infinite scrolling
To support infinite scrolling we will use the seek method [[1]](#seek_method).
This method allows us to fetch the next set of comments efficiently.
Basically what we do is take the id of the last displayed comment and
fetch the next batch of comments following that id.
Let's say we have the following set of comments in the database:
```sql
insert into comments(id, video_id, content, author_name, likes, created_at)
values
(1,  1, 'comment 1',  'user',  6,  '2023-03-01 13:30:00'),
(2,  1, 'comment 2',  'user',  7,  '2023-03-01 13:29:00'),
(3,  1, 'comment 3',  'user',  10, '2023-03-01 13:30:00'),
(4,  1, 'comment 4',  'user',  9,  '2023-03-01 13:30:00'),
(5,  1, 'comment 5',  'user',  8,  '2023-03-01 13:30:00'),
(6,  1, 'comment 6',  'user',  7,  '2023-03-01 13:30:00'),
(7,  1, 'comment 7',  'user',  7,  '2023-03-01 13:30:00'),
(8,  1, 'comment 8',  'user',  7,  '2023-03-01 13:30:00'),
(9,  1, 'comment 9',  'user',  5,  '2023-03-01 13:30:00'),
(10, 1, 'comment 10', 'user',  4,  '2023-03-01 13:30:00'),
(11, 1, 'comment 11', 'user',  3,  '2023-03-01 13:30:00'),
(12, 1, 'comment 12', 'user',  2,  '2023-03-01 13:30:00'),
(13, 1, 'comment 13', 'user',  1,  '2023-03-01 13:30:00'),
(14, 1, 'comment 14', 'user',  0,  '2023-03-01 13:30:00');
```
And this is how the whole table looks like when sorted by likes, date, and id:
```sql
select author_name, content, likes, created_at, id from comments
order by likes desc, created_at desc, id asc;
```
```
+-------------+------------+-------+---------------------+----+
| author_name | content    | likes | created_at          | id |
+-------------+------------+-------+---------------------+----+
| user        | comment 3  |    10 | 2023-03-01 13:30:00 |  3 |
| user        | comment 4  |     9 | 2023-03-01 13:30:00 |  4 |
| user        | comment 5  |     8 | 2023-03-01 13:30:00 |  5 |
| user        | comment 6  |     7 | 2023-03-01 13:30:00 |  6 |
| user        | comment 7  |     7 | 2023-03-01 13:30:00 |  7 |
| user        | comment 8  |     7 | 2023-03-01 13:30:00 |  8 |
| user        | comment 2  |     7 | 2023-03-01 13:29:00 |  2 |
| user        | comment 1  |     6 | 2023-03-01 13:30:00 |  1 |
| user        | comment 9  |     5 | 2023-03-01 13:30:00 |  9 |
| user        | comment 10 |     4 | 2023-03-01 13:30:00 | 10 |
| user        | comment 11 |     3 | 2023-03-01 13:30:00 | 11 |
| user        | comment 12 |     2 | 2023-03-01 13:30:00 | 12 |
| user        | comment 13 |     1 | 2023-03-01 13:30:00 | 13 |
| user        | comment 14 |     0 | 2023-03-01 13:30:00 | 14 |
+-------------+------------+-------+---------------------+----+
```
We will query five comments at a time. The initial query for top comments looks like this:
```sql
select author_name, content, likes, created_at, id from comments
where video_id = '1' and parent_id is null
order by likes desc, created_at desc, id asc
limit 5;
```
```
+-------------+-----------+-------+---------------------+----+
| author_name | content   | likes | created_at          | id |
+-------------+-----------+-------+---------------------+----+
| user        | comment 3 |    10 | 2023-03-01 13:30:00 |  3 |
| user        | comment 4 |     9 | 2023-03-01 13:30:00 |  4 |
| user        | comment 5 |     8 | 2023-03-01 13:30:00 |  5 |
| user        | comment 6 |     7 | 2023-03-01 13:30:00 |  6 |
| user        | comment 7 |     7 | 2023-03-01 13:30:00 |  7 |
+-------------+-----------+-------+---------------------+----+
```
The reason we sort by id at the end is that the seek method
that we use requires a deterministic order whereas a combination
of likes and timestamps may not be unique. We can imagine a situation
where two comments are created at the same time and have the same
number of likes.
To fetch the following batches of comments we need to adjust the query:
```sql
select author_name, content, likes, created_at, id from comments
where video_id = '1' and parent_id is null and likes <= 7 and not (likes = 7 and created_at >= '2023-03-01 13:30:00' and id <= 7)
order by likes desc, created_at desc, id asc
limit 5;
```
```
+-------------+------------+-------+---------------------+----+
| author_name | content    | likes | created_at          | id |
+-------------+------------+-------+---------------------+----+
| user        | comment 8  |     7 | 2023-03-01 13:30:00 |  8 |
| user        | comment 2  |     7 | 2023-03-01 13:29:00 |  2 |
| user        | comment 1  |     6 | 2023-03-01 13:30:00 |  1 |
| user        | comment 9  |     5 | 2023-03-01 13:30:00 |  9 |
| user        | comment 10 |     4 | 2023-03-01 13:30:00 | 10 |
+-------------+------------+-------+---------------------+----+
```
In this case the `where` clause is more complicated.
To get the correct result we need to select the comments with
the same number of likes as the last comment (7 in this case). However, by doing that
we may also select comments from the previous batch that had 7 likes.
To filter these comments out we remove the ones with 7 likes, timestamps that
are larger or equal to the last comment (comments are ordered by timestamp in descending order)
and id that is smaller or equal to the last comment (comments are ordered by id in ascending order).

If we `explain` this query:
```
+----+-------------+----------+------------+------+-----------------------------------+------+---------+------+------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys                     | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+----------+------------+------+-----------------------------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | comments | NULL       | ALL  | PRIMARY,id,parent_id,top_comments | NULL | NULL    | NULL |   14 |     7.14 | Using where |
+----+-------------+----------+------------+------+-----------------------------------+------+---------+------+------+----------+-------------+
```
We can see that MySQL decided to not use an index and instead go for a full table scan.
This may happen because even though the columns in the `where` clause are in the
`top_comments` composite index order, we have a range condition `likes <= 7` after which
rest of the index can't be used in which case MySQL decides that a full table scan is faster.
In this case, we can `force index`:
```sql
explain select author_name, content, likes, created_at, id from comments force index (top_comments)
where video_id = '1' and parent_id is null and likes <= 7 and not (likes = 7 and created_at >= '2023-03-01 13:30:00' and id <= 7)
order by likes desc, created_at desc, id asc
limit 5;
```
```
+----+-------------+----------+------------+-------+---------------+--------------+---------+------+------+----------+-----------------------+
| id | select_type | table    | partitions | type  | possible_keys | key          | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+----------+------------+-------+---------------+--------------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | comments | NULL       | range | top_comments  | top_comments | 23      | NULL |   11 |   100.00 | Using index condition |
+----+-------------+----------+------------+-------+---------------+--------------+---------+------+------+----------+-----------------------+
```
### Bibliography
- <a name="seek_method"></a>[1] [https://use-the-index-luke.com/sql/partial-results/fetch-next-page](https://use-the-index-luke.com/sql/partial-results/fetch-next-page)