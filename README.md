# Lab: Transactions

In the [consistency lab](https://github.com/mikeizbicki/lab-consistency), you created a simple database for a double-entry accounting system.
In this lab, we will extend this database to have a simple python library interface.
You will see:

1. that database code that looks "obviously correct" can be horribly flawed,
2. how to fix these flaws using transactions and locks, and
3. how using the wrong lock can slow down your code.

Because of the difficulty and importance of this topic,
you should be extra careful in this lab.

<img src=img/pitfalls1.jpg width=300px>

## Task 0: Project Setup

You will work with multiple partners throughout this lab.
There are 4 tasks below, each of which involves modifying some python.
You will need 2 different partners for completing tasks 2 and 3.

### Getting the code

Create a new repo in github.
Do not fork this repo---you won't be the main "owner" of the repo and that will make subsequent git steps harder.

Since you didn't fork,
you'll have to run some simple git incantations to get the code into your repo.
The following should do the trick:
```
$ git clone <your_repo_url>
$ cd <your_repo>
$ git remote add upstream <lab_url>
$ git pull upstream master
$ git push origin master
```

### Bringing up the Database

We will use postgres as our database for this lab.
Try to bring up the database with the command
```
$ docker-compose up -d
```
You should get an error about a bad port number.

In previous projects, this never happened because the database port was never exposed to the lamdba server.
In this project, however, the database needs to be exposed to the lambda server.
That's because we will be running python code on the lambda server (and not the `pg` container) that needs to connect to the database.

Edit the `ports` field of the `docker-compose.yml` file so that your database will be exposed on a unique port that doesn't conflict with other students.
(Your UID is a reasonable choice, and you can find it by running `id -u` in the shell.)
Then re-run
```
$ docker-compose up -d
```
and ensure that you get no errors.
Verify that you are able to successfully connect to the database using psql and the command
```
$ psql postgresql://postgres:pass@localhost:<PORT>
```
where `<PORT>` should be replaced with the port number in your `docker-compose.yml` file.
Running the command
```
postgres=# \d
```
should list a handful of tables.

> **NOTE:**
> The `psql` command above is running directly on the lambda server and not inside the container.
> The long url passed into `psql` tells psql how to connect to the database inside the container.
> Previously, we were running psql inside the container using a command like
> ```
> $ docker-compose exec pg psql
> ```
> This `psql` incantation runs inside the container (because of `docker-compose exec pg`), and so the url is not needed.
> Both of these two commands are essentially equivalent.
> All of the SQL commands you run from inside `psql` will have the exact same effects no matter how you get inside of `psql`.

### The Schema

The most important file in any project working with databases is the `.sql` file containing the schema.
This project's schema is stored in `services/pg/sql/ledger-pg.sql`.
The `Dockerfile` automatically loads this sql file into the database when the database first starts.

Let's take a look at the contents:
```
$ cat services/pg/sql/ledger-pg.sql
CREATE TABLE accounts (
    account_id SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE transactions (
    transaction_id SERIAL PRIMARY KEY,
    debit_account_id INTEGER REFERENCES accounts(account_id),
    credit_account_id INTEGER REFERENCES accounts(account_id),
    amount NUMERIC(10,2),
    CHECK (amount > 0),
    CHECK (debit_account_id != credit_account_id)
);

CREATE TABLE balances (
    account_id INTEGER PRIMARY KEY REFERENCES accounts(account_id),
    balance NUMERIC(10,2)
);
```
This file is very similar to the schema from the consistency lab.
The `accounts` and `transactions` tables are exactly the same.
Recall that `accounts` just stores the names of all of our accounts,
and `transactions` stores every transfer of money between two accounts.

The `balances` table is new.

Recall that in the last lab instead of having a `balances` table, we created a `balances` view.
The view computed the total balance in each account by summing over the `transactions` table.
It looked like
```
CREATE VIEW balances AS
SELECT
    account_id,
    name,
    coalesce(credits, 0) as credits,
    coalesce(debits, 0) as debits,
    coalesce(credits, 0) - coalesce(debits, 0) AS balance
FROM accounts
LEFT JOIN (
    SELECT credit_account_id as account_id, sum(amount) as credits
    FROM transactions
    GROUP BY credit_account_id
) AS credits USING (account_id)
LEFT JOIN (
    SELECT debit_account_id as account_id, sum(amount) as debits
    FROM transactions
    GROUP BY debit_account_id
) AS debits USING (account_id)
ORDER BY account_id
;
```
It turns out that this view requires $\Omega(n)$ time to compute,
where $n$ is the total number of transactions in our history.

> **ASIDE:**
> We will see in the coming weeks that this query is implemented using an algorithm called SEQUENTIAL SCAN.
> This algorithm is basically a for loop over the entire `transactions` table,
> and that's where the $\Omega(n)$ runtime comes from.

For the small databases we used in the last lab,
that wasn't a problem.
But in the real world, this would be a problem.
Real accounting systems can have trillions of transactions stored in them,
and so an $\Omega(n)$ operation would be very slow.
We need something that will take $O(1)$ time.

> **NOTE:**
> I am being formally correct with the $O$ and $\Omega$ notaion above.
> Recall that $\Omega$ is a *lower bound* and $O$ is an upper bound,
> and so when we say that a runtime is $\Omega(n)$ we are saying that it cannot be any faster than linear.
>
> Many intro data structures classes do not teach the technically correct versions of $O$ and $\Omega$ and rely only on intuition.
> Wikipedia has an excellent article explaining these distinctions that you are welcome to review: <https://en.wikipedia.org/wiki/Big_O_notation#Formal_definition>.

The `balances` table will let us achieve this faster lookup through *caching*.
The basic idea is that we should pre-compute the balances as we insert the transactions.
That is, whenever we insert a new transaction,
we should also update the corresponding rows in the `balances` table at the same time.
That way, when we want the balance, all we need to do is look at a single row in the `balances` table instead of summing over the entire `transactions` table.

This type of caching is very widely used in realworld databases.
In postgres, these cached tables are often colloquially referred to as [rollup tables](https://www.citusdata.com/blog/2018/06/14/scalable-incremental-data-aggregation/).

> **NOTE:**
> This is confusing terminology because there is a [ROLLUP sql command](https://www.educba.com/postgresql-rollup/) that is totally unrelated to the idea of a rollup table.
> The naming of a rollup table was invented by a group of postgres developers at a data analytics company called citus.
> Citus is famous company in the postgres world for their efficient large scale postgres products and tutorials,
> and so their idiosyncratic naming has become standard.
> [Citus was acquired by Microsoft in 2019](https://blogs.microsoft.com/blog/2019/01/24/microsoft-acquires-citus-data-re-affirming-its-commitment-to-open-source-and-accelerating-azure-postgresql-performance-and-scale/) and their tools now are the backend for many big data projects at Microsoft.

### Adding Accounts (I)

The file `Ledger/__init__.py` contains our library's python code.
In this section, we'll see how to use this library to manipulate the database.

First, we'll verify there are no accounts in the database.
Run the following commands
```
$ psql postgresql://postgres:pass@localhost:<PORT>
postgres=# select * from accounts;
 account_id | name
------------+------
(0 rows)
```
Next, we will open python in interactive mode,
create a new `Ledger` object,
and run the `create_account` method on that object.
```
$ python3
>>> import Ledger
>>> ledger = Ledger.Ledger('postgresql://postgres:pass@localhost:<PORT>')
>>> ledger.create_account('test')
2024-03-22 00:09:04.560 - 55670 - INSERT INTO accounts (name) VALUES (:name);
2024-03-22 00:09:04.565 - 55670 - SELECT account_id FROM accounts WHERE name=:name
2024-03-22 00:09:04.568 - 55670 - INSERT INTO balances VALUES (:account_id, 0);
```
The library is structured so that every time it runs a SQL command, it logs those commands to the screen for you to see.
You can see the timestamp that the command run, followed by the process id of the command, followed by the actual command.

> **NOTE:**
> The `import` command above is likely to give you an error about missing libraries.
> You will need to use the `pip3 install` command to install these libraries.

Here, we can see that three SQL commands were run by the `create_account` method.
Open the file `Ledger/__init__.py` and read through the `create_account` method to understand why three SQL statements were run.

Now reconnect to psql and verify that an account has been created.
```
$ psql postgresql://postgres:pass@localhost:<PORT>
postgres=# select * from accounts;
 account_id | name
------------+------
          1 | test
(1 row)
```
### Adding Accounts (II)

We're going to be creating some test cases soon.
To do that, we'll need an automated way of populating the database with accounts.
The file `scripts/create_accounts.py` calls the `create_account` method in a for loop to do this for us.

First, reset the database by bringing it down and back up.
```
$ docker-compose down
$ docker-compose up -d
$ psql postgresql://postgres:pass@localhost:<PORT>
postgres=# select * from accounts;
 account_id | name
------------+------
(0 rows)
```
Now run the command
```
$ python3 scripts/create_accounts.py postgresql://postgres:pass@localhost:<PORT> 
```
You probably get an error message that looks something like
```
Traceback (most recent call last):
  File "scripts/create_accounts.py", line 1, in <module>
    import Ledger
ModuleNotFoundError: No module named 'Ledger'
```
This is because when python is running a script, it defaults to assuming all `import` commands are loading installed libraries.
When we were running in interactive mode above, this was not the case, and the `import` command correctly looked in our current folder.

In order to get python to look into our current folder, we need to set the `PYTHONPATH` environment variable with the following command
```
$ export PYTHONPATH=.
```
Now, rerun the `create_accounts.py` script.
You should see a lot of output of `SELECT` and `INSERT` statements.

> **NOTE:**
> `import` statements in python are notoriously counterintuitive.
> There are *many* ways that `import` looks for modules,
> and basically no one fully understands how the `import` command works.
>
> As always, there's an [xkcd](python_environment_2x.png) for that:
>
> <img width=400x src=img/python_environment_2x.png />

Connect to psql and run the command
```
SELECT count(*) FROM accounts;
```
You should see that 1000 accounts have been created.

The tasks below will occasionally ask you to reset the database.
To do so, you'll need to bring it down, then back up, then recreate these accounts.

## Task 1: Correctness

We saw in the last lab that the database is able to automatically enforce certain types of correctness.
But there are other types of correctness that no database can check automatically.

In our this project, one of the properties of our `balances` table is that
```
SELECT sum(balance) FROM balances;
```
should always be 0.
Make sure you understand why before continuing.

> **NOTE:**
> A natural question to ask is:
> why can't a database enforce this constraint automatically?
> The answer is that arbitrary constraints can take an arbitrary amount of compute to verify.
> (The SELECT statement above requires $\Omega(n)$ runtime because we must touch every row in the database to compute it.)
> If we verified these expensive constraints everytime we inserted data into the database,
> the database would be very slow.
> Therefore, the SQL standard only supports automatic checks that run in $O(\log n)$ time.
> The `CHECK` constraint, for example, is a row-only constrain and runs in time $O(1)$;
> The `UNIQUE` constraint uses an index (which we will learn about soon) to run in time $O(\log n)$.

It is common practice to document these *invariants* that a database should maintain by writing scripts that verify the invariant.
The script `scripts/integrity_check.sh` verifies that the above invariant is maintained.
Run it.
```
$ sh scripts/integrity_check.sh
sum(balance) = 0.00
PASS
```
> **NOTE:**
> If your command has no output,
> then look for an error in the way docker compose is being called.

At this point, we haven't made any transactions.
All the balances are initialized to 0,
and so the script passes by default.

### The Problem

Run the command
```
$ python3 scripts/random_transfers.py postgresql://postgres:pass@localhost:<PORT>
```
You should see a large number of SQL commands scroll through your screen.
This script performs 1000 random transfers between accounts by calling the `Ledger.transfer_funds` method.
(I recommend you read through the source and understand it before continuing.)

Unfortunately, the `Ledger.transfer_funds` method is currently incorrect.
Rerun the integrity check.
```
$ sh scripts/integrity_check.sh
```
You should see that the sum of the balances is non-zero,
and that the check fails.
The `random_transfers.py` script is nondeterministic,
so everyone will have different sums,
but they should all be non-zero.

### The Solution

Modify the `transfer_funds` method so that it is correct.

To test your solution, run the following commands to reset the database and then rerun the test scripts.
```
$ docker-compose down
$ docker-compose up -d
$ python3 scripts/create_accounts.py postgresql://postgres:pass@localhost:<PORT> 
$ python3 scripts/random_transfers.py postgresql://postgres:pass@localhost:<PORT>
$ sh scripts/integrity_check.sh
```
You won't be able to complete the next task until these checks pass.

## Task 1-2 Interlude

Push your changes to github.
Then find a partner for Task 2 below.
You should clone your partner's repo (that has the task 1 changes)
and complete all the changes mentioned in task 2 inside their repo.

## Task 2: Correctness (With Failing Processes)

[Chaos monkey](https://netflix.github.io/chaosmonkey/) is a famous netflix tool for testing robust systems.

<img src=img/chaosmonkey.png />

Chaos monkey works by randomly killing running processes,
and then checking to see if there was any data corruption.
The idea is to simulate that there is a "chaos monkey" running around the server room randomly smashing computers with a hammer,
and netflix wants to be robust to this particular type of failure.

We will will use our own "mini chaos monkey" in this lab to test the robustness of the code you wrote for the previous task.

### The Problem

Run the command
```
$ sh scripts/chaosmonkey_sequential.sh postgresql://postgres:pass@localhost:<PORT> 
```
This file runs the `scripts/random_transfers.py` file in a loop,
but kills each process after only 1 second.
(I recommend you read through the source and understand it before continuing.)

The database will now likely once again fail the integrity check.
```
$ sh scripts/integrity_check.sh
```
This is because your `transfer_funds` method is not atomic.
If the python process is killed while it is the middle of this function,
then only some of the UPDATE/INSERT commands will take effect and not others.

### The Solution

To make your code atomic, you need to wrap it in a transaction.

Using the SQLAlchemy library, we don't directly call the `BEGIN` and `COMMIT` SQL commands.
Instead, we use the `connection.begin()` method to create a transaction.
This is commonly done inside of a `with` block so that the transaction is automatically committed when the block closes.
The code will look something like
```
with self.connection.begin():
    # insert SQL commands here
```
The provided `create_account` method is atomic,
and you can reference this function as an example.

Once you've fixed the `transfer_funds` method,
rerun the test script to verify that the integrity check is now maintained.

<img width=300px src=img/monkeys.gif />

The incantation might look something like:
```
$ docker-compose down
$ docker-compose up -d
$ python3 scripts/create_accounts.py postgresql://postgres:pass@localhost:<PORT>
$ sh scripts/chaosmonkey_sequential.sh postgresql://postgres:pass@localhost:<PORT>
$ sh scripts/integrity_check.sh
```

Like before, you won't be able to complete the next task until these checks pass.

## Task 2-3 Interlude

Push your changes to github and then issue a pull request to your partner's repo.
Once you've accepted their pull request, you'll need to find a *different* partner for task 3 below.
Fork their repo, and make the changes described in task 3 to their code.

## Task 3: Correctness (With Concurrency)

Transactions prevent certain types of data corruption,
but not all types of data corruption.
In this section, we will introduce a parallel version of the chaos monkey script.
We'll see that your corrected code you wrote above will still corrupt the database when run concurrently,
and we'll need a lock to fix the problem.

### The Problem

Run the command
```
$ sh scripts/chaosmonkey_parallel.sh postgresql://postgres:pass@localhost:<PORT> 
```
This file runs many instances of the `scripts/random_transactions.py` file concurrently.
Then after waiting 10 seconds, it kills all of the running processes.
(I recommend you read through the source and understand it before continuing.)

The database will now likely once again fail the integrity check.
```
$ sh scripts/integrity_check.sh
```
This is because of a *race condition* in the code.

A **race condition** is a bug whose presence depends on the particular order in which concurrent processes happen to interleave their operations.
The `transfer_funds` method contains one because the SELECT-then-UPDATE sequence on the `balances` table is atomic with respect to crashes,
but *not* with respect to other concurrent transactions.

Consider what happens if we make two transfers to the same account at the same time.
The account (we'll call it `A`) starts with 100 dollars; T1 debits $30 and T2 debits $50:
```
T1: SELECT balance WHERE account_id=A;  --> 100
T2: SELECT balance WHERE account_id=A;  --> 100     (both read old value)
T1: UPDATE balances SET balance=70  WHERE account_id=A;   (100-30 calculated inside python)
T2: UPDATE balances SET balance=50  WHERE account_id=A;   (100-50 calculated inside python)  <-- T1's write lost
```
Both transactions commit without error.
The `transactions` table correctly records two debits totaling 80,
but `balances` shows only 50 debited---the 30 debited by T1 was silently lost.
Now the `SELECT sum(balance)` invariant fails.
This type of race condition is called a **lost update**.

### The Solution

The fix is to make the read-modify-write sequence atomic against other writers.
We do that with a LOCK.

At the top of your `transfer_funds` method,
add a SQL command that locks the `balances` table in ACCESS EXCLUSIVE MODE.
This will ensure that only one process is able to write to the table at a time.

Once you've made the necessary changes,
verify they work by rerunning the `chaosmonkey_parallel.sh` script and then verifying the integrity check.

### Performance

To measure the performance of our application, we can measure the total number of transactions that were inserted in the 10 seconds of the chaos monkey script.
Run the SQL command
```
SELECT count(*) FROM transactions
```
Make a note of the result so you can compare it to the result in the next section.
My solution got a result around 1500.

## Task 3-4 Interlude

Once again, get your changes to your partner with a pull request.
You will do Task 4 below back in your own repo.
You will have to pull the changes that your partner's made to your code back to the lambda server.

## Task 4: More Speed

Finally, our code is correct!
But unfortunately, it's slow.
We will now see how to speed it up and actually get the correct data inserted.

### The Problem

The ACCESS EXCLUSIVE lock ensures that only one process can access the `balances` table at a time.
This causes two types of problems.

The first is related to "realtime" systems,
where the database is being updated in realtime by real users.
As an example, imagine if a credit card company like Visa or Mastercard implemented their accounts ledger this way with an ACCESS EXCLUSIVE lock.
Then only one person in the world would be able to use a credit card at a time.
That's obviously not good from a business perspective.

The second problem is related to data warehousing.
Imagine we have a large dataset (like the Twitter dataset) that we want to load into a database.
We would like to do this in parallel with many processes to speed up the insertion.
But if we use ACCESS EXCLUSIVE locks to guarantee correctness,
then only one process can run at a time,
and we can't get any parallel speedup.

### The Solution

The ACCESS EXCLUSIVE lock is a table-level lock and is too restrictive for our purposes.
A row-level lock would ensure that two transactions don't overwrite the balance of a single user,
while still allowing two transactions to write to two different users in different rows.

The SELECT/UPDATE pattern in the `transfer_funds` method is an extremely common pattern in database applications.
(And, as we've seen, an extremely common source of very subtle bugs!)
Postgres has a special SELECT FOR UPDATE syntax that simplifies this pattern.

To use the row level lock:
1. Comment out the LOCK statement that you added in the previous task.
2. Modify the SELECT statements to use the FOR UPDATE clause.

    The FOR UPDATE clause is added to the end of SELECT statements,
    so the overall commands will have the form
    ```
    SELECT columns FROM table WHERE condition FOR UPDATE
    ```

Once you've made the necessary changes,
verify they work by rerunning the `chaosmonkey_parallel.sh` script and then verifying the integrity check.

### The Solution (part b)

Your code is now fast... but it's incorrect again!!!

When you run the `chaosmonkey_parallel.sh` script,
you should see a large number of deadlock errors being reported.

> **NOTE:**
> These error messages can be hard to find because they are hiding in the sea of debug messages that you have been printing.
> You can disable the debug messages by modifying the line
> ```
> level=logging.DEBUG,
> ```
> to
> ```
> level=logging.WARNING,
> ```
> inside the configuration of the `logging` module at the top of the script.
> After making this change, you will see the deadlock errors being printed to the screen because they will not be drowned in a sea of debug statements.

Recall that a *deadlock* occurs when two processes are each blocked by the other process,
and without any intervention both processes will be waiting forever.
Postgres solves this problem by arbitrarily picking one of the processes to send a deadlock error... then sqlalchemy will raise an exception.
If you do not catch this exception,
then python will terminate early and unexpectedly.
This will cause your script to not insert any more data points.
You will pass the integrity check above, but you will still have the wrong contents in the database!
This is like Visa just denying your credit card transaction because of a deadlock... that's better than corrupting their internal database, but still not good from a business perspective.

To fix this problem, you need to wrap your code in a try/except block and catch the error.
Repeat the insertion command until both the SELECT and UPDATE command jointly succeed without error.

### Verifying Speed Boost

Now let's verify that we are in fact inserting more transactions with the FOR UPDATE version of the code.
Run the SQL command
```
SELECT count(*) FROM transactions;
```
to count the total number of transactions inserted with your improved FOR UPDATE code.

You should get a number significantly larger than you got in the previous task.
The `chaosmonkey_parallel.sh` script runs 100 processes in parallel, so you should theoretically see a 100x increase in the number of rows inserted.
Due to overhead issues, you might see only an 70-90x increase.
If your increase is smaller than that,
then you probably did not solve the deadlock issue correctly.

## Takeaway

Inserting data into databases correctly is hard.
There are many subtle ways to get code that looks correct,
but generates incorrect results in the presence of crashes or concurrency.
Transactions and locks are our only tools to solve these problems.
But they are hard to use too :(

Writing scripts that test the integrity of your data is one of the few useful methods we have for debugging these types of problems.
Whenever you have a dataset that is supposed to maintain some sort of invariant,
you should always write a script that tests that invariant.

## Submission

Git has a convenient tool to show who made changes to a particular file called `git blame`.
Run it on your python code:
```
$ git blame ./Ledger/__init__.py
```
You will see that most of the code is written by me, and a bit by your partners.
(Depending on how you did Task 4, you might have overwritten your Task 3 partner's contribution, and so they might not appear in the `git blame` output.
But your Task 2 partner's code probably should.)

Every git command like this that shows useful debug info about a repo has a corresponding github endpoint.
Find the corresponding github endpoint for your `Ledger/__init__.py` file in your repo.
Submit that url to canvas.

I will view the blame output to check that everyone's contributions are correct when grading.
