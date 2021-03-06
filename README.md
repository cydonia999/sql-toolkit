# [SQL Toolkit](https://github.com/craigmichaelmartin/sql-toolkit) &middot; [![Build Status](https://travis-ci.org/craigmichaelmartin/sql-toolkit.svg?branch=master)](https://travis-ci.org/craigmichaelmartin/sql-toolkit) [![Greenkeeper badge](https://badges.greenkeeper.io/craigmichaelmartin/sql-toolkit.svg)](https://greenkeeper.io/) [![codecov](https://codecov.io/gh/craigmichaelmartin/sql-toolkit/branch/master/graph/badge.svg)](https://codecov.io/gh/craigmichaelmartin/sql-toolkit)

## Installation

```bash
npm install --save sql-toolkit
```

## What is SQL Toolkit?

SQL Toolkit is a node library built for `pg-promise` which allows you to write regular native SQL and receive back properly structured (nested) pure business objects.

#### Philosophy

- Have _pure_ "business objects" which can represent the data of a table and be the subject of the app's business logic.
  - These objects are pure javascript objects which describe the shape of data for table, but are not connected to the database.
  - They will be full of userland business logic methods, and their purity will allow them to be easy to test/use.
- Have _database-connected_ "data access objects" which allow the unobstructed writing of normal SQL.
  - This is not an ORM. There are not hundreds of methods mapping the complexity, expressiveness, and nuance of SQL to class objects.
  - Rather, a data access layer in which native SQL (not ORM-abstracted SQL-ish) is written, and which understands pure business objects as inputs, and returns them property nested and structured as output.

#### Concepts

A **Business Object** (BO) is a pure javascript object corresponding to a table.

A **Data Access Object** (DAO) is a database-aware abstraction layer where SQL is written.

DAO methods return pure BOs. Likewise, BOs may be passed to DAO methods to get or update records.

---

## Examples

### Data Access Object

Our data access layer where SQL is written.

```javascript
class Person extends BaseDAO {
  // ...
}
```

Lets start with a basic example which just uses the
**`BaseBO.createOneFromDatabase`** method to map the column names to our desired
javascript properties.

```javascript
getRandom() {
  const query = `
    SELECT person.id, person.first_name, person.last_name, person.created_date, person.employer_id
    FROM person
    ORDER BY random()
    LIMIT 1;
  `;
  return this.db
    .one(query)
    .then(result => Right(Person.createOneFromDatabase(result)))
    .catch(err => {
      if (!err.name === 'QueryResultError') {
        logError(err);
      }
      return Left(err);
    });
}
// OUTPUT: Person {id, firstName, lastName, createdDate, employerId}
```

Creating our business object and catching the error is tedious. Instead,
lets use `BaseDAO.one`.

```diff
getRandom() {
  const query = `
    SELECT person.id, person.first_name, person.last_name, person.created_date, person.employer_id
    FROM person
    ORDER BY random()
    LIMIT 1;
  `;
+ return this.one(query);
- return this.db
-   .one(query)
-   .then(result => Right(Person.createOneFromDatabase(result)))
-   .catch(err => {
-     if (!err.name === 'QueryResultError') {
-       logError(err);
-     }
-     return Left(err);
-   });
}
// OUTPUT: Person {id, firstName, lastName, createdDate, employerId}
```

Also, specifying all the columns is tedious; lets use
**`BaseBo.getSQLSelectClause()`** to get them for free.

```diff
getRandom() {
  const query = `
-   SELECT person.id, person.first_name, person.last_name, person.created_date, person.employer_id
+   SELECT ${Person.getSQLSelectClause()}
    FROM person
    ORDER BY random()
    LIMIT 1;
  `;
  return this.one(query);
}
// OUTPUT: Person {id, firstName, lastName, createdDate, employerId}
```

More important than saving the tedium, though, is how
**`BaseBo.getSQLSelectClause()`** namespaces each select expression name
under the hood, and which **`BaseBo.createOneFromDatabase`** knows how to handle.
This means that when joining, not only is the select expression easy,
select expression names won't collide:

```diff
getRandom() {
  const query = `
-   SELECT ${Person.getSQLSelectClause()}
+   SELECT ${Person.getSQLSelectClause()}, ${Employer.getSQLSelectClause()}
    FROM person
