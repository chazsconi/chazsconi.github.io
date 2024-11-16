---
layout: post
title:  "Generating slugs in PostgreSQL"
date:   2024-11-16 10:00:00 +1:00
published: true
---

![Garden slug]({{ site.url }}/assets/slugs-in-postgres/garden-slug.png){:style="border: 1px solid #ddd; width: 80%;"}

This is not about the garden pests, but rather about generating short, unique, difficult to guess alphanumeric ids that can be used as an identifier in a URL, 
for example for references to shortened URLs, or a link to join a video call. e.g.
``https://my-url-shortner/fwugnmhm7o``.


The requirement for these slugs is that they should be:
1. Short
2. Unique
3. Not easily guessable - the user should not be able to just increment the value and get another valid slug.

There are a couple of ways that this could be done:

## Potential solutions

## Generate a hash

In the case of a video call the ids or email addresses of the recipients could be used to generate a hash.  However, to avoid collisions 
it would probably not be sufficient to use a short hash algorithm like `md5`, but use `sha1` or `sha256`, however this generates very long hashes.

## Random number and check for duplicates

Another solution would be to generate a random number, and then base-36 or base-64 encode it, and then check for any existing duplicate.  If there is
a duplicate then retry.  Typically this would be done in your backend code and is a bit of an ugly solution.

It would be something like this in psuedo code:
```
function insert_row_with_slug(data) {
    slug = base36(random())
    try insert_into_table(slug: slug, data: data) {
        return OK
    } catch DUPLICATE_KEY_EXCEPTION {
        insert_slug(data)
    }
}
```

    

## Better solution

The nicest solution would be if Postgres could generate a unique the slug for us automatically when we insert a new row, and we could have a
table definition something like this:

```sql
    CREATE TABLE shortened_urls (
          id integer,
          slug varchar(10) DEFAULT generate_slug(),
          original_url varchar(255)
          ...
      );
```

## How can we do this?

We can generate a incremental number (a sequence in Postgres) and then encrypt it with a symmetric encryption algorithm.  We'll never need
to decrypt the result, but by encrypting it symmetrically we ensure that it is unique, secure and not easily guessable.

### A Feistel cypher

A simple, cheap, symmetric encryption function is a [Feisel cypher](https://en.wikipedia.org/wiki/Feistel_cipher).  The function below,
which is taken from the [PostgreSQL wiki](https://wiki.postgresql.org/wiki/Pseudo_encrypt), does this:

```sql
CREATE FUNCTION pseudo_encrypt(value int) returns int AS $$
DECLARE
l1 int;
l2 int;
r1 int;
r2 int;
i int:=0;
BEGIN
l1:= (value >> 16) & 65535;
r1:= value & 65535;
WHILE i < 3 LOOP
    l2 := r1;
    r2 := l1 # ((((1366 * r1 + 150889) % 714025) / 714025.0) * 32767)::int;
    l1 := l2;
    r1 := r2;
    i := i + 1;
END LOOP;
return ((r1 << 16) + l1);
END;
$$ LANGUAGE plpgsql strict immutable;
```

This will take an integer and return another one.  The input integer would be a sequence number that gets automatically incremented 
for each slug generated.

I suggest changing the contstant `150889` to some other number of a similar size as so your sequence of numbers are different.  

I wasn't sure if any number can be used here, so when I changed it, I then ran the function on all values from 1 to 10 million 
and verified that no duplicate swere created, which only took a couple of minutes.  For my use case I will probably never even reach
100k slugs, so 10m is far in excess of my requirements.

I recommend that you do the same up to the maximum input number you can envisage using.

### Base-36 encoding

We then need to pass the result to a base-36 encode function:

```sql
CREATE FUNCTION to_base36(num int) RETURNS text AS $$
DECLARE
base36_chars text := '0123456789abcdefghijklmnopqrstuvwxyz';
result text := '';
remainder int;
BEGIN
IF num = 0 THEN
    RETURN '0';
END IF;

WHILE num > 0 LOOP
    remainder := num % 36;
    result := substring(base36_chars from remainder + 1 for 1) || result;
    num := num / 36;
END LOOP;

RETURN result;
END;
$$ LANGUAGE plpgsql STRICT IMMUTABLE;
```

### Add some more randomness

To make the slug a bit longer we can append some random characters also, which will ensure that if a someone can determine the salt constants
used in the cypher, they still cannot generate other values in the sequence.

This function will generate a given number of base-36 random characters:
```sql
CREATE FUNCTION random_chars(char_count int, salt int default 0) RETURNS text AS $$
DECLARE
base int:= 36;
shift int := base^(char_count-1);
modulus int := base^(char_count) - shift;
rand bigint := random() * 2^48;

BEGIN
RETURN to_base36(((rand + salt) % modulus + shift)::int);
END;
$$ LANGUAGE plpgsql STRICT IMMUTABLE;
```

#### Why the `salt` parameter here?

Suppose you have a table with an `id` and a `code` column:

`select * from my_table;`

| id | code |
|----|------|
|  1 | ``null`` |
|  2 | ``null`` |
|  3 | ``null`` |

If you then run this:

`update my_table set code = random_chars(4);`

then the `random()` function inside `random_chars()` will only be run once, not for each row,
so the result will look like this:

| id | code |
|----|------|
|  1 |  ``r168`` |
|  2 |  ``r168`` |
|  3 |  ``r168`` |

However, by passing setting the `salt` to the `id` then the output will be different for each row:

`update my_table set code = random_chars(4, id);`

| id | code |
|----|------|
|  1 | ``mef0`` |
|  2 | ``hyng`` |
|  3 | ``8uuo`` |


### Generate the slug
Now we can use these other functions to create the `generate_slug` function:

```sql
CREATE FUNCTION generate_slug(sequence_no int, random_seed int default 0) RETURNS text AS $$
DECLARE
deterministic_part text := to_base36(pseudo_encrypt(sequence_no));
random_part text := random_chars(10 - length(deterministic_part), random_seed);
BEGIN
RETURN deterministic_part || random_part;
END;
$$ LANGUAGE plpgsql STRICT IMMUTABLE;
```

This uses the Feistel cypher and the input value to generate the first part of the slug, which on its own is guaranteed to be collission-free.
It then appends random characters (typically 4 or 5) to pad the length to 10 characters.


### How do we use this?

To use the `generate_slug` function we need to first generate a Postgres sequence for our slug, which is a integer that will increase each time it is used:

```sql
CREATE SEQUENCE slug_sequence START 1;
```

Now can can create the table which needs the slugs, for example:

```sql
CREATE TABLE shortened_urls (
    id integer primary key generated always as identity,
    slug varchar(10) DEFAULT generate_slug(nextval('slug_sequence')::int),
    original_url varchar(255)
);
```

Now when we insert a new row, our slug is automatically generated:

```sql
insert into shortened_urls (original_url) values ('https://foo.com/some-long-path');
insert into shortened_urls (original_url) values ('https://bar.com/some-even-longer-path');
```

Checking the results:

```sql
select * from shortened_urls;
```

|id|slug|original_url|
|--|----|------------|
|1|``fwugnmhm7o``|``https://foo.com/some-long-path``|
|2|``wu4dhepl2n``|``https://bar.com/some-even-longer-path``|


Now we can write our app that allows users to access URLs with a short slug, e.g. 

``https://my-url-shortner/fwugnmhm7o``

## Github repo

All the functions can be found in `functions.sql` file in [chazsconi/postgres_slug_functions](https://github.com/chazsconi/postgres_slug_functions)