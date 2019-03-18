# Datamodel v1.1

## Motivation

Over the last months, we have worked with the community to define a new [datamodel specification](https://github.com/prisma/prisma/issues/3408) for Prisma. This new version is called datamodel v1.1.

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

### From scratch

TBD

### With an existing database

TBD

### Migrating from datamodel v1.1

TBD

## Migrations and introspection with the Prisma CLI

### `prisma deploy`

The command `prisma deploy` got enhanced. From now on, there will be two modes that allow developers to choose whether the Prisma migration system should migrate their database:

- `prisma deploy`: This command reads your datamodel and migrates the underlying database to match it.
- `prisma deploy --no-migrate`: This command does *not* migrate the underlying database. Instead, it expects the database schema to be in the right state already (which means you need to manually migrate the database _before_ running `prisma deploy --no-migrate`)! If the datamodel and database schema do not align, `prisma deploy` will throw an error.

### `prisma introspect`

The command `prisma introspect` has been adapted to output the new datamodel format. Make sure to use the `--prototype` flag when you use it: `prisma introspect --prototype`. The command has been improved so that it will pay respect to existing Datamodel files. This way you can use this command to translate your database schema into a datamodel while the introspection will preserve the ordering of types and fields of your existing datamodel file :pray:

You can fluently switch between those two commands however you like. For instance you could use `prisma deploy` for most of your migration needs. When you hit an advanced usecase where the Prisma migration system is not powerful enough yet for your needs, migrate the database manually. Then use `prisma introspect --prototype` to update your Datamodel with those changes from the database and `prisma deploy --no-migrate` to let the Prisma server know about that. Afterwards you can start to use `prisma deploy` again. 

## Where should I report bugs?

Please report bugs directly here in this repo! :pray:

## What is not part of this beta?

1. [Multi column indexes](https://github.com/prisma/prisma/issues/3405) are not of this beta yet. We are currently working on implementing them.
2. [Polymorphic relations](https://github.com/prisma/prisma/issues/3407) are not part of this beta. We will make a separate effort to implement them.

## FAQ

**How can I migrate my Prisma project with existing data to a new optimised Database schema? E.g. join tables have been removed**

We recommend the following:

1. Use `prisma export` to export the data from your existing Prisma project.
1. Copy your existing Datamodel to a new Prisma project and apply your desired optimisations to your Datamodel.
1. Use `prisma import` to import the data into your new Prisma project.

