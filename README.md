# Datamodel v1.1 for SQL

*What is this beta all about?*
In the last months we have worked with the community to define a new (datamodel specification)[https://github.com/prisma/prisma/issues/3408] called v1.1. With our old Datamodel the Prisma user experience was very different depending on whether you were using Prisma with a new database or an existing database. The syntax was slightly different and it was not possible to use our migration system on top of an existing database. The main goal of this effort was to unify this experience. From now on you can use the migration system with existing databases as well.
This also implies that the user can from now on take control over the layout of the database schema. Prismas migration system used to only allow the use of join tables to realise relations. This limitation has been lifted with this effort.

*Prerequisites*:
1. Use the latest alpha Prisma CLI: `npm install -g prisma@alpha`
2. Use the latest alpha Prisma Docker image: `prismagraphql/prisma:1.28-alpha`
3. Enable the datamodel v1.1 by adding the `prototype: true` flag to your server config. E.g.:
```yaml
port: 4466
prototype: true
databases:
  default:
    connector: postgres
    host: 127.0.0.1
    port: 5432
    user: postgres
    password: prisma
    rawAccess: true
```

Have a look at the `example` folder for a setup that is ready to use.

*What do i need to know apart from the Datamodel changes?*
The command `prisma deploy` got enhanced. From now on there will be 2 modes that allow you to choose whether the Prisma migration system should migrate your database or not.
1. `prisma deploy`: This command takes your Datamodel and migrates the underlying database to match it.
1. `prisma deploy --no-migrate`: This command does *not* migrate the underlying database. Instead it expects the database schema to be in the right state. If the Datamodel and database schema do not align you will get an error.

The command `prisma introspect` has been adapted to output the new datamodel format. Make sure to use the `--prototype` flag when you use it: `prisma introspect --no-prototype`. The command has been improved so that it will pay respect to existing Datamodel files. This way you can use this command to translate your database schema into a datamodel while the introspection will preserve the ordering of types and fields of your existing datamodel file :pray:

You can fluently switch between those two commands however you like. For instance you could use `prisma deploy` for most of your migration needs. When you hit an advanced usecase where the Prisma migration system is not powerful enough yet for your needs, migrate the database manually. Then use `prisma introspect --no-prototype` to update your Datamodel with those changes from the database and `prisma deploy --no-migrate` to let the Prisma server know about that. Afterwards you can start to use `prisma deploy` again. 

*Where should i report bugs?*
Please report bugs directly here in this repo! :pray:

*What is not part of this beta?*
1. [Multi column indexes](https://github.com/prisma/prisma/issues/3405) are not of this beta yet. We are currently working on implementing them.
2. [Polymorphic relations](https://github.com/prisma/prisma/issues/3407) are not part of this beta. We will make a separate effort to implement them.
