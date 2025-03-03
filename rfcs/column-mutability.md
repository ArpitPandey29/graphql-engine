# Column Mutability

## Metadata

```
---
authors: Philip Lykke Carlsen <philip@hasura.io>
discussion:
   https://github.com/hasura/graphql-engine-mono/issues/2407
   https://github.com/hasura/graphql-engine-mono/pull/2507
---
```

## Description

Various features of databases influence the set of columns that should be
exposed under different circumstances. Examples of this include Identity Columns
and (DB) Generated Columns [^1].
[^1]: https://www.postgresql.org/docs/14/sql-createtable.html

This RFC proposes to add the basic, shared concept of _Column Mutability_ with
the goal of simplifying the implementation of features including those mentioned
above and similar, as well as improving code reuse, specifically for Schema
Generation.

### Problem

In order to motivate this proposal, we'll start with a historical summary a
previous effort of implementing support for handling identity columns.

Identity Columns were introduced as first class concept, present throughout the
different parts of the implementation that needed to adapt its behavior
depending on them.

What we didn't know at the time was that this would turn out to be awkward,
because identity columns in Postgresql and MSSQL have different semantics. And
because Postgres even has two variations of identity columns which need to be
treated differently. (More on this later)

The specific way in which this turned awkward was in the schema code. We want to
avoid duplicating this code, so that we produce consistent schemas across
backends. However, in order for this code to deal correctly with different
behaviors of different identity column variants it had to accept further
backend-specific customization options (or, hypothetically, be refactored).

Concretely, apart from case-switching on whether columns were identity columns
or not the code was amended with the (`class Backend b` type-level) feature flag
`type XOnConflict b`, indicating whether a backend would support
`_on_conflict`-based upserts. [^2]

[^2]: MSSQL would not support upserts, as a concession for us missing an
  implementation that would distinguish between what columns to insert and what
  columns to update owning to `_on_conflict` being part of `insert_` mutations.

