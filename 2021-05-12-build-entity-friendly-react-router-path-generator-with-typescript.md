```ic-metadata
{
  "name": "Build Entity-Friendly react-router Paths Generator with Typescript",
  "series": null,
  "date": "2021-05-12",
  "lastModifiedDate": "2021-05-12",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "react"],
  "canonicalLink": "https://dev.to/bwca/build-entity-friendly-react-router-paths-generator-with-typescript-fpd"
}
```

# Build Entity-Friendly react-router Paths Generator with Typescript

So the other day I was thinking if it is possible to create a route generator that would be of any use and would respect entities in URLS, i.e. `:entity(post|article)`.

Naturally, `react-router` provides means of generating paths, the `generatePath` function, and while the `@types/react-router` types package does pretty decent job securing the param names, as of yet, it leaves entities vulnerable, without any kind of restrictions, they are treated same as any other param, meaning you can drop `string | number | boolean` into them.

Let's fix that with typescript's 4+ template literal types and generics.

First of all let's figure out what types we want to be allowed to be passed to our parameters. We could go with `string` in `string` out attitude, since when we extract params they are strings, but for the sake of compatibility and tribute to the original `@types/react-router` let's go with union `string | number | boolean`:

```ts
type AllowedParamTypes = string | number | boolean;
```

That's a nice start. Now, we need a type that would represent our union of values for entities, into which we will be dropping all possible values for our entity and recursively adding them to the union:

```ts
type EntityRouteParam<T extends string> =
  /** if we encounter a value with a union */
  T extends `${infer V}|${infer R}`
  /* we grab it and recursively apply the type to the rest */
  ? V | EntityRouteParam<R>
  /** and here we have the last value in the union chain */
  : T;
```

Now we need a param type that can be either an entity which is limited to a union of values, or just a regular param, which is simply an allowed type:

```ts
type RouteParam<T extends string> =
  /** if we encounter an entity */
  T extends `${infer E}(${infer U})`
  /** we take its values in union */
  ? { [k in E]: EntityRouteParam<U> }
  /** if it's an optional entity */
  : T extends `${infer E}?`
  /** we make its values optional as well */
  ? Partial<{ [k in E]: AllowedParamTypes }>
  /** in case it's merely a param, we let any allowable type */
  : { [k in T]: AllowedParamTypes };
```

Now to craft a generic that can break down an url into fragments and extract an interface of params:

```ts
type RouteParamCollection<T extends string> =
  /** encounter optional parameter */
  T extends `/:${infer P}?/${infer R}`
  /** pass it to param type and recursively apply current type
   *  to what's left */
  ? Partial<RouteParam<P>> & RouteParamCollection<`/${R}`>
  /** same stuff, but when the param is optional */
  : T extends `/:${infer P}/${infer R}`
  ? RouteParam<P> & RouteParamCollection<`/${R}`>
  /** we encounter static string, not a param at all */
  : T extends `/${infer _}/${infer R}`
  /** apply current type recursively to the rest */
  ? RouteParamCollection<`/${R}`>
  /** last case, when param is in the end of the url */
  : T extends `/:${infer P}`
  ? RouteParam<P>
  /** unknown case, should never happen really */
  : unknown;
```

That's basically all the magic we need. Now all that's needed is to create a couple of wrapper functions that would provide us with more type safety and run `generatePath` from `react-router` inside under their hoods.

A function for path generation with param and entity hints is pretty simple and you can even use enums with it:

```ts
function routeBuilder<K extends string>(route: K, routeParams: RouteParamCollection<K>): string {
  return generatePath(route, routeParams as any)
}
routeBuilder('/user/:userId/:item(post|article)/', { item: 'article', userId: 2 });
// ^ will get angry if 'item' receives something else than 'post' or 'article'
```

Now we can come up with even more advanced function that could generate route fragments of even longer route, and provide same type safety.

In order to craft such function we first need to make a couple of types for crafting path fragments of a given route, respecting the params in it:

```ts
type RouteFragment<T extends string, Prefix extends string = "/"> = T extends `${Prefix}${infer P}/${infer _}`
  ? `${Prefix}${RouteFragmentParam<P>}` | RouteFragment<T, `${Prefix}${P}/`>
  : T
 
type RouteFragmentParam<T extends string> = T extends `:${infer E}(${infer U})`
  ? EntityRouteParam<U>
  : T extends `:${infer E}(${infer U})?`
  ? EntityRouteParam<U>
  : T
```

And obviously now we need a factory to produce our path builder:

```ts
function fragmentedRouteBuilderFactory<T extends string>() {
  return <K extends RouteFragment<T>>(route: K, routeParams: RouteParamCollection<K>): string => {
    return routeBuilder(route, routeParams as any)
  }
}
const fragmentRouteBuilder = fragmentedRouteBuilderFactory<"/user/:userId/:item(post|article)/:id/:action(view|edit)">();
fragmentRouteBuilder('/user/:userId/:item(post|article)/:id', { userId: 21, item: 'article', id: 12 });
```

