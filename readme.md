```
Original repo - https://github.com/jamiepine/prismix
```

# `Prismix`
 
### *The Prisma schema mixer 🍹*
_Made for Prisma `^2.0`_

[![Downloads](https://img.shields.io/npm/dt/prismix.svg?style=flat&colorA=000000&colorB=000000)](https://www.npmjs.com/package/prismix)
[![Downloads](https://img.shields.io/npm/v/prismix.svg?style=flat&colorA=000000&colorB=000000)](https://www.npmjs.com/package/prismix)


Prisma restricts your schema to a single file, Prismix allows you to write as many schema files as you'd like, wherever you like—all while supporting cross-file model relations 🤯

> Learn more about Prisma: [prisma.io](https://prisma.io)


Unlike `prisma-merge`, Prismix allows model relations to exist between files by combining models and enums, allowing you to extend and override Models as you please. This is ideal when working in a monorepo where parts of your schema need to exist in separate modules.


## Installation
1. Install Prismix
```
yarn add prismix --dev
```
or NPM
```
npm install prismix --dev
```
2. Create a `prismix.config.json` file in the root of your project. This allows you to define how you would like Prismix to merge your schemas. 
```json
{
  "mixers": [
    {
        "input": [
            "base.prisma",
            "./modules/auth/auth.prisma", 
            "./modules/posts/posts.prisma",
        ],
        "output": "prisma/schema.prisma"
    }
  ]
}
```
The order of your input files effects how overrides are considered, the later inputs take priority over the earlier inputs.

The default `output` value is `prisma/schema.prisma` (it can be omitted from the config) and Prisma encourages you to keep is as such, especially if you want to use `prisma format`.

3. Add the `npx prismix` command as a prefix to your `package.json` scripts.
```json
{
  "name": "my-app",
  "version": "1.0.0",
  "scripts": {
    "prismix": "npx prismix && prisma format",
    "dev": "yarn prismix && ts-node server.ts",
  }
}
```
Or just run `npx prismix` from within the repo that contains the Prismix config file. 

Using `prisma format` is optional, but I like clean code.

Note: If you are using a monorepo you won't be able to run this command from the root, if you add a script `"prismix": "npx prismix"` you could run `yarn workspace my-app prismix`.

## Cross-file relations 

In order to relate to a model defined in another file you **must create an alias model**. This is simply a model with only the `@id` field. 

This is to ensure Prisma can parse the schema file, as Prismix uses the Prisma SDK to parse each file individually before merging models at field level.

```prisma
model Post {
    id String @id @default(autoincrement())
}
```
In the above example `Post` would be a model defined in another schema, any fields added in this schema file will be merged with any `Post` model(s) in other schema files. The `@id` field is the only required field in an extended model, however you may add fields specific to a relation. (this is shown in greater detail below)

## Example
Let's go over how Prismix merges schemas. We'll keep it simple with two schemas that need to relate to each other.

## `base.prisma`


```prisma
generator client {
    provider = "prisma-client-js"
}

datasource db {
    provider = "postgresql"
    url      = "postgresql://..."
}

model Account {
    id       Int    @id @default(autoincrement())
    username String
    email    String
    status   String

    @@map("accounts")
}
```
We've established our `generator` and `datasource`, as well as our first model, `Account`. It is required that these two definitions be present in at least one of your schemas.

## `posts.prisma`
Now we'll create the Posts schema in a different file. In order for posts to relate to accounts we can define an **empty model** to represent the account.

```prisma
model Posts {
    id         Int     @id @default(autoincrement())
    title      String
    content    String
    account_id Int
    account    Account @relation(fields: [account_id], references: [id])

    @@map("posts")
}

model Account {
    id     String @id
    posts  Post[]
}
```
When Prismix merges these two schemas the relations will be connected. 

## `schema.prisma`
This is the generated file, do not edit this file!

```prisma
// *** GENERATED BY PRISMIX :: DO NOT EDIT ***

generator client {
    provider = "prisma-client-js"
}

datasource db {
    provider = "postgresql"
    url      = "postgresql://..."
}

model Account {
    id       Int    @id @default(autoincrement())
    username String
    email    String
    status   String
    posts    Post[]
}

model Posts {
    id         Int     @id @default(autoincrement())
    title      String
    content    String
    account_id Int
    account    Account @relation(fields: [account_id], references: [id])

    @@map("posts")
}
```

As you can see the property `posts` was added on to the original Account schema and the `account` relation on the Posts schema links to the original Account schema.



## How it works
Using the Prisma SDK we parse the input schemas into a DMMF objects, then process the schema merge into a single DMMF object as per the config file, finally it is converted back into Prisma schema format using a custom deserializer, adapted from `@IBM/prisma-schema-transformer` and written to the output location.


## To-do
- [x] Make it work
- [x] Add glob support for wildcard schema discovery
- [ ] Make `prismix.config.json` optional
- [ ] Add command flags

Created by [@jamiepine](https://twitter.com/jamiepine)