+   JOIN employer on person.employer_id = employer.id
    ORDER BY random()
    LIMIT 1;
  `;
  return this.one(query);
}
// OUTPUT: Person {id, firstName, lastName, createdDate, employer: Employer}
```

Rather than being flat, with the employer id and createdDate colliding with
person's id and createDate, the result is a nice Person BO with a nested
Employer BO.

Lets move to a different example to show off another aspect of
**`BaseBo.createOneFromDatabase`**: how it handles flattening data. Lets say
there are three tags for article being retrieved, rather than the data
being an array of 3 results with article repeated, the result is
a nice Article BO with the tags nested in it.

```javascript
getBySlug(slug) {
  const query = `
    SELECT
      ${Article.getSQLSelectClause()},
      ${Person.getSQLSelectClause()},
      ${ArticleTag.getSQLSelectClause()},
      ${Tag.getSQLSelectClause()}
    FROM article
    JOIN person
        ON article.author_id = person.id
    LEFT JOIN article_tags
        ON article.id = article_tags.article_id
    LEFT JOIN tag
        ON article_tags.tag_id = tag.id
    WHERE article.slug = $(slug);
  `;
  return this.one(query, { slug });
}
// OUTPUT: Article {person: Person, tags: Tags[Tag, Tag, Tag]}
```

Notice that we're using `this.one`, which is what we want. The DAO methods for `one`, `oneOrNone`, `many`, `any` ensure their count against the number of generated top level business objects - not the number of rows the sql expression returns!

Lets say we want to get more than one article. We can make slug an array, and
**`BaseBo.createFromDatabase`** handles it seemlessly, giving us an Articles
collections

```diff
-getBySlug(slug) {
+getBySlugs(slugs) {
  const query = `
    SELECT
      ${Article.getSQLSelectClause()},
      ${Person.getSQLSelectClause()},
      ${ArticleTag.getSQLSelectClause()},
      ${Tag.getSQLSelectClause()}
    FROM article
    JOIN person
        ON article.author_id = person.id
    LEFT JOIN article_tags
        ON article.id = article_tags.article_id
    LEFT JOIN tag
        ON article_tags.tag_id = tag.id
-   WHERE article.slug = $(slug);
+   WHERE article.slug in ($(slugs:csv));
  `;
- return this.many(query, { slugs });
+ return this.many(query, { slugs });
}
-// OUTPUT: Article {person: Person, tags: Tags[Tag, Tag, Tag]}
+// OUTPUT: Articles[
+//  Article {person: Person, tags: Tags[Tag, Tag, Tag]}
+//  Article {person: Person, tags: Tags[Tag, Tag]}
+// ]
```

Lastly, lets switch gears one more time to see how meta data can be intertwined:

```javascript
getBloggerPayout(id, startDate, endDate) {
  const query = `
    SELECT
      person.id,
      person.slug,
      person.email,
      person.first_name,
      person.last_name,
      person.last_paid_date,
      person.pay_frequency,
      COALESCE(SUM(article.blogger_payout), 0) as meta_amount,
    FROM
      person
    LEFT JOIN article
      ON article.author_id = person.id
      AND (article.created_date BETWEEN $(startDate) AND $(endDate))
    WHERE
      person.id = $(id)
    GROUP BY person.id, person.slug, person.email,
      person.first_name, person.last_name, person.last_paid_date,
      person.pay_frequency
    ORDER BY meta_amount DESC NULLS LAST;
  `;
  return this.one(query, {id, startDate, endDate });
}
```

### Business Object Usage

Now lets look at our business logic layer where we use the DAO to get/persist pure data.

```javascript
let raw = new Person({
  email: 'foobar@gmail.com',
  firstName: 'craig',
  lastName: 'martin'
});

const personDAO = new PersonDAO({ db });

// Returns a person business object with the persisted data
let person = await personDAO.create(raw);

person.email = 'craigmartin@gmail.com';

// Returns a person business object with the updated persisted data
person = await personDAO.update(person);

// Gets or creates a person business object
same = await personDAO.getOrCreate(raw);
same.id === person.id; // true

// Returns the person business object which matches this data
same = await personDAO.getMatching(
  new Person({ email: 'craigmartin11@gmail.com' })
);
same.id === person.id; // true

// Deletes the person data form the database
await personDAO.delete(person);

// Returns the person business object returned by whatever DAO methods your write
person = await personDAO.getFromCustomMadeMethod();
```

