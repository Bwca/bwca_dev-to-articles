```ic-metadata
{
  "name": "Typed Filtering with Generics in Typescript",
  "series": null,
  "date": "2022-12-01",
  "lastModifiedDate": "2022-12-01",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "generics"],
  "canonicalLink": "https://dev.to/bwca/typed-filtering-with-generics-in-typescript-204f"
}
```

# Typed Filtering with Generics in Typescript

// NOTE: all links to source code can be found at the end of the article.

Imagine you have an array of numbers:

```typescript
const arrayOfNumbers: number[] = [1, 2, 3, 4, 5, 6];
```

What should be the type of the array you get when you filter it? Let's say you filter out all numbers that do not equal to `2`:

```typescript
const filteredArray = arrayOfNumbers.filter((n) => n === 2);
```

The resulting inferred type is `Array<number>`, while realistically it can be narrowed to `Array<2>`, this narrowing of resulting type is what the article is about.

Similarly, provided you have an array of users:

```typescript
interface User {
  details: {
    name: string;
  };
}

const users: User[] = [
  {
    details: {
      name: "Bob",
    },
  },
  {
    details: {
      name: "non Bob",
    },
  },
];

const arraysOfBobs = users.filter((user) => user.details.name === "Bob");
```

We could argue the resulting array is a subtype of `Array<User>`, where the type of `name` is no longer `string`, but literally `Bob`:

```typescript
interface Bob {
  details: {
    name: "Bob";
  };
}
```

Let us see how we can perform this type narrowing with the amazing features of Typescript 4.

In order to create a custom filter that narrow down resulting array types, we would need to write 3 powerful generics:

- first and foremost, a generic capable of generating all permutations of all possible property paths in an object, so we can perform filtering of objects with nested objects;
- then a generic which can extract the type of a property in an object by the path produced by the first generic;
- and finally, a generic that can swap field types of a property represented by a string path provided by the first generic.

