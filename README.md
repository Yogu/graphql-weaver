# NO LONGER MAINTAINED

We no longer us this library at AEB, so unfortunately, we are not able to maintain it properly. The repository is therefor archived.

# graphql-weaver

[![npm version](https://badge.fury.io/js/graphql-weaver.svg)](https://npmjs.org/graphql-weaver) [![Build Status](https://github.com/AEB-labs/graphql-weaver/workflows/Node%20CI/badge.svg)](https://github.com/AEB-labs/graphql-weaver/actions?workflow=Node+CI)

A tool to combine, link and transform GraphQL schemas

Use graphql-weaver if you have multiple GraphQL servers and want to combine them into one API. Features like namespacing, links and custom transformation modules allow you to augment the API as you like.

**[Try it online in Apollo Launchpad](https://launchpad.graphql.com/0m5x0k495)**

## How to use

```bash
npm install --save graphql-weaver
```

Basic usage:

```typescript
const schema: GraphQLSchema = await weaveSchemas({
    endpoints: [{
        namespace: 'model',
        typePrefix: 'Model',
        url: 'http://localhost:8080/graphql' // url to a GraphQL endpoint
    }, {
        namespace: 'local',
        schema: new GraphQLSchema(/* ... */) // use schema instance directly
    }]
})
```

A *woven schema* is an executable GraphQL schema built from several *endpoints*. For each endpoint, you can either specify a URL to a GraphQL server, pass an executable GraphQL schema instance, or implement the [`GraphQLClient`](src/graphql-client/graphql-client.ts) interface yourself.

In its basic configuration, `weaveSchemas` merges the query, mutation and subscription fields of all endpoints. To avoid name collisions, you can specify the `namespace` and `typePrefix` properties like seen above. The `typePrefix` will be prepended to all types; `namespace` causes the fields of this endpoint to be wrapped in a field, to be queried via `{ model { aFieldOfModel } }`.

### Links

In the spirit of GraphQL, this tool allows you to create links between objects of different endpoints. Suppose you have a music recommendation service and a music library service. You can make the whole properties of a song available in the recommendation API without the recommendation service knowing all song properties.

```typescript
const schema: GraphQLSchema = await weaveSchemas({
    endpoints: [{
        namespace: 'library',
        url: 'http://example.com/library/graphql'
    }, {
        namespace: 'recommendations',
        url: 'http://example.com/recommendations/graphql',
        fieldMetadata: {
            'Recommendation.song': { // Field song in type Recommendation
                link: {
                    field: 'library.Song', // field Song in namespace library
                    argument: 'id', // argument of library.Song
                    batchMode: false,
                }
            }
        }
     }]
});
```
This assumes the library schema has a field `Song` with argument `id`, and the recommendations schema has a type `Recommendation` with a field `song` which contains the song id. Then, you can query the recommendations with all song information like this:

```graphql
query {
    recommendations {
        myRecommendations {
            recommendedAt
            song {
                id
                artist
                title
                year
            }
        }
    }
}
```

If there are many recommendations, this is ineficcient because all songs are queried independently.  If the library schema supports querying multiple songs at once, you can set `batchMode` to `true`. If the library schema may return the songs in a different order than the ids its get, you need to set `keyField` too.

```typescript
const schema: GraphQLSchema = await weaveSchemas({
    endpoints: [{
        namespace: 'library',
        url: 'http://example.com/library/graphql'
    }, {
        namespace: 'recommendations',
        url: 'http://example.com/recommendations/graphql',
        fieldMetadata: {
            'Recommendation.song': {
                link: {
                    field: 'library.allSongs',
                    argument: 'filter.ids', // allSongs has an argument filter with an array field ids
                    batchMode: true,
                    keyField: 'id' // the name of a field in Song type that contains the id
                }
            }
        }
     }]
});
```

In the case where you want to perfom a link from a single id to a list of many related items, include `linkFieldName` as an alias for the linked list of items, and the `oneToMany` flag. Since we are making a single request for a GraphQL list of items, `batchMode` is not required and should be set to `false`. 
```typescript
const schema: GraphQLSchema = await weaveSchemas({
    endpoints: [{
        namespace: 'library',
        url: 'http://example.com/library/graphql'
    }, {
        namespace: 'recommendations',
        url: 'http://example.com/recommendations/graphql',
        fieldMetadata: {
            'Artist.name': { // Field name in type Artist
                link: {
                    field: 'library.allSongs', // field allSongs in namespace library
                    argument: 'filter.artistName', // argument of library.Song
                    batchMode: false, // batchMode must be set to false
                    oneToMany: true,
                    linkFieldName: 'songsByThisArtist' // the "virtual" field that will be populated
                }
            }
        }
     }]
});
```
This assumes the library schema has a field `allSongs` which accepts a `filter` argument, where one of the filter properties is `artistName`. This also assumes that the recommendations schema has a type `Artist` with a field `name` which contains the artists name. This configuration will add new virtual field `songsByThisArtist` on the `Artist` type which will allow you to query an Artist along with their songs like this:

```graphql
query {
    recommendations {
        myFavoriteArtists {
            name
            genre
            songsByThisArtist {
                id
                title
                year
            }
        }
    }
}
```
### Joins

What if you want to sort the recommendations by the song age, or filter by artist? The recommendation service currently does not know about these fields, so it does not offer an API to sort or order by any of them. Using graphql-weaver, this problem is easily solved:

```typescript
const schema: GraphQLSchema = await weaveSchemas({
    endpoints: [{
        namespace: 'library',
        url: 'http://example.com/library/graphql'
    }, {
        namespace: 'recommendations',
        url: 'http://example.com/recommendations/graphql',
        fieldMetadata: {
            'Recommendation.song': {
                link: {
                    field: 'library.allSongs',
                    argument: 'filter.ids',
                    batchMode: true, // is now required
                    keyField: 'id' // this one too
                }
            },
            'Query.myRecommendations': { // Field myRecommendations on type Query
                join: {
                    linkField: 'song', // The field name song in the type Recommendation
                }
            }
        }
     }]
});
```

This assumes that the library service offers a way to filter and sort songs via the `orderBy` and `filter` arguments. Using it is simple:

```graphql
query {
    recommendations {
        myRecommendations(filter: { song: { artist: "Ed Sheeran" } }, orderBy: song_year_DESC) {
            recommendedAt
            song {
                id
                artist
                title
                year
            }
        }
    }
}
```

A note on efficiency: The list of recommendations should be relatively small (not more than a few hundred), as all recommendations need to be fetched so that their ids can be sent to the library for filtering and sorting.

### Custom transformations

All four presented features (namespaces, type prefixes, links and joins) are implemented as independent modules. If you need something else, you can just write your own module:

```typescript
class MyModule implements PipelineModule {
    transformExtendedSchema(schema: ExtendedSchema): ExtendedSchema {
        // do something with the schema
        return schema;
    }
    transformQuery(query: Query): Query {
        // do something with the query
        return query;
    }
}

const schema: GraphQLSchema = weaveSchemas({
    endpoints: [{
        namespace: 'library',
        url: 'http://example.com/library/graphql',
        
    }],
    pipelineConfig: {
        transformPreMergePipeline(modules: PipelineModule[], context: PreMergeModuleContext): PipelineModule[] {
            // These modules are executed for each endpoint
            return [
                ...modules,
                new MyModule()
            ]
        },
        transformPostMergePipeline(modules: PipelineModule[], context: PostMergeModuleContext): PipelineModule[] {
            // These modules are executed once for the merged schema
            return [
                ...modules,
                new MyModule()
            ]
        }
    }
});
```

For a simple module, see [`TypePrefixModule`](src/pipeline/type-prefixes.ts). The section *Architecture* below gives an overview over the pipeline architecture.

To simplify modifications to a schema, graphql-weaver ships [`graphql-transformer`](https://github.com/AEB-labs/graphql-transformer) (and [`transformExtendedSchema`](src/extended-schema/extended-schema-transformer.ts)). You can change types and fields as you like with a simple function:

```typescript
const transformedSchema = transformSchema(originalSchema, {
    transformField(field: GraphQLNamedFieldConfig<any, any>, context) {
        // Rename a field in a type
        if (context.oldOuterType.name == 'MyType') {
            return {
                ...field,
                name: field.name + 'ButCooler'
            }
        }
        return field;
    },

    transformObjectType(type: GraphQLObjectTypeConfig<any, any>) {
        if (type.name == 'MyType') {
            return {
                ...type,
                name: 'MyCoolType'
            };
        }
        return type;
    },

    transformFields(fields: GraphQLFieldConfigMap<any, any>, context) {
        // You can even copy types on the fly and transform the copies
        const type2 = context.copyType(context.oldOuterType, {
            transformObjectType(typeConfig: GraphQLObjectTypeConfig<any, any>) {
                return {
                    ...typeConfig,
                    name: typeConfig.name + '2'
                };
            }
        });

        // This just adds a reflexive field "self" to all types, but its type does not have
        // the "self" field (because it is a copy from the original type, see above)
        // it also won't have the "cool" rename applied because the top-level transformers are not applied
        return {
            ...fields,
            self: {
                type: type2,
                resolve: (source: any) => source
            }
        }
    }
});
```

For more information, refer to the [graphql-transformer](https://github.com/AEB-labs/graphql-transformer) project.

### Schema error handling

By default, `weaveSchemas` throws a `WeavingError` if an endpoint schema could not be fetched or the link and join features
are configured incorrectly. You can change this behavior with the `errorHandling` property:

```typescript
weaveSchemas({
    endpoints: [ /* ... */ ],
    errorHandling: WeavingErrorHandlingMode.CONTINUE_AND_REPORT_IN_SCHEMA
})
```

There are four modes available:

* `THROW` is the default behavior which throws all schema errors
* `CONTINUE` ignores errors. If the endpoint schema cannot be created at all, it will be missing in the result config.
    If a join or link feature is malconfigured, this one configuration will be skipped.
    Use `weaveSchemasExt` to retrieve a list of errors.
* `CONTINUE_AND_REPORT_IN_SCHEMA` behaves like `CONTINUE`, but errors are additionally displayed to the user via a
    special _errors field on the root query type.
* `CONTINUE_AND_ADD_PLACEHOLDERS` is like `CONTINUE_AND_REPORT_IN_SCHEMA`, but namespaced endpoints that completely fail
    are also replaced by an object with a field _error. 
    
### Runtime error handling

Since version 0.11, graphql-weaver propagates runtime errors of endpoints transparently and in the most intuitive way.

* If a user requested fields from two endpoints of which one failed and one succeeded, the result data of the successful
    endpoint will be returned and the error of the failing one will be propagated.
* If the endpoint reported errors for some fields but returned data for other fields, graphql-weaver will pass this
    through exactly. The errors will be reported at the correct fields in the woven schema.
* If the error reported by the endpoint included source location information, graphql-weaver will try to map this these
    source location to the woven schema. If the location can not be mapped, the field's location in the woven schema
    will be used, so if you get a location, you can be sure it is a location in your request to graphql-weaver.
    
For the most part, this means everything should work as expected. However, the mapping is more complex as it might
seem, so if you find an error mapping that looks wrong, please open an issue. 

## Contributing

After cloning the repository, run

```bash
npm install
npm start
```

To run the test suite, run

```bash
npm test
```

To debug/run the application (or tests) in WebStorm, right-click on `graphql-weaver.js` (or `graphql-weaver-tests.js`, respectively) and choose *Debug*/*Run*.

### Release workflow

* For **normal development**, create a branch from master, commit and create a merge request to master. 
* To **fix a bug in a previous release**, find the *release-* branch for the corresponding version, increase the *patch* level in `package.json` and push the changes. Once the tests pass, manually trigger the *deploy* stage in Gitlab. You can also release a *-rc.1* version before the actual release for prior testing in dependent modules.
* To prepare a **new feature release** (currently, this means a new minor version), create a `release-0.x` branch from master. Set the version to `0.x-rc.1`, push and manually trigger the *deploy* stage in Gitlab. Test the changes in dependent modules. Once everything is ok, change the version to `0.x` and deploy again. Finally, merge the release branch into *master*. Do not delete the release branch as it is used for hotfixes.

## Architecture

graphql-weaver takes a set of GraphQL endpoints, transforms them through pipelines, merges them, transforms the merged schema again and exposes that as its *woven schema*.

```
           +------+  +------+  +------+
Endpoints  |Schema|  |Schema|  |Schema|
           +------+  +------+  +------+

           +------+  +------+  +------+
            X    X    X    X    X    X
Pipelines    X  X      X  X      X  X
              XX        XX        XX

               +                  +
Merge           +----------------+

                     +------+
                      X    X
Pipeline               X  X
                        XX

                     +------+
Server               |Schema|
                     +------+

```

The *merge* in the middle simply merges all the fields of the Query/Mutation types. All the other features, like type prefixing, field namespacing, even resolvers, is implemented by pipeline modules.

You'll find the list of modules in `src/pipeline/pipeline.ts`. For a description of each module, please refer to the TypeDoc comments.

## Module structure

* `graphql` - general utilities for working with GraphQL schemas and queries
* `extended-schema` - an implementation of storing and exposing metadata on fields, the concept being [discussed on GitHub](https://github.com/facebook/graphql/issues/300)
* `graphql-client` - GraphQL client library, with local and http implementations
* `pipeline` - the core, being framework and modules for graphql-weaver's features
* `config` - configuration parameter types for `weaveSchemas`
* `utils` - utilities unrelated to GraphQL
* `typings` - typings for thirdparty modules
