# SQL Injection
## 1. SQL injection vulnerability in WHERE clause allowing retrieval of hidden data
<img src="./sql-where.png" width="400">

### Description
The lab entailed exploiting the SQL query `SELECT * FROM products WHERE category = '{category}' AND released = 1` through a SQL injection vulnerability.

### Solution
Encoding `'OR released=0` as `'OR%20released%3d0%20--` and submitting it as a parameter to `/filter?category=` was sufficient.
`/filter?category='OR%20released%3d0%20--`

---

## 2. SQL injection vulnerability allowing login bypass
<img src="sql-login.png" width="400">

### Description
The lab entailed exploiting a vulnerable login form through SQL injection. The goal was to force a login to the account `administrator`.

### Solution
Submitting the form with the username `' OR username='administrator' AND 1=1 --` and the password as any valid string bypassed the login.

---

## 3. SQL injection attack, querying the database type and version on Oracle
<img src="./oracle-version.png" width="400">

### Description
The lab entailed exfiltrating the version of Oracle SQL from the server by injecting a payload through the URL parameters.

### Solution
The original SQL query is something along the lines of: `SELECT * FROM table WHERE category='{category}' ...`. \
With the above information, we can first submit a non-malicious request and observe how many columns we expect the table to have. In this lab, the following is returned upon entering `category=Gifts`:

<img src="./oracle-gifts.png" width="400">

We can observe a list of four items, each with a title and description. Therefore, we could perhaps guess that it has 2 or 3 columns, depending on whether or not this table has an `id` column.

Next, we construct another command to retrieve the version using the `UNION` keyword. We can retrieve the version from a view labelled `v$version`, under the column `banner`. Therefore, we can use `SELECT banner FROM v$version` under normal circumstances to retrieve the version.

However, because we have to use `UNION` to execute this command, the must ensure that the 2nd `SELECT` command returns the same number of **compatible** columns as the 1st. We may "pad" the number of columns by adding `NULL` (type-inferred) as a parameter for the `SELECT` keyword: `SELECT banner, NULL FROM v$version`. 

For this lab, I simply guessed that the `products` table had two columns, for which the following URL query payload became the final solution: `'%20UNION%20SELECT%20banner,%20NULL%20FROM%20v$version%20--`

---

## 4. SQL injection attack, querying the database type and version on MySQL and Microsoft
<img src="./mysql-version.png" width="400">

### Description
Similar to question [3](#3-sql-injection-attack-querying-the-database-type-and-version-on-oracle), but the underlying database is MySQL.

### Solution
There are only two differences from question. Instead of `SELECT banner FROM v$version`, MySQL can directly use `SELECT VERSION()`. This can be adapted to `SELECT VERSION(), NULL FROM dual` to support the union. Another difference is that the comment marker `--` **must** be followed by whitespace in order to start a comment. 

The final solution is the following payload: `'+UNION+SELECT+VERSION(),+NULL+FROM+v$version+--+`

I have also started to use `+` instead of `%20` for the sake of ~~my sanity~~ readability.

---

## 5. SQL injection attack, listing the database contents on non-Oracle databases

### Description
Difficulty ramped a bit with this one.
The lab entailed exfiltrating details about the tables present within the database along with the columns of a particular table, on top of exfiltrating data out of that particular table. The goal was to find the account details for the user `administrator`.

### Solution
Before this question, we have only had access to the `products` table. However, we need to retrieve user details in this question. Therefore, we needed a way to find all the existing tables on the database. It was specified that the database was non-Oracle, so we may retrieve all database names with the query `SELECT table_name FROM information_schema.tables`. We can apply the previous tricks to fit this into a union in the context of this lab: `UNION SELECT table_name, NULL FROM information_schema.tables`. 
The payload to retrieve all database names is as follows:
`'+UNION+SELECT+table_name,+NULL+FROM+information_schema.tables+--+`

Submitting the above payload results in the following (I started using BurpSuite as it was a lot faster to repeatedly test):
<img src="union-login-burp-tables.png" width=400>

As seen, we managed to get the database to regurgitate all of its tables onto the HTML. I did a quick `CTRL+F` for the keyword "user", and found one that seems to be the most likely candidate: `users_rrudmq`. It seems like they've attached the hash value `rrudmq` to the table name so that people couldn't just guess!

The next step was to now check for all the columns. Due to the `UNION`, we have a hard limit of **two** columns to retrieve from the table. A `users` table may contain many more than two columns. Incidentally, we may still use the `information_schema` system database to query for the columns of a table, using `information_schema.columns`. 
Therefore, the final payload for this is:
`'+UNION+SELECT+column_name,+NULL,FROM,information_schema.columns+WHERE+table_name='users_rrudmq'+--+`

Upon submitting this payload using BurpSuite, the following is what I received:
<img src="./union-login-burp-users.png" width=400>

As seen, even the column names had hash values attached to them to prevent guessing. There's even an extra "email" column to prevent using the wildcard operator to simply fetch everything (as it would cause an error in the `UNION` because the `products` table only had 2 columns).

As the final step, I only had to fetch the columns `username_kthzzr` and `password_vxdqhk` from the table `users_rrudmq`. 
The following payload solved this:
`'+UNION+SELECT+username_kthzzr,+password_vxdqhk+FROM+users_rrudmq+--+`

And with the details fetched, I logged into the account and solved the lab.