Doesn't look that difficult now, does it? :)

Oh, you can also check it out in the [typescript playground](https://www.typescriptlang.org/play?#code/C4TwDgpgBAggNnA9gdwgEwAoEMBOWC2AKuBAM5QC8UpwOAlgHYDmUAPlAwK74BGEObKD0SI4ELAwDcAKGmhIUAKINgdUACVEnYBGx58AHkJQIADx0M05GvWYA+StKhQA9ACo3UOgDMoqEwwAxloq-FBYUABuWHCc0MhqABbhUJwMdIgMUG4uTlDGZhZWUAAGACQA3ozeYQBqAL6sldVh6vUlee5+0Ex4PF7A4ZZQOBCBnDikdJEQcCDhYGBzUMCJ0PLriCtrI2SDOXkA-FC1gsqqGlo6egQG6nadHkNoUGuj3a9YM9vQcFg0URicS8WVW0DSGSygUSWEY2VyzgAXPkZHkNlBNNpdLhbgVzBBLNZaIwmA4KI9PD4PgTgmkdAIJAELvMDs48UVyOUqgwagJFPUABTNHlhACq9QAlB1nO5PP5gFgANbQNTkaKxMgg1LpTLwo5QCpQADaiq1igAusjzmoQJjrjjDKKHPUKV5fGoAOTkRmIMCqTIxJk2vVI-ImfGE0rC3lKeqHaWuJ7+fBKlXANVAzW+-0MQP-boIENQY56VQxAyGk1my2wBAodA3Igkcj1B4yp5wwL-NNeqD4fizeYRMAOgA0HzEgwkQ7ryCwPDEKxIReRldNcMINfgSFQmAdxEgLdRznRdux+gAwqIxIEc0ZwxzqMT7I5254aSF6VBs5DAyP9BAX6smGhQEsUJQuIi0ZhBgcYuNBAhtAmspQCOpDkGoKxbP+BBLgoEgvKM4yTNMg4LEs8zEaMKh4RAeTZCeWzIDCwC9mI3j7AixZQKWdDlmejYGBgdgOAAZBiVzngQV4IGMd4QZUSFtomnikAQ0A0Jw3jeOOPDaH4aygjsOH4F45A-gGcAriBEbgZBCE8fU8HcjGSH6gJDpCWJElYo2Mk3vJzlKa6-gfnSYQ0FgqiBE+thMOODCIFOqEOuEU6FsByLsmBnLOS0AgAPpOQ5blvuRyxUQSgzokRExTDMyzAFsYK7ACwHHB5l7XnJkIGApFTBWVfwAl2pAQOOzEEil+hmVqLVgd+vgtRMVmZTZj4QVBLkwe07mSYJwmumkiqJcgULduOpCJFocAvAwEAzAIMKLFNowxMsa3HadUiyM43hpLekIjJJABCnB0Ld-AGAA0g+OWxSSdgCjgknItD44o75DqkMinXSd1gOZDDdgSsiNgkgaeSjMAExZEwBL8FF2KrMjkkY-t2PhN6DAgBK0gupjOhgxDaD8AKHouJwY04JBUv8AAkmgkFqBA+ACmAiA0KwuDRWIEouB646Gir+DIh6Ot0IEYiG6k0uK8iABMUCSjILguFAAB6fgQ1Z9NTswODzFSHomx6uyBBApHWIg-arBTsxjdsjIehrNBh4gAjmzgusQB6sinpJABieBMP2Kj3qBkbk8w44YKM3h0KYcNV8+LBUAARC47dkut8NcnXEAN6Y9QObBeUioVu3OMc-f143I8DUXJdl8AB12O0ghnsXWCl1VRjjrPg-z5UY8lMpWVOHIy5b8vVWCdlLdxT3D-gVt+WxkK20COKUr6talxY30AYJ0eQsrN1fg5fkn934-3jH-FQNo8aOnPvkX6UB-pBBzOg2+oQ0BnmFpDHAhcsC3gziACutkiRPwFBKSmzhqa0ygDDcB5Ab47xXkYESrMsRo3ZoAggOMfL2i6rJQmDBiakwRswSgDgKj0V2DTHAWRBYQAIaLHA3CdB8OEQIrmQxeZ5BdC6YIDAATeBwcAfB4NCGUGwewqq6ArEi34MQ0hgcDCdzljLREXjFbKx0GrVOwBtbZ0tnrZWStEQkJzAKSIdAIDIFYOgNQEpu40JkOY+xKgnGEPFpLaWss7aRJNurTWwSLZWwgPrREdA0A20NL4tAjsACM44TZmwqdbNpTSoDNKdi7IAA).
