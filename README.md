# Datamodel v1.1

Over the last months, we have worked with the community to define a new [datamodel specification](https://github.com/prisma/prisma/issues/3408) for Prisma. This new version is called datamodel v1.1 and is currently available in an early preview. 

## Table of contents

- [Motivation](#motivation)
  - [More control over database layout](#more-control-over-database-layout)
  - [Consolidated migrations](#consolidated-migrations)
- [Getting started with the datamodel v1.1](#getting-started-with-the-datamodel-v11)
  - [Prerequisites](#prerequisites)
  - [Option 1: From scratch](#option-1-from-scratch)
  - [Option 2: With an existing database](#option-2-with-an-existing-database)
  - [Option 3: Migrating from datamodel v1](#option-3-migrating-from-datamodel-v11)
- [What's new in datamodel v1.1](#whats-new-in-datamodel-v11)
  - [Directives](#directives)
  - [Relations](#relations)
- [Migrations and introspection with the Prisma CLI](#migrations-and-introspection-with-the-prisma-cli)
  - [`prisma deploy`](#prisma-deploy)
  - [`prisma introspect`](#prisma-introspect)
  - [Common workflows](#common-workflows)
- [FAQ](#faq)

## Motivation

The main motivations for the new datamodel are:

- Enabling more control over how Prisma lays out the database schema
- Consolidated migrations (unifying the "active" and "passive" database connectors)

### More control over database layout

The current datamodel is opinionated about the database layout in many ways. Here are a few things that are now possible with datamodel v1.1:

- Model/field names in the datamodel can differ from the names of the underlying tables/columns
- Specify whether a relation should use a JOIN table or inline references
- Use any field as `id` field
- Use any field as `createdAt` or `updatedAt` fields

### Consolidated migrations

With the current datamodel, developers need to decide whether Prisma should perform database migrations for them, by setting the `migrations` flag in [`PRISMA_CONFIG`](https://www.prisma.io/docs/prisma-server/deployment-environments/docker-rty1/#prisma_config-reference) when the Prisma server is deployed. In datamodel v1.1 the `migrations` flag is removed, meaning developers can at all times either migrate the database manually or use Prisma for the migration.

## Getting started with the datamodel v1.1

### Prerequisites

The datamodel v1.1 is available in the [latest beta](https://www.prisma.io/docs/releases-and-maintenance/releases-and-beta-access/installing-the-beta-b5op/) version of Prisma.

#### 1. Install the latest beta of the Prisma CLI

```
npm install -g prisma@beta
```

#### 2. Use the latest beta Prisma Docker image

In the Docker Compose file for your Prisma server, make sure to use the latest beta Docker image:

```yml
prismagraphql/prisma:1.30-beta
```

#### 3. Set the `prototype` flag to `true` in `PRISMA_CONFIG`

Enable the datamodel v1.1 by adding the `prototype` flag in your [`PRISMA_CONFIG`](https://www.prisma.io/docs/prisma-server/deployment-environments/docker-rty1/#prisma_config-reference):

```yml
port: 4466
prototype: true
databases:
  default:
    connector: postgres # or mysql
    host: 127.0.0.1
    port: 5432
    user: prisma
    password: prisma
```

> Note that you might need to adjust the database connection details to match your setup. Check out the [`example`](./example) folder for a ready-to-use setup.

### Option 1: From scratch

#### 1. Create Docker Compose file

Create a new file callled `docker-compose.yml` and add the following code to it:

```yml
version: '3'
services:
  prisma:
    image: prismagraphql/prisma:1.30-beta
    restart: always
    ports:
    - "4466:4466"
    environment:
      PRISMA_CONFIG: |
        port: 4466
        prototype: true
        databases:
          default:
            connector: postgres
            host: postgres
            user: prisma
            password: prisma
            port: 5432
  postgres:
    image: postgres
    restart: always
    ports:
    - "5432:5432"
    environment:
      POSTGRES_USER: prisma
      POSTGRES_PASSWORD: prisma
    volumes:
      - postgres:/var/lib/postgresql/data
volumes:
  postgres:
```

Note that in this example, we're using a PostgreSQL database. We're also exposing the PostgreSQL port `5432` so that you can e.g. connect to it from a database GUI like Postico.

#### 2. Deploy Prisma server

Run the following command to deploy your server:

```
docker-compose up -d
```

#### 3. Setup Prisma project

A Prisma project always requires at least two files: `prisma.yml` and a datamodel file (e.g. called `datamodel.prisma`).

Create the following `prisma.yml`:

```yml
endpoint: http://localhost:4466
datamodel: datamodel.prisma
```

And this `datamodel.prisma`:

```graphql
type User @db(name: "user") {
  id: ID! @id
  createdAt: DateTime! @createdAt
  email: String! @unique
  name: String
  role: Role @default(value: USER)
  posts: [Post!]!
  profile: Profile @relation(link: INLINE)
}

type Profile @db(name: "profile") {
  id: ID! @id
  user: User!
  bio: String!
}

type Post @db(name: "post") {
  id: ID! @id
  createdAt: DateTime! @createdAt
  updatedAt: DateTime! @updatedAt
  author: User!
  published: Boolean! @default(value: false)
  categories: [Category!]! @relation(link: TABLE, name: "PostToCategory")
}

type Category @db(name: "category") {
  id: ID! @id
  name: String!
  posts: [Post!]! @relation(name: "PostToCategory")
}

type PostToCategory @db(name: "post_to_category") @linkTable {
  post: Post
  category: Category
}

enum Role {
  USER
  ADMIN
}
```

Let's understand some important bits of the datamodel:

- Each model is mapped a table that's named after the model but lowercased using the `@db` directive
- There are the following relations:
  - **One-to-one** between `User` and `Profile`
  - **One-to-many** between `User` and `Post`
  - **Many-to-many** between `Post` and `Category`
- The **one-to-one** relation between `User` and `Profile` is annotated with `@relation(link: INLINE)` on the `User` model. This means `user` records in the database have a reference to a `profile` record if the relation is present (because the `profile` field is not required, the relation might just be `NULL`). An alternative to `INLINE` is `TABLE` in which case Prisma would track the relation via a dedicated relation table.
- The **one-to-many** relation between `User` and `Post` is is tracked inline the relation via the `author` column of the `post` table, i.e. the `@relation(link: INLINE)` directive is inferred on the `author` field of the `Post` model.
- The **many-to-many** relation between `Post` and `Category` is tracked via a dedicated relation table called `PostToCategory`. This relation table is part of the datamodel and annotated with the `@linkTable` directive.
- Each model has an `id` field annotated with the `@id` directive.
- For the `User` model, the database automatically tracks _when_ a record is created via the field annotated with the `@createdAt` directive.
- For the `Post` model, the database automatically tracks _when_ a record is created and updated via the fields annotated with the `@createdAt` and `@updatedAt` directives.

#### 4. Deploy datamodel (migrate database)

With `prisma.yml` and `datamodel.prisma` in place, you can deploy your datamodel:

```
prisma deploy
```

The underlying database that's created by Prisma is called after the service name and stage (separated by the `$` character). Because the `endpoint` in `prisma.yml` doesn't use these properties, Prisma defaults to `default` for both. The database name consequently is `default$default`.

Here is an overview of the tables that are being created in `default$default`:

##### `catgegory`

```sql
CREATE TABLE "default$default"."category" (
    "id" varchar(25) NOT NULL,
    "name" text NOT NULL,
    PRIMARY KEY ("id")
);
```

| index_name | index_algorithm | is_unique | column_name |
| --- | --- | --- | --- |
| `category_pkey` | `BTREE` | `TRUE` | `id` |


##### `post`

```sql
CREATE TABLE "default$default"."post" (
    "id" varchar(25) NOT NULL,
    "author" varchar(25),
    "published" bool NOT NULL,
    "createdAt" timestamp(3) NOT NULL,
    "updatedAt" timestamp(3) NOT NULL,
    "title" text NOT NULL,
    PRIMARY KEY ("id")
);
```

| index_name | index_algorithm | is_unique | column_name |
| --- | --- | --- | --- |
| `post_pkey` | `BTREE` | `TRUE` | `id` |

##### `post_to_category`

```sql
CREATE TABLE "default$default"."post_to_category" (
    "category" varchar(25) NOT NULL,
    "post" varchar(25) NOT NULL
);
```

| index_name | index_algorithm | is_unique | column_name |
| --- | --- | --- | --- |
| `post_to_category_AB_unique` | `BTREE` | `TRUE` | `category`,`post`|
| `post_to_category_B` | `BTREE` | `FALSE` | `post`|

##### `profile`

```sql
CREATE TABLE "default$default"."profile" (
    "id" varchar(25) NOT NULL,
    "bio" text NOT NULL,
    PRIMARY KEY ("id")
);
```

| index_name | index_algorithm | is_unique | column_name |
| --- | --- | --- | --- |
| `profile_pkey` | `BTREE` | `TRUE` | `id` |

##### `user`

```sql
CREATE TABLE "default$default"."user" (
    "id" varchar(25) NOT NULL,
    "email" text NOT NULL,
    "name" text,
    "role" text NOT NULL,
    "createdAt" timestamp(3) NOT NULL,
    "profile" varchar(25),
    PRIMARY KEY ("id")
);
```

| index_name | index_algorithm | is_unique | column_name |
| --- | --- | --- | --- |
| `user_pkey` | `BTREE` | `TRUE` | `id` |
| `default$default.user.email._UNIQUE` | `BTREE` | `TRUE` | `email` |


#### 5. View and edit your data

From here on, you can use the [Prisma client](https://www.prisma.io/docs/prisma-client/) if you want to access the data in your database programmatically. In the following, we'll highlight two _visual_ ways to interact with the data.

##### Using Prisma Admin

To access your data in [Prisma Admin](https://www.prisma.io/docs/prisma-admin/), you need to navigate to the Admin endpoint of your Prisma project: `http://localhost:4466/_admin`

![](https://imgur.com/wJCyxVy.png)

##### Using TablePlus

While Prisma Admin is focussed on convenient data management workflows, you can also connect to your database from other database GUIs. In contrast to Admin, these tools typically highlight the actual database structure (instead of the Prisma datamodel abstraction). In this example, we're using [TablePlus](https://tableplus.io/). 

> To connect to your database from a database GUI, you need to [map the port](https://docs.docker.com/compose/compose-file/#ports) in the database configuration of your Docker Compose file. In our case, this is why we're adding the `5432:5432` line to it.

When opening TablePlus, you need to:

1. Create a new connection
1. Select **PostgreSQL**
1. Provide a the database connection details:
    - **Name**: Can be anything, e.g. `Local PostgreSQL`
    - **Host/Socket**: `localhost`
    - **Port**: `5432`
    - **User**: `prisma`
    - **Password**: `prisma` 
    - **Database**: `prisma`
1. Click **Connect**

After you connected to the database, you can explore the data and table structure in the TablePlus GUI:

![](https://imgur.com/AFjmZV5.png)

### Option 2: With an existing database

#### 1. Introspect database & generate datamodel

If you have a database that already contains some data, you can get started by introspecting the database schema and let Prisma generate the right datamodel for you.

To get started, run:

```
prisma init --prototype
```

This launches an interactive wizard that guides you through the process of connecting to your existing database. The following interaction shows how to connect to a local PostgreSQL database:

```
$ prisma init myapp --prototype
? Set up a new Prisma server or deploy to an existing server? Use existing database
? What kind of database do you want to deploy to? PostgreSQL
? Does your database contain existing data? Yes
? Enter database host localhost
? Enter database port 5432
? Enter database user prisma
? Enter database password prisma
? Enter database name (the database includes the schema) myblog
? Use SSL? No
? Please select the schema you want to introspect public
Introspecting database public 113ms
Created datamodel definition based on 6 tables.
? Select the programming language for the generated Prisma client Don't generate

Created 3 new files:                                                                          

  prisma.yml           Prisma service definition
  datamodel.prisma    GraphQL SDL-based datamodel (foundation for database)
  docker-compose.yml   Docker configuration file

Next steps:

  1. Start your Prisma server: docker-compose up -d
  2. Deploy your Prisma service: prisma deploy
  3. Read more about introspection:
     http://bit.ly/prisma-introspection
```

Once the process has terminated, the Prisma CLI will have generated the following files inside the new `myapp` directory for you:

- `docker-compose.yml`: The Docker Compose file that specifies your Prisma server and its database connection.
- `prisma.yml`: The root configuration file for your Prisma project.
- `datamodel.prisma`: Your database schema represented in datamodel v1.1 syntax.

**IMPORTANT**: You need to manually [add the `prototype: true` flag](#3-set-the-prototype-flag-to-true-in-prisma_config) to `docker-compose.yml`:

```yml
port: 4466
prototype: true
databases:
  default:
    connector: postgres # or mysql
    host: 127.0.0.1
    port: 5432
    user: prisma
    password: prisma
```

#### 2. Deploy Prisma setup

Now you can deploy your Prisma setup:

```
cd myapp
docker-compose up -d
prisma deploy
```

#### 3. View and edit your data

From here on, you can use the [Prisma client](https://www.prisma.io/docs/prisma-client/) if you want to access the data in your database programmatically. 

To access your data in [Prisma Admin](https://www.prisma.io/docs/prisma-admin/), you need to navigate to the Admin endpoint of your Prisma project: `http://localhost:4466/_admin`

### Option 3: Migrating from datamodel v1.1

#### 1. Datamodel v1 setup

Assume you have a deployed Prisma project with the following datamodel:

```graphql
type User {
  id: ID! @unique
  createdAt: DateTime!
  email: String! @unique
  name: String
  role: Role @default(value: "USER")
  posts: [Post!]!
  profile: Profile
}

type Profile {
  id: ID! @unique
  user: User!
  bio: String!
}

type Post {
  id: ID! @unique
  createdAt: DateTime!
  updatedAt: DateTime!
  title: String!
  published: Boolean! @default(value: "false")
  author: User!
  categories: [Category!]!
}

type Category {
  id: ID! @unique
  name: String!
  posts: [Post!]!
}

enum Role {
  USER
  ADMIN
}
```

When using the datamodel v1, the following tables are created by Prisma in the underlying database:

- `User`
- `Profile`
- `Post`
- `Category`
- `_CategoryToPost`
- `_PostToUser`
- `_ProfileToUser`
- `_RelayId`

Each relation is represented via a relation table and the `_RelayId` table to be able to identity any record by its ID. With the datamodel v1, these are Prisma opinionations that can not be worked around.

#### 2. Enable datamodel v1.1

To enable the datamodel v1.1, you need to:

- Use the [latest beta](https://www.prisma.io/docs/releases-and-maintenance/releases-and-beta-access/installing-the-beta-b5op/) of Prisma (Prisma server _and_ CLI)
- Manually [add the `prototype: true` flag](#3-set-the-prototype-flag-to-true-in-prisma_config) to your Docker Compose file

For example, if you have are connecting to a local PostgreSQL database, your updated `docker-compose.yml` might look as follows:

```yml
version: '3'
services:
  prisma:
    image: prismagraphql/prisma:1.30-beta
    restart: always
    prototype: true
    ports:
    - "4466:4466"
    environment:
      PRISMA_CONFIG: |
        port: 4466
        databases:
          default:
            connector: postgres
            host: localhost
            user: prisma
            password: prisma
            port: '5432'    
```

Once you have specified the `prismagraphql/prisma:1.30-beta` image and introduced the `prototype: true` flag, you can redeploy your Prisma server:

```
docker-compose up -d
```

Now you can install the latest CLI beta, e.g. with npm:

```
npm install -g prisma@beta
```

At this point, you can already try to redeploy your Prisma datamodel. It won't work and the CLI will show some errors:

```
$ prisma deploy
Deploying service `default` to stage `default` to server `local` 585ms

Errors:

  User
    ✖ One field of the type `User` must be marked as the id field with the `@id` directive.
    ✖ The value "USER" is not a valid default for fields of type Enum.

  Profile
    ✖ One field of the type `Profile` must be marked as the id field with the `@id` directive.

  Post
    ✖ One field of the type `Post` must be marked as the id field with the `@id` directive.
    ✖ The value "false" is not a valid default for fields of type Boolean.

  Category
    ✖ One field of the type `Category` must be marked as the id field with the `@id` directive.

Deployment canceled. Please fix the above errors to continue deploying.
Read more about deployment errors here: https://bit.ly/prisma-force-flag
```

#### 3. Migrating to datamodel v1.1 syntax

To fix the errors that Prisma threw after `prisma deploy`, you need to:

- Use the `@id` directive instead of `@unique` on the `id` fields of your models
- Remove the quotes around the arguments of the `@default` directives
- Specify a relation type (_inline_ or _relation table_) on the one-to-one relation between `User` and `Profile` (i.e. add the `@relation` directive with the `link` argument to one end of the relation)

Here is the datamodel updated to the new syntax:

```graphql
type User {
  id: ID! @id
  createdAt: DateTime!
  email: String! @unique
  name: String
  role: Role @default(value: USER)
  posts: [Post!]!
  profile: Profile @relation(link: TABLE)
}

type Profile {
  id: ID! @id
  user: User!
  bio: String!
}

type Post {
  id: ID! @id
  createdAt: DateTime!
  updatedAt: DateTime!
  title: String!
  published: Boolean! @default(value: false)
  author: User!
  categories: [Category!]!
}

type Category {
  id: ID! @id
  name: String!
  posts: [Post!]!
}

enum Role {
  USER
  ADMIN
}
```

Now you can deploy the datamodel with the new syntax:

```
prisma deploy
```

#### 4. Optimizing the database schema

When updating the datamodel syntax to v1.1, the existing database schema will remain the same, therefore keeping the opinionations of the datamodel v1. For example, you can't turn a relation that was represented via a relation table in v1 into an inline relation using the Prisma migration system. 

If you want to optimize your database schema and take advantage of the new features, there are two options:

- Option 1: Export the data from the old project and import it into a new Prisma project where the optimisations are applied
- Option 2: Manually migrate the database schema and subsequently adjust the datamodel to match it

##### 4.1. Export data 

To export the data, you can run the following command (inside the directory where your `prisma.yml` is located):

```
prisma export
```

This creates the following file: `export-TIMESTAMP.zip`, where `TIMESTAMPT` represents the time of the export, e.g. `export-2019-03-21T09:08:48.816Z.zip`.

##### 4.2. Optimize the database schema

Next, you need to initialize a new Prisma project that has a similar configuration as your previous one, e.g. the following `prisma.yml`:

```yml
endpoint: http://localhost:4466
datamodel: datamodel.prisma
```

When creating the new datamodel file, you can copy over your current datamodel and apply the optimizations you want to introduce. In this case, we will:

- Turn the one-to-one relation between `User` and `Profile` into an _inline_ relation tracked via the `User` table
- Turn the one-to-manby relation between `User` and `Post` into an _inline_ relation tracked via the `Post` table

Here's the datamodel that incorporates these changes, put it into a file called `datamodel.prisma`:

```graphql
type User {
  id: ID! @id
  createdAt: DateTime!
  email: String! @unique
  name: String
  role: Role @default(value: USER)
  posts: [Post!]!
  profile: Profile @relation(link: INLINE)
}

type Profile {
  id: ID! @id
  user: User!
  bio: String!
}

type Post {
  id: ID! @id
  createdAt: DateTime!
  updatedAt: DateTime!
  title: String!
  published: Boolean! @default(value: false)
  author: User! @relation(link: INLINE)
  categories: [Category!]!
}

type Category {
  id: ID! @id
  name: String!
  posts: [Post!]!
}

enum Role {
  USER
  ADMIN
}
```

> Note that `@relation(link: INLINE)` on `author` could also be omitted since `INLINE` is the default relation type for a one-to-many relation.

With these changes in place, you can deploy the datamodel: 

```
prisma deploy
```

The new database schema now has the following (empty) tables:

- `User`
- `Profile`
- `Post`
- `Category`
- `_CategoryToPost`

##### 4.3. Import the data

## What's new in datamodel v1.1?

### Directives

#### `@db(name: String!)`

TBD

#### `@linkTable`

TBD

#### `@id`

TBD

#### `@createdAt` & `@updatedAt`

TBD

#### `@relation`

TBD

### Relations

[Draft](https://gist.github.com/nikolasburk/e02d0fcb1302fddf80ecaf841ca64446)

## Migrations and introspection with the Prisma CLI

### `prisma deploy`

The command `prisma deploy` got enhanced. From now on, there will be two modes that allow developers to choose whether the Prisma migration system should migrate their database:

- `prisma deploy`: This command reads your datamodel and migrates the underlying database to match it.
- `prisma deploy --no-migrate`: This command does *not* migrate the underlying database. Instead, it expects the database schema to be in the right state already (which means you need to manually migrate the database _before_ running `prisma deploy --no-migrate`)! If the datamodel and database schema do not align, `prisma deploy` will throw an error.

### `prisma introspect`

The command `prisma introspect` has been adapted to output the new datamodel format when using if with the `--prototype` flag: 

```
prisma introspect --prototype
```

The command has been improved so that it will pay respect to existing datamodel files. This way you can use it to translate your database schema into a datamodel while the introspection will preserve the ordering of types and fields of your existing datamodel file.

### Common workflows

You can fluently switch between those two commands however you like. For instance you could use `prisma deploy` for most of your migration needs. When you hit an advanced use case where the Prisma migration system is not yet powerful enough, you can:

1. Migrate the database manually 
1. Use `prisma introspect --prototype` to update your datamodel with those changes from the database
1. Use `prisma deploy --no-migrate` to let the Prisma server know about that

## FAQ

#### Where should I report bugs?

Please report bugs directly [here](https://github.com/prisma/datamodel-v1.1-for-sql-beta/issues) in this repo!

#### What is not part of this beta?

1. [Multi column indexes](https://github.com/prisma/prisma/issues/3405) are not of this beta yet. We are currently working on implementing them.
2. [Polymorphic relations](https://github.com/prisma/prisma/issues/3407) are not part of this beta. We will make a separate effort to implement them.


#### How can I migrate my Prisma project with existing data to a new optimised database schema (e.g. JOIN tables have been removed)? 

We recommend the following:

1. Use `prisma export` to export the data from your existing Prisma project.
1. Copy your existing datamodel to a new Prisma project and apply your desired optimisations to your datamodel.
1. Use `prisma import` to import the data into your new Prisma project.
