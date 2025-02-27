---
id: api-makeSchema
title: makeSchema
sidebar_label: makeSchema
---

Defines the GraphQL schema, by combining the GraphQL types defined by the GraphQL Nexus layer or any manually defined GraphQL named types (Scalars, ObjectTypes, Enums, etc).

We require at least one type be named "Query", which will be used as the root query type. The `makeSchema` takes several options which all should be specified by the user, detailed below:

## types

The `types` property is required, and should contain all of the possible Nexus/GraphQL types that make up a schema (at least the root-level ones), imported into one place. The type is opaque, it can be an array or object, we recursively walk through the object looking for types, so the following will all have the same result, and non-GraphQL objects will be ignored.

- `{ typeA, typeB, typeC, typeD }`
- `[[typeA], [{ typeB: typeB }], typeC, { typeD }]`
- `[typeA, typeB, typeC, typeD]`

```ts
import { makeSchema } from "nexus";
import * as types from "./allNexusTypes";

export const schema = makeSchema({
  types,
});
```

## plugins

The `plugins` property is an array for adding "Plugins", or ways of extending/changing the runtime behavior of Nexus and GraphQL. Unlike the `types` property, this must be an array, and the order of the plugins matters because this influences the order of any resolver "middleware" the plugin may optionally provide.

```ts
import { makeSchema, nullabilityGuard, authorizeField } from "nexus";
import * as types from "./allNexusTypes";

export const schema = makeSchema({
  types,
  plugins: [
    authorizeField({
      /* ... */
    }),
    nullabilityGuard({
      /* ... */
    }),
  ],
});
```

## shouldGenerateArtifacts, outputs, typegenAutoConfig

The `shouldGenerateArtifacts` is a boolean value which determines whether artifact files (graphql and TypeScript) are emitted when the code for `makeSchema`

`outputs` is an object which specifies the absolute path for where the emitted files are generated.
If you do not wish to generate one of these types

`typegenAutoConfig` is an object which gives nexus more information about how to properly generate the
type definition file. An example of the `sources` is provided below:

```ts
makeSchema({
  types,
  shouldGenerateArtifacts: process.env.NODE_ENV === "development",
  outputs: {
    // I tend to use `.gen` to denote "auto-generated" files, but this is not a requirement.
    schema: path.join(__dirname, "generated/schema.gen.graphql"),
    typegen: path.join(__dirname, "generated/nexusTypes.gen.ts"),
  },
  typegenAutoConfig: {
    headers: [
      'import { ConnectionFieldOpts } from "@packages/api-graphql/src/extensions/connectionType"',
    ],
    sources: [
      // Automatically finds any interface/type/class named similarly to the and infers it
      // the "source" type of that resolver.
      {
        source: "@packages/types/src/db.ts",
        alias: "dbt",
        typeMatch: (name) =>
          new RegExp(`(?:interface|type|class)\\s+(${name}s?)\\W`, "g"),
      },
      // We also need to import this source in order to provide it as the `contextType` below.
      {
        source: "@packages/data-context/src/DataContext.ts",
        alias: "ctx",
      },
    ],
    // Typing from the source
    contextType: "ctx.DataContext",
    backingTypeMap: {
      Date: "Date",
      DateTime: "Date",
      UUID: "string",
    },
    debug: false,
  },
});
```

The [Ghost Example](https://github.com/prisma-labs/nexus/blob/develop/examples/ghost/src/ghost-schema.ts) is the best place to look for an example of how we're able to capture the types from existing runtime objects or definitions and merge them with our schema.

## shouldExitAfterGenerateArtifacts

If you are not checking in your artifacts and wish to run them, this will allow you to exit right after the artifacts have been generated. There is no default behavior for this, but you could do something like the following, to be able to run a script which will exit if `--nexus-exit` is provided:

```ts
makeSchema({
  // ... options like above
  shouldExitAfterGenerateArtifacts: process.argv.includes("--nexus-exit"),
});
```

```sh
ts-node -T ./path/to/my/schema.ts --nexus-exit
```

## prettierConfig

Either an absolute path to a `.prettierrc` file, or an object with a valid "prettier" config options.

```ts
makeSchema({
  // ... options like above
  prettierConfig: path.join(__dirname, "../../../.prettierrc"),
});
```

## nonNullDefaults

Controls the nullability of the input / output types emitted by `nexus`. The current Nexus default is
`{ output: true, input: false }`, though the `graphql-js` / spec default is `{ output: false, input: false }`.

You should make a decision on this and supply the option yourself, it may be changed / required in the future.

Read more on this in the [getting-started](getting-started.md) guide.

### typegenConfig, formatTypegen

Escape hatches for more advanced cases which need further control over. You typically won't need these.

### customPrintSchemaFn

Optional, allows you to override the `printSchema` when outputting the generated `.graphql` file:

```ts
makeSchema({
  // ...
  customPrintSchemaFn: (schema) => {
    return printSchema(schema, { commentDescriptions: true });
  },
});
```

#### Footnotes: Annotated config option for typegenAutoConfig:

```ts
export interface TypegenAutoConfigOptions {
  /**
   * Any headers to prefix on the generated type file
   */
  headers?: string[];
  /**
   * Array of files to match for a type
   *
   *   sources: [
   *     { source: 'typescript', alias: 'ts' },
   *     { source: path.join(__dirname, '../backingTypes'), alias: 'b' },
   *   ]
   */
  sources: TypegenConfigSourceModule[];
  /**
   * Typing for the context, referencing a type defined in the aliased module
   * provided in sources e.g. 'alias.Context'
   */
  contextType?: string;
  /**
   * Types that should not be matched for a backing type,
   *
   * By default this is set to ['Query', 'Mutation', 'Subscription']
   *
   *   skipTypes: ['Query', 'Mutation', /(.*?)Edge/, /(.*?)Connection/]
   */
  skipTypes?: (string | RegExp)[];
  /**
   * If debug is set to true, this will log out info about all types
   * found, skipped, etc. for the type generation files.
   */
  debug?: boolean;
  /**
   * If provided this will be used for the backing types rather than the auto-resolve
   * mechanism above. Useful as an override for one-off cases, or for scalar
   * backing types.
   */
  backingTypeMap?: Record<string, string>;
}

export interface TypegenConfigSourceModule {
  /**
   * The module for where to look for the types.
   * This uses the node resolution algorithm via require.resolve,
   * so if this lives in node_modules, you can just provide the module name
   * otherwise you should provide the absolute path to the file.
   */
  source: string;
  /**
   * When we import the module, we use 'import * as ____' to prevent
   * conflicts. This alias should be a name that doesn't conflict with any other
   * types, usually a short lowercase name.
   */
  alias: string;
  /**
   * Provides a custom approach to matching for the type
   *
   * If not provided, the default implementation is:
   *
   *   (type) => [
   *      new RegExp(`(?:interface|type|class|enum)\\s+(${type.name})\\W`, "g")
   *   ]
   *
   */
  typeMatch?: (
    type: GraphQLNamedType,
    defaultRegex: RegExp
  ) => RegExp | RegExp[];
  /**
   * A list of typesNames or regular expressions matching type names
   * that should be resolved by this import. Provide an empty array if you
   * wish to use the file for context and ensure no other types are matched.
   */
  onlyTypes?: (string | RegExp)[];
  /**
   * By default the import is configured 'import * as alias from', setting glob to false
   * will change this to 'import alias from'
   */
  glob?: false;
}
```
