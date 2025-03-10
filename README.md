## Precise fractions for PostgreSQL

An efficient custom type. Perfect for exact arithmetic or user-specified
table row ordering. Holds values as big as an integer, with matching
precision in the denominator.

### Features

* Stores fractions in exactly 64 bits (same size as float)
* Written in C for high performance
* Detects and halts arithmetic overflow for correctness
* Uses native CPU instructions for fast overflow detection
* Defers GCD calculation until requested or absolutely required
* Supports btree and hash indices
* Implements Stern-Brocot trees for finding intermediate points
* Coercion from integer/bigint/tuple
* Custom aggregate

### Motivation

See my blog post about [User-Defined Order in
SQL](https://begriffs.com/posts/2018-03-20-user-defined-order.html).

### Usage

Basics

```sql
-- fractions are precise
-- this would not work with a float type
select 1::rational / 3 * 3 = 1;
-- => t

-- provides the usual operations, e.g.
select '1/3'::rational + '2/7';
-- => 13/21

-- helper "ratt' type to coerce from tuples
select 1 + (i,i+1)::ratt from generate_series(1,5) as i;
-- => 3/2, 5/3, 7/4, 9/5, 11/6

-- simplify if desired
select rational_simplify('36/12');
-- => 3/1

-- convert float to rational
select 0.263157894737::float::rational;
-- => 5/19

-- convert rational to float
select '-1/2'::rational::float;
-- => -0.5
```

Next, reordering items without renumbering surrounding items.

Note we use an integer sequence rather than the default of bigint, and
explicitly cast `nextval()`. There is no conversion from bigint to rational
because numerators in this extension can hold at most integer range anyway.

```sql
create sequence todos_seq as integer;

create table todos (
  prio rational unique
    default nextval('todos_seq')::integer,
  what text not null
);

insert into todos (what) values
  ('install extension'),
  ('read about it'),
  ('try it'),
  ('profit?');

select * from todos order by prio asc;
/*
┌──────┬───────────────────┐
│ prio │       what        │
├──────┼───────────────────┤
│ 1/1  │ install extension │
│ 2/1  │ read about it     │
│ 3/1  │ try it            │
│ 4/1  │ profit?           │
└──────┴───────────────────┘
*/

-- put "try" between "install" and "read"
update todos
set prio = rational_intermediate(1,2)
where prio = 3;

select * from todos order by prio asc;
/*
┌──────┬───────────────────┐
│ prio │       what        │
├──────┼───────────────────┤
│ 1/1  │ install extension │
│ 3/2  │ try it            │
│ 2/1  │ read about it     │
│ 4/1  │ profit?           │
└──────┴───────────────────┘
*/

-- put "read" back between "install" and "try"
update todos
set prio = rational_intermediate(1,'3/2')
where prio = 2;

select * from todos order by prio asc;
/*
┌──────┬───────────────────┐
│ prio │       what        │
├──────┼───────────────────┤
│ 1/1  │ install extension │
│ 4/3  │ read about it     │
│ 3/2  │ try it            │
│ 4/1  │ profit?           │
└──────┴───────────────────┘
*/
```

This extension uses Stern-Brocot trees to find efficient intermediate points as fractions in lowest terms. It can continue to split deeper between fractions as much as any practical application requires.

Using floats, on the other hand, and picking the midpoints between adjacent values runs out of space rapidly (you only need 50-odd inserts at the wrong spot to start hitting problems).

### Installation

Clone this repo, go inside and simply run:

```bash
make
sudo make install
```

Then, in your database:

```sql
create extension pg_rational;
```

### Docker Image
```bash
docker pull mnafisiasl/be-rational-postgres:15.2
```

be-rational-postgres is Postgres official image with pg_rational installed.

[GitHub Repo](https://github.com/MRNafisiA/be-rational-postgres)  
[Docker Repo](https://github.com/MRNafisiA/be-rational-postgres) 

pg_rational is only installed, so do not forget to enable it:

```sql
create extension pg_rational;
```

### Caveats

The `rational_intermediate` function is super fast on typical intervals, but the narrower the range between arguments the longer it takes. We may want to add a max search depth parameter to prevent malicious values from hogging the server.

### Thanks

This is my first PostgreSQL extension, and these resources were helpful in learning to write it.

* https://www.postgresql.org/docs/10/static/extend-extensions.html
* https://www.postgresql.org/docs/10/static/xtypes.html
* http://big-elephants.com/2015-10/writing-postgres-extensions-part-i/
* https://wiki.postgresql.org/wiki/User-specified_ordering_with_fractions
* #postgresql and ##c channels on freenode