Then came [issue #7557](https://github.com/hasura/graphql-engine/issues/7557),
and we realised the importance of distinguishing between `GENERATED BY DEFAULT AS
IDENTITY` and `GENERATED ALWAYS AS IDENTITY` identity columns in Postgres. [^3]

[^3]: `BY DEFAULT` may be inserted but not updated, while `ALWAYS` may neither.
  So now that the PG backend had a concept of id cols it would interpret them the
  same way as in MSSQL (which are more similar to the PG `ALWAYS` kind), meaning
  one user's insert mutations on a table with a `GENERATED BY DEFAULT` identity
  col suddenly stopped working, because the id col was elided from the schema!

To summarize, it seems our approach to modelling identity columns was
inappropriate: We introduced a single _Identity Column_ modelling concept which
had to cover for the various idiosyncratic variants between (and within!)
databases, and as a result the shared schema generation code needed to know
about different backends' idiosyncracies.

### Why is it important?

At the level of concrete features our product should support, Identity columns
are already something our customers rely on (as evidenced by issue #7557). On Postgresql,
Identity Columns are even the recommended successor to `SERIAL` columns [^4].

[^4]: [Don't use serial](https://wiki.postgresql.org/wiki/Don%27t_Do_This#Don.27t_use_serial):
     > For new applications, identity columns should be used instead.
     >
     > Why not?
     > 
     > The serial types have some weird behaviors that make schema, dependency,
     > and permission management unnecessarily cumbersome.
   
Additionally, a generic type of generated columns (database backed rather than
GraphQL Engine backed) could also be an attractive feature to support, which
should be easy to build on top of this proposal.

On a more meta-feature level, faithfully exposing relational databases on
GraphQL is our bread and butter. Being able to develop new features confidently
with less effort is an important enabler of this.

Our ability to achieve this (and thus the quality of the finished product)
is affected by our choices of the abstractions that we use to express our solution.
Therefore, it's important we pick those that enable us rather than hamper us.

### Success

This effort is a success once insert, update, and upsert mutations are implemented
using a backend-agnostic notion of column mutability, unaffected by backend-specific
concepts such as Identity Columns and Generated Columns.

## How

This specific proposal describes an implementation strategy and does not itself
have any user visible aspects. 

Implementing this proposal in a basic form amounts to three conceptually simple
things:

1. Amend the table metadata with column mutability information. E.g. by changing
   `Hasura.RQL.Types.Column.ColumnInfo` or
   `Hasura.RQL.Types.Table.TableCoreInfoG`. We should be able to record whether
   each column is insertable and updatable at the least.
2. Updating the schema generation code to respect this new metadata. Points of
   interest include update mutations, insert mutations, and insert with
   `on_conflict` (the `update_columns` field should only contain updatable
   columns).
3. Updating the permissions code to respect this new metadata. Column Update
   Permissions should apply to only updatable columns etc.

In the actual table metadata collection code we may initially just mark every
column as insertable and updatable, unless the work is in the context of
e.g. implementing identity columns.

### Effects and Interactions

This feature may be perfectly well confined to the server. It might be
interesting however to show or use the column mutability information in the
Console, for example to reflect to a user how HGE is understanding their
database schema.

### Alternatives

If you do not have a generic concept of column mutability, all features that
influence the set of columns exposed in certain situations will have to
themselves deal directly with those situations.

Examples of code in an implementation without generic column mutability:
* Schema generation needs to know and understand the concepts of Identity
  Columns, and Generated Columns. These work differently across backends, which
  hampers code reuse.
* Permissions need to know about Identity Columns, and Generated Columns,
  because Update Permissions don't make sense those columns. But only sometimes,
  and that in backend specific ways.

### Costs and Drawbacks

A drawback of using this approach over the alternatives is that, as a feature
implementer you need to know prior knowledge of the column mutability feature in
order to exploit it. Nothing prohibits you from implementing some feature in
parallel, that could have benefited from using it.

But then again, if not exploiting the column mutability feature, you would have
to know about all the relevant interactions of column mutability in order to
produce a well-rounded feature.

Another drawback is that this could potentially result in poor usability if it
turns out that some features need to be handled more specifically than just
including or excluding columns depending on their mutability affords.

A somewhat mild but illustrative example of this is a user wanting to update a
non-updatable column and being puzzled as to why this column is not present as a
field in the update mutation.

This concrete example could be remedied by annotating the metadata that
indicates mutability with a human-readable text describing the cause, which we
could then display in the console.

But care must be taken: In the (contrived) case that we wanted to, say, give
rich error messages in response only to update mutations that mention identity
columns but not those that mention generated columns, we would have encountered
a need that is not met by a generic column mutability concept. That would leave
us having to overload/pollute column mutability with data and logic for dealing
with non-universal edge cases, and we wouldn't have achieved anything.

### Unresolved Questions

> A place for authors to list out open questions
> Ideally, it would serve as a collection point for others to chime in during
> the review process to either identify their own questions or offer potential
> solutions

### Future Work / Out of Scope

As stated earlier, this RFC does not directly imply any application of its
ideas. Applications such as Identity Columns are treated separately.

Here is a first attempt at looking more closely at how Identity Columns would
use Column Mutability to fulfill its needs:

* In MSSQL.

  We have a couple of options for how we want to identity columns to work.

  ```
  IDENTITY(..), with SET IDENTITY_INSERT => not updatable, insertable
  ``` 
  (This variant will also require specific handling at SQL translation)

  ```
  IDENTITY(..), w/o  SET_IDENTITY_INSERT => not updatable, not insertable
  ```

* In Postgres

  Similarly to MSSQL, we need to choose how to handle one flavor of identity column:

  ```
  GENERATED ALWAYS AS IDENTITY, with OVERRIDING SYSTEM VALUE => not updatable, insertable
  ```
  (This variant will also require specific handling at SQL translation)

  ```
  GENERATED ALWAYS AS IDENTITY, w/o  OVERRIDING SYSTEM VALUE => not updatable, not insertable
  ```

  The third variant does not even register:
  ```
  GENERATED BY DEFAULT AS IDENTITY => updatable, insertable
  ```

* General Computed Columns, as supported by all of MSSQL, PG 14, and MySQL:
  `generated => not updatable, not insertable` 
  
Note: Careful inspection of the above suggests that the
upsert-via-insert+on_conflict schema should work for both MSSQL and Postgresql,
regardless of the flavors we choose to support. The only troublesome combination
is `updatable, not insertable` which is absent above. [^why-troublesome]

[^why-troublesome]: The reason `updatable, not insertable` is troublesome is that the values to be upserted are given as:
      ```
      insert_some_table(
        objects: [{ <values here> }],
          on_conflict: { update_columns: [<columns to be updated>], ...  }
      )
      ```
      We want to elide fields `not insertable` from the schema of `<values here>`,
      but that is also the place where values upserted go.

Also, for something completely different, it might be interesting to make column
mutability something a user can specify. This use case does however have some
overlap with the Columns Permissions feature.
