# Ecto.Function

[![Hex.pm](https://img.shields.io/hexpm/dt/ecto_function.svg)](https://hex.pm/packages/ecto_function)
[![Travis](https://img.shields.io/travis/hauleth/ecto_function.svg)](https://travis-ci.org/hauleth/ecto_function)

A helper macro for defining macros which simplify calling DB functions.

## Installation

The package can be installed by adding `ecto_function` to your list of dependencies in `mix.exs`:

```elixir
def deps do
  [
    {:ecto_function, "~> 1.0.1"}
  ]
end
```

The docs can be found at <https://hexdocs.pm/ecto_function>.

## Usage

When you use a lot of DB functions inside your queries then it probably looks
like this:

```elixir
from item in "items",
  where: fragment("date_trunc(?, ?)", "hour", item.inserted_at) < fragment("date_trunc(?, ?)", "hour", fragment("now()")),
  select: %{regr: fragment("regr_sxy(?, ?)", item.y, item.x)}
```

There are a lot of `fragment` calls which makes this code quite challenging to read.
However, there is way out: you can write macros:

```elixir
defmodule Foo do
  defmacro date_trunc(part, field) do
    quote do: fragment("date_trunc(?, ?)", ^part, ^field)
  end

  defmacro now do
    quote do: fragment("now()")
  end

  defmacro regr_sxy(y, x) do
    quote do: fragment("regr_sxy(y, x)", ^y, ^x)
  end
end
```

And then cleanup your query to:

```elixir
import Foo
import Ecto.Query

from item in "items",
  where: date_trunc("hour", item.inserted_at) < date_trunc("hour", now()),
  select: %{regr: regr_sxy(item.y, item.x)}
```

However, there is still a lot of repetition in your new fancy helper module. You
need to repeat function name twice, name each argument, insert all that carets
and stuff.

How about little help?

```elixir
defmodule Foo do
  import Ecto.Function

  defqueryfunc date_trunc(part, field)
  defqueryfunc now
  defqueryfunc regr_sxy/2
end
```

Much cleaner…

## Reasoning

[Your DB is powerful](http://modern-sql.com/slides). Really. A lot of
computations can be done there. There is a whole [chapter][chapter] dedicated to
describing all the functions PostgreSQL has to offer, and Ecto only supports
a tiny fraction.

- `sum`
- `avg`
- `min`
- `max`

...to be exact.

Saying that we have "limited choice" would be disrespectful to DB systems like
PostgreSQL or Oracle. Yet, the Ecto core team has walid reasoning for only
supporting these functions: they are likely available in every DB system out
there, so it's a no brainer! However, you as an end-user shouldn't be limited to
such a small set...

Let's be honest, you will probably never change your DB engine and if you do,
you'll likely be faced with a rewrite of your whole system from the ground up.
So this is why this module was created: To allow you to easily opt-in to all
functions available in your SQL DB (it could work with NoSQL DBs but I test only
against PostgreSQL).

For completeness, you can also check [Ecto.OLAP][olap] which provide helpers for
some more complex functionalities like `GROUPING` (and in near future also
window functions).

### But why not introduce this directly to Ecto?

Because there is no need. Personally, I would like to see Ecto split up
a little&mdash;like changesets should be in separate library in my humble
opinion. Also, I believe that such PR would never be merged as "non scope" for
the reasons I gave earlier.

[chapter]: https://www.postgresql.org/docs/current/static/functions.html "Chapter 9. Functions and Operators"
[olap]: https://github.com/hauleth/ecto_olap