To see everything in action, check out [the examples directory](https://github.com/craigmichaelmartin/sql-toolkit/tree/master/examples) and the [tests](https://github.com/craigmichaelmartin/sql-toolkit/blob/master/src/bo/base-bo.spec.js).

---

## Comparisons

Low Level Abstractions

- **Database Drivers** (eg [node-postgres](https://github.com/brianc/node-postgres), [mysql](https://github.com/mysqljs/mysql), [node-sqlite3](https://github.com/mapbox/node-sqlite3)) - These are powerful low level libraries that handle connecting to a database, executing raw SQL, and returning raw rows. All the higher level abstractions are built on these. While they can be used directly, `sql-toolkit` aims at providing structure fo the returned raw rows by creating pure, nested business objects for the result rows.

Middle Level Abstractions

- **Query Builders** (eg [knex](https://github.com/tgriesser/knex)) - query builder libraries (built on database drivers) aim at the _writing of raw SQL_: seeking to solve the problem of composing sql queries in a dialetic-generic api. `sql-toolkit` takes the approach that the tradeoff of developers having to learn the huge surface area of dialetic-generic api, and having to map the complexity and nuance of SQL to it, are simply not worth the cost, and so does not use a query building library. With `sql-toolkit` you just write SQL. The tradeoff on `sql-toolkits` side that is indeed being tied to a sql dialect and in the inability to compose sql expressions (strings don't compose nicely). Yet all this considered, `sql-toolkit` sees writing straight SQL heaviliy as a feature, not a defect needing solved.
- **Result Resolvers** (eg `sql-toolkit`) - result resolvers librarys (built on database drivers) aim at the _receiving of raw result rows_: seeking to provided pure, properly nested business objects (with custom methods and dynamic properties) in place of raw, flat result rows. To my knowledge, `sql-toolkit` is the only player in this space.

High Level Abstractions

- **ORMs** (eg [sequelize](https://github.com/sequelize/sequelize), [waterline](https://github.com/balderdashy/waterline), [bookshelf](https://github.com/bookshelf/bookshelf), [typeorm](https://github.com/typeorm/typeorm)) - orms (built on database drivers as well as sometimes standalone query builders, and with some result resolving baked in) aim at providing it all. `sql-toolkit` is more than just the preference against the query builder portion of ORMs - it also resolves the result rows to _pure_ business objects, not stateful, db-connected objects. This purity in business objects fosters a clean layer of the business layer from the data access layer.

---

## API

### Classes

#### `BaseBo`

An abstract class which is the base class your BO classes to extend.

**Abstract Methods** to be implemented

- `get BoCollection(): BoCollection` - Returns the business object collection class constructor.
- `static get tableName(): string` - Returns the string table name which the business object associates with from the database.
- `static get sqlColumnsData(): Array<string|ColumnData>` - Returns an array of the database column data. The type is either:
  - `ColumnData {column, property?, references?, primaryKey?, transform?}`
    - `column: string` - The sql column name
    - `propery: string` - The javascript property name for this column (defaults to camelCase of `column`)
    - `references: Bo` - The relationship to another Bo (defaults to null)
    - `primaryKey: boolean` - Is this column (part of) the primary key (defaults to false)
    - `transform: fn` - When this data is pulled, a transform that runs on it; eg, creating a momentjs object for dates (defaults to `() => {}`)
  - `string` - If a string, it is applied as the `column` value, with all others defaulted.
  - (Note: if there is no primary key, `id` is defaulted)

Optional

- `static get displayName(): string` - Returns the string display name of the business object (defaults to camelcase of tableName)

**Public Methods**

- `constructor(props: object)`
- `static primaryKey()`
- `static get columns()`
- `static get sqlColumns()`
- `static get references()`
- `static get displayName()`
- `static getPrefixedColumnNames()`
- `static getSQLSelectClause()`
- `static objectifyDatabaseResult(result)`
- `static mapToBos(objectified)`
- `static clumpIntoGroups(processed)`
- `static nestClump(clump)`
- `static createFromDatabase(result)`
- `static createOneFromDatabase(result)`
- `getSqlInsertParts()`
- `getSqlUpdateParts()`
- `getMatchingParts()`
- `getMatchingPartsObject()`
- `getNewWith(sqlColumns, values)`
- `getValueBySqlColumn(sqlColumn)`

#### `BaseBoCollection`

An abstract class which is the base class your Bo Collection classes extend.

**Abstract Methods** to be implemented

- `static get Bo(): BO` - Returns the individual (singular) business object class constructor.

Optional

- `get displayName(): BO` - Returns the string display name of the business object collection (defaults to bo displayName with an "s")

**Public Methods**

- `constructor(props: object)`
- `static get displayName()`

#### `BaseDAO`

The base class your DAO classes extend.

**Abstract Methods** to be implemented

- `get Bo(): BO` - Returns the business object class constructor.
- `get BoCollection(): BO` - Returns the collection business object class constructor.

**Public Methods**

- `constructor({ db }})`

Abstractions over `pg-promise`'s query methods:

- `one(query: string, params: object)` - executes a query and returns a Bo, or throws.
- `oneOrNone(query: string, params: object)` - executes a query and returns a Bo or undefined, or throws.
- `many(query: string, params: object)` - executes a query and returns a BoCollection with at least one model, or throws.
- `any(query: string, params: object)` - executes a query and returns a BoCollection.

(Note, these methods assert the correct number on the created BO's - not the raw postgres sql result. Thus, for example, `one` understands that there may be multiple result rows (which pg-promise's `one` would throw at) but which could correctly nest into one BO.)

Helper functions if usint `pr-promise`'s query methods directly:

- `errorHandler(err)` - helper function if using pg-promise `db` directly

Built-in "basic" / generic functions which your extending DAO class instance gets for free

- `getMatching(bo: BaseBO)`
- `getAllMatching(bo: BaseBO)`
- `getOrCreate(bo: BaseBO)`
- `create(bo: BaseBO)`
- `update(bo: BaseBO)`
- `delete(bo: BaseBO)`

Lets take a few examples to show this.

### Methods

#### `createBaseBO({ getBusinessObjects }): BaseBo`

**Parameters**

- `getBusinessObjects: () => Array<BusinessObject>` - A function which returns an array of all the business objects, used to construct joined row data in the business object.

**Return Value**

- The BaseBo class to extend for your business objects.

#### `createBaseDAO({ db, logError }): BaseDAO`

**Parameters**

- `logError: function`
- `db: pg-promise database`

**Return Value**

- The BaseDAO class to extend for your business objects.

---

## Current Status

#### Current Limitations (PRs welcome!)

- `pg-promise`/`node-postgres` is the only database driver supported. There is not technical reason for this, other than that the project I'm using has a postgres database and so I only had `node-postgres` in mind. It would be great if `sql-toolkit` was database driver agnostic.
- the dao you are writing your sql in must always be in the "select" and must be the one you want as your root(s) return objects
  - the query can start from some other table, and join a bunch of times to get there, though
- there must be a clear path in the "select" to your leaf joined-to-entities (eg, (Good): Article, ArticleTag, Tag, TagModerator, Moderator; not (Bad): Article, Moderator).
- the result of _the select_ must always be a non-circular tree (eg, (Bad): Article, Person, Group, GroupArticle, Article)

#### Current Todos (PRs welcome!):

- Performance. While the API has been somewhat thought through and iterated on to this point, the implementation details have been secondary, knowing that they can be perfected in time. Probably about time now.
- Add more tests
- Known Bug: if a table references the same table twice, the first one is found as the nodePointingToIt and so ends up throwing.
  - ideally the fix to this will change the behavior of when a table points to another table by another name (author_id -> person)
- Think about how to handle the none case of oneOrNone, any, and none

#### Is it production ready?

It is in production at [www.kujo.com](www.kujo.com) - powering the marketing pages and blog, as well as the customer, affiliate, and admin platforms (behind login). When considering for your case, note the Current Limitations and TODOs sections above.