I wrote [an article](https://dev.to/bwca/create-object-property-string-path-generator-with-typescript-13e3) how to write a property path generator with generics earlier. But it is too powerful to have a derived types using it, since it is infinite and makes typescript compiler go nuts. To nerf it down a little, a limit can be set how far the generic should go down the nested paths. Let's do `99` as default and leave room to pass a number.

More about how to create number limits with generics can be found in an [article](https://dev.to/tylim88/typescript-numeric-range-type-15a5) by Acid Coder.

So thus we get

```typescript
type DEFAULT_DEPTH_LEVEL = 99;

type Primitive = string | number | bigint | boolean | undefined | symbol;

type PropertyPath<
  TYPE,
  DEPTH extends number = DEFAULT_DEPTH_LEVEL,
  LEVEL extends number[] = [],
  PREFIX = ""
> = {
  [KEY in keyof TYPE]: LEVEL["length"] extends DEPTH
    ? never
    : KEY extends "valueOf" | "toString"
    ? never
    : TYPE[KEY] extends Primitive | Array<unknown>
    ? `${string & PREFIX}${string & KEY}`
    :
        | `${string & PREFIX}${string & KEY}`
        | PropertyPath<
            TYPE[KEY],
            DEPTH,
            [1, ...LEVEL],
            `${string & PREFIX}${string & KEY}.`
          >;
}[keyof TYPE];
```

So now we have a property path generic that generates all possible paths within an object.

```typescript
type UserPaths = PropertyPath<User>; // 'details' | 'details.name'
```

Now we are ready to create a generic to probe objects and extract the type of a property represented by a path generated from our `PropertyPath` generic:

```typescript
type DeepNested<
  TYPE,
  DEPTH extends number,
  PATH extends PropertyPath<TYPE, DEPTH>,
  LEVEL extends number[] = []
> = {
  [KEY in keyof TYPE]: LEVEL["length"] extends DEPTH
    ? never
    : PATH extends `${string & KEY}.${infer REMAINING_PATH}`
    ? // @ts-ignore
      DeepNested<TYPE[KEY], DEPTH, REMAINING_PATH, [1, ...LEVEL]>
    : KEY extends PATH
    ? TYPE[KEY]
    : never;
}[keyof TYPE];
```

Note how we unpack the path by splitting it layer by layer, until we either reach the limit of depth or arrive at the sought property:

```typescript
type NameType = DeepNested<User, 2, "details.name">; // that's a string
```

Generic for substituting a type of a property represented by a path follows the similar approach of having depth, level and a new type, which the given property should get:

```typescript
type DeepSubstituted<
  TYPE,
  DEPTH extends number,
  PATH extends PropertyPath<TYPE, DEPTH>,
  SUBSTITUTION,
  LEVEL extends number[] = []
> = {
  [KEY in keyof TYPE]: LEVEL["length"] extends DEPTH
    ? never
    : PATH extends `${string & KEY}.${infer REMAINING_PATH}`
    ? DeepSubstituted<
        TYPE[KEY],
        DEPTH,
        // @ts-ignore
        REMAINING_PATH,
        SUBSTITUTION,
        [1, ...LEVEL]
      >
    : KEY extends PATH
    ? SUBSTITUTION
    : TYPE[KEY];
};
```

It is time for the final touch: create a filter function which performs swapping types for nested fields. Ideally we would need to implement a function which takes a predicate add then throws it into the filter function of the array, but let's simplify things a bit and instead implement filtering by a value. That means we take an array of objects, path to the property by which we filter and the value, by which we will perform the filtering, and which will become the new type of the property represented by the path.

Also, for the sake of simplicity let's assume we do not have optional fields and all paths are guaranteed.

Given these conditions we can create a function for extracting a value from a given property path, which would look the following way:

```typescript
function getNestedValue<TYPE, DEPTH extends number>(
  a: TYPE,
  p: PropertyPath<TYPE, DEPTH>
): unknown {
  return (<string>p)
    .split(".")
    .reduce<unknown>(
      (c: Record<string, unknown> | unknown, p: string) =>
        (<Record<string, unknown>>c)[p],
      a
    );
}
```

The typed filter, which we're aiming for would use it for value extraction would then take full advantage of all our generics and extractor function. We will be accepting the type of object in the array, a path to one of its properties, a substitution value, which has to be a narrowed type of the property represented by the given path, and an optional parameter of depth for the typescript compiler not to go insane over potentially infinite depth:

```typescript
function typeFilter<
  TYPE,
  PATH extends PropertyPath<TYPE, DEPTH>,
  SUBSTITUTION extends DeepNested<TYPE, DEPTH, PATH>,
  DEPTH extends number = DEFAULT_DEPTH_LEVEL
>(
  arr: TYPE[],
  field: PATH,
  compareWith: SUBSTITUTION
): Array<DeepSubstituted<TYPE, DEPTH, PATH, SUBSTITUTION>> {
  return arr.filter(
    (a: TYPE) => getNestedValue<TYPE, DEPTH>(a, field) === compareWith
  ) as Array<DeepSubstituted<TYPE, DEPTH, PATH, SUBSTITUTION>>;
}
```

Now getting an array of users named Bob is as easy as:

```typescript
const arrayOfBobs = typeFilter<User, "details.name", "Bob">(
  users,
  "details.name",
  "Bob"
);
arrayOfBobs[0].details.name; // name is of narrowed type 'Bob' now, not the broader type string
```

Yet, that is a quite complex filter for infinitely deep objects, but what if we want to filter only arrays with primitives? It is even easier in this case because we don't need to leverage the power of recursion in generics:

```typescript
function primitiveTypeFilter<TYPE extends Primitive, SUBSTITUTION extends TYPE>(
  arr: TYPE[],
  compareWith: SUBSTITUTION
): SUBSTITUTION[] {
  return arr.filter((a) => a === compareWith) as SUBSTITUTION[];
}
```

Now to get an array of, let's say, 2s we could simply pass the type and value to the generic:

```typescript
const arrayOf2s = primitiveTypeFilter<number, 2>([1, 2, 3, 4, 5], 2); // get Array<2> type
```

Isn't it amazing how powerful typescript generics are? It is incredible what amazing possibilities for typing they provide.

The source code for this article is available in the [monorepo](https://github.com/Bwca/monorepo_merry-solutions_generics).

The filters as an npm package are available [on npm](https://www.npmjs.com/package/@merry-solutions/type-filter).
