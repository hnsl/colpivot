# Colpivot
*Dynamic row to column pivotation/transpose in Postgres made simple.*

Author: Hannes Landeholm <hannes@jumpstarter.io>

`colpivot.sql` defines a single Postgres function:

    create or replace function colpivot(
        out_table varchar, in_query varchar,
        key_cols varchar[], class_cols varchar[],
        value_e varchar, col_order varchar
    ) returns void

The `colpivot()` function groups data by a specific key and returns a column
for every unique class with a specified value expression in a new destination
temporary output table. The result is returned in a new temporary table.

This is the similar to the problem solved by
[crosstab](http://www.postgresql.org/docs/9.2/static/tablefunc.html).
However, `colpivot()` can be used with *completely* dynamic data. With crosstab
you MUST know what categories/classes you have before hand. The crosstab
function is also incompatible with multiple key or category/class columns.

Overall, the benefits of `colpivot()` benefits are:

* Completely dynamic, no row specification required.
* Supports multiple rows and classes/attributes columns.
* Gives complete control over output columns order and limit.
* Easier to understand and use.
* Does not require loading extra modules.
* Does not require writing extra functions.

See also [this thread on Stack Overflow](http://stackoverflow.com/questions/2099198/sql-transpose-rows-as-columns).

Essentially the function transposes query results with columns like:

     key1, key2, ..., keyN, class1, class2, ... classN, *

to:

     key1, key2, ..., keyN, classC1, classC2, ..., classCN

The output is undefined if the input query has more than one row with the
same `(key1, key2, ..., keyN, class1, class2, ... classN)` value. In most
real world use cases of `colpivot()` it is sensible to have a primary key
or unique index over these columns when the input query selects a table
directly or using an input query with a distinct/group by over these columns
when uniqueness is not guaranteed.

* The output will have one row per unique key combination.
* The output will have one class column per unique class combination.
* The value of the corresponding class column is the result of the specified
expression or `null` if the corresponding class combination has no associated
key.

To avoid having to specify an output column definition (since there is many
real world cases where this is not possible to say beforehand) the result is
stored in a temporary table with the specified name. It is impossible to return
a result set with dynamic columns in Postgres without using temporary tables.
The resulting temporary table is deleted when the transaction ends (on commit
drop).

## Parameter reference:

* **`out_table`** - Unquoted new temporary table to create.
* **`in_query`** [1] - Query to run that generates source data to colpivot().
* **`key_cols`** - Array of unquoted key columns.
* **`class_cols`** - Array of unquoted class columns.
* **`value_e`** [1] - Value expression. You must use the `#` token as an alias
if you are referencing a column in the input result.
For example, specify `#.salary'` instead of `'salary'`.
* **`col_order`** [1] - Column order. Specify as null to simply use the sorted
classes. This is useful if you want another specific column order.
For example, you may want to have the class with the highest salary as the
first column. In that case you can specify `sum(salary) desc`.
You can also (ab)use this parameter to limit the number of columns returned
like: `sum(salary) desc limit 10` to only get the 10 highest salaries.

[1] These parameters are concatenated directly in evaluated queries to allow
maximum flexibility for the caller and therefore unsafe. Ensure that you
are not feeding dirty/non-validated/unquoted data into these parameters
as that will allow a SQL injection exploit.

## Example / Test

    begin;

    create temp table _test (
        year int,
        month int,
        country varchar,
        state varchar,
        income int
    ) on commit drop;

    insert into _test values
        (1985, 01, 'sweden', '', 10),
        (1985, 01, 'denmark', '', 11),
        (1985, 01, 'usa', 'washington', 13),
        (1985, 02, 'sweden', '', 20),
        (1985, 02, 'usa', 'washington', 21),
        (1985, 03, 'sweden', '', 34),
        (1985, 03, 'denmark', '', 31),
        (1985, 03, 'usa', 'washington', 39),
        (1990, 12, 'sweden', '', 42),
        (1990, 12, 'denmark', '', 43),
        (1990, 12, 'usa', 'washington', 49),
        (1990, 12, 'germany', '', 45);

    select colpivot('_test_pivoted', 'select * from _test',
        array['year', 'month'], array['country', 'state'], '#.income', null);

    select * from _test_pivoted order by year, month;

    -- returns:
    --  year | month | 'denmark', '' | 'germany', '' | 'sweden', '' | 'usa', 'washington'
    -- ------+-------+---------------+---------------+--------------+---------------------
    --  1985 |     1 |            11 |               |           10 |                  13
    --  1985 |     2 |               |               |           20 |                  21
    --  1985 |     3 |            31 |               |           34 |                  39
    --  1990 |    12 |            43 |            45 |           42 |                  49
    -- (4 rows)

    rollback;

## Licence

MPLv2 (https://www.mozilla.org/MPL/2.0/)
