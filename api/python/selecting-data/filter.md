---
layout: api-command
language: Python
permalink: api/python/filter/
command: filter
related_commands:
    get: get/
    get_all: get_all/
    between: between/
---

# Command syntax #

{% apibody %}
selection.filter(predicate[, default=False]) &rarr; selection
stream.filter(predicate[, default=False]) &rarr; stream
array.filter(predicate[, default=False]) &rarr; array
{% endapibody %}

# Description #

Return all the elements in a sequence for which the given predicate is true. The return value of `filter` will be the same as the input (sequence, stream, or array). Documents can be filtered in a variety of ways&mdash;ranges, nested values, boolean conditions, and the results of anonymous functions.

By default, `filter` will silently skip documents with missing fields: if the predicate tries to access a field that doesn't exist (for instance, the predicate `{'age': 30}` applied to a document with no `age` field), that document will not be returned in the result set, and no error will be generated. This behavior can be changed with the `default` optional argument.

* If `default` is set to `True`, documents with missing fields will be returned rather than skipped.
* If `default` is set to `r.error()`, an `RqlRuntimeError` will be thrown when a document with a missing field is tested.
* If `default` is set to `False` (the default), documents with missing fields will be skipped.

{% infobox %}
__Note:__ `filter` does not use secondary indexes. For retrieving documents via secondary indexes, consider [get_all](/api/python/get_all/), [between](/api/python/between/) and [eq_join](/api/python/eq_join/).
{% endinfobox %}

## Basic predicates ##

__Example:__ Get all users who are 30 years old.


```py
r.table('users').filter({'age': 30}).run(conn)
```

The predicate `{'age': 30}` selects documents in the `users` table with an `age` field whose value is `30`. Documents with an `age` field set to any other value *or* with no `age` field present are skipped.

While the `{'field': value}` style of predicate is useful for exact matches, a more general way to write a predicate is to use the [row](/api/python/row) command with a comparison operator such as [eq](/api/python/eq) (`==`) or [gt](/api/python/gt) (`>`), or to use a lambda function that returns `True` or `False`.

```py
r.table('users').filter(r.row["age"] == 30).run(conn)
```

In this case, the predicate `r.row["age"] == 30` returns `True` if the field `age` is equal to 30. You can write this predicate as a lambda function instead:

```py
r.table('users').filter(lambda user:
    user["age"] == 30
).run(conn)
```

Predicates to `filter` are evaluated on the server, and must use ReQL expressions. Some Python comparison operators are overloaded by the RethinkDB driver and will be translated to ReQL, such as `==`, `<`/`>` and `|`/`&` (note the single character form, rather than `||`/`&&`).

Also, predicates must evaluate document fields. They cannot evaluate [secondary indexes](/docs/secondary-indexes/).

__Example:__ Get all users who are more than 18 years old.

```py
r.table("users").filter(r.row["age"] > 18).run(conn)
```

__Example:__ Get all users who are less than 18 years old and more than 13 years old.

```py
r.table("users").filter((r.row["age"] < 18) & (r.row["age"] > 13)).run(conn)
```

__Example:__ Get all users who are more than 18 years old or have their parental consent.

```py
r.table("users").filter(
    (r.row["age"] >= 18) | (r.row["hasParentalConsent"])).run(conn)
```

## More complex predicates ##

__Example:__ Retrieve all users who subscribed between January 1st, 2012
(included) and January 1st, 2013 (excluded).

```py
r.table("users").filter(
    lambda user: user["subscription_date"].during(
        r.time(2012, 1, 1, 'Z'), r.time(2013, 1, 1, 'Z'))
).run(conn)
```

__Example:__ Retrieve all users who have a gmail account (whose field `email` ends with `@gmail.com`).

```py
r.table("users").filter(
    lambda user: user["email"].match("@gmail.com$")
).run(conn)
```

__Example:__ Filter based on the presence of a value in an array.

Given this schema for the `users` table:

```py
{
    "name": <type 'str'>
    "places_visited": [<type 'str'>]
}
```

Retrieve all users whose field `places_visited` contains `France`.

```py
r.table("users").filter(lambda user:
    user["places_visited"].contains("France")
).run(conn)
```

__Example:__ Filter based on nested fields.

Given this schema for the `users` table:

```py
{
    "id": <type 'str'>
    "name": {
        "first": <type 'str'>,
        "middle": <type 'str'>,
        "last": <type 'str'>
    }
}
```

Retrieve all users named "William Adama" (first name "William", last name
"Adama"), with any middle name.


```py
r.table("users").filter({
    "name": {
        "first": "William",
        "last": "Adama"
    }
}).run(conn)
```

If you want an exact match for a field that is an object, you will have to use `r.literal`.

Retrieve all users named "William Adama" (first name "William", last name
"Adama"), and who do not have a middle name.

```py
r.table("users").filter(r.literal({
    "name": {
        "first": "William",
        "last": "Adama"
    }
})).run(conn)
```

You may rewrite these with lambda functions.

```py
r.table("users").filter(
    lambda user:
    (user["name"]["first"] == "William")
        & (user["name"]["last"] == "Adama")
).run(conn)
```

```py
r.table("users").filter(lambda user:
    user["name"] == {
        "first": "William",
        "last": "Adama"
    }
).run(conn)
```

## Handling missing fields ##

By default, documents missing fields tested by the `filter` predicate are skipped. In the previous examples, users without an `age` field are not returned. By passing the optional `default` argument to `filter`, you can change this behavior.

__Example:__ Get all users less than 18 years old or whose `age` field is missing.

```py
r.table("users").filter(r.row["age"] < 18, default=True).run(conn)
```

__Example:__ Get all users more than 18 years old. Throw an error if a
document is missing the field `age`.

```py
r.table("users").filter(r.row["age"] > 18, default=r.error()).run(conn)
```

__Example:__ Get all users who have given their phone number (all the documents whose field `phone_number` exists and is not `None`).

```py
r.table('users').filter(
    lambda user: user.has_fields('phone_number')
).run(conn)
```

__Example:__ Get all users with an "editor" role or an "admin" privilege.

```py
r.table('users').filter(
    lambda user: (user['role'] == 'editor').default(False) |
        (user['privilege'] == 'admin').default(False)
).run(conn)
```

Instead of using the `default` optional argument to `filter`, we have to use default values on the fields within the `or` clause. Why? If the field on the left side of the `or` clause is missing from a document&mdash;in this case, if the user doesn't have a `role` field&mdash;the predicate will generate an error, and will return `False` (or the value the `default` argument is set to) without evaluating the right side of the `or`. By using `.default(False)` on the fields, each side of the `or` will evaluate to either the field's value or `False` if the field doesn't exist.
