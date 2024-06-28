```ic-metadata
{
  "name": "Deep Readonly Generic in Typescript",
  "series": null,
  "date": "2023-09-16",
  "lastModifiedDate": "2024-06-26",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial"],
  "canonicalLink": "https://dev.to/bwca/deep-readonly-generic-in-typescript-4b04"
}
```

# Deep Readonly Generic in Typescript

TL/DR: the best way is found [here](https://dev.to/grief/comment/2calg), which is:
```typescript
type DeepReadonly<T> = {
  readonly [K in keyof T]: DeepReadonly<T[K]>;
};
```

The image is how NightCafe sees _deep readonly type_.
The link to the playground for this article is down below.

Immutability could be useful for certain cases when building applications, and today we will take a look how to enforce immutability using Typescript.

Normally to get an immutable object, it would need to be frozen with the appropriate JavaScript function, but that would only freeze a single nesting level of it and would not affect anything nested inside, whether in its objects inside properties or arrays of objects.

A possible solution would be to iterate over all properties recursively, freezing all objects and repeating the process for any nested arrays of objects.

However, Typescript provides means that can potentially eliminate this need if used wisely.

Suppose we have the following interfaces representing a family:
```typescript
interface Person {
  firstName: string;
  lastName: string;
  age: number;
  children?: Person[];
}

interface Family {
  parents: Person[];
  grandparents?: {
    paternal: Person[];
    maternal: Person[];
  };
}
```

And a corresponding variable representing the family instance:

```typescript
const family: Family = {
  parents: [
    { firstName: "John", lastName: "Doe", age: 40 },
    { firstName: "Jane", lastName: "Doe", age: 38 },
  ],
  grandparents: {
    paternal: [
      { firstName: "PaternalGrandfather", lastName: "Doe", age: 70 },
      { firstName: "PaternalGrandmother", lastName: "Doe", age: 68 },
    ],
    maternal: [
      { firstName: "MaternalGrandfather", lastName: "Smith", age: 75 },
      { firstName: "MaternalGrandmother", lastName: "Smith", age: 72 },
    ],
  }
};
```

In order to provide means of immutability, we could write a generic, which recursively traverses and interface field and marks everything it encounters as `Readonly`, essentially mimicking the freezing, but eliminating the fuss of the actual deep freeze.

```typescript
type DeepReadonly<T> = Readonly<{
  [K in keyof T]: 
    // Is it a primitive? Then make it readonly
    T[K] extends (number | string | symbol) ? Readonly<T[K]> 
    // Is it an array of items? Then make the array readonly and the item as well
    : T[K] extends Array<infer A> ? Readonly<Array<DeepReadonly<A>>> 
    // It is some other object, make it readonly as well
    : DeepReadonly<T[K]>;
}>
```

There, now we can create objects, which can be real constants:

```typescript
const family2: DeepReadonly<Family> = {
  parents: [
    { firstName: "John", lastName: "Doe", age: 40 },
    { firstName: "Jane", lastName: "Doe", age: 38 },
  ],
  grandparents: {
    paternal: [
      { firstName: "PaternalGrandfather", lastName: "Doe", age: 70 },
      { firstName: "PaternalGrandmother", lastName: "Doe", age: 68 },
    ],
    maternal: [
      { firstName: "MaternalGrandfather", lastName: "Smith", age: 75 },
      { firstName: "MaternalGrandmother", lastName: "Smith", age: 72 },
    ],
  }
};
```

Any changes to the object typed with the generic are going to be stopped by the compiler:

```typescript
family.parents = []; // ok
family2.parents = []; // error

family.parents[0].age = 1; // ok
family2.parents[0].age = 1; // error

// ok
family.parents.push({
  age: 40,
  firstName: 'Joseph',
  lastName: 'Doe'
});

// error
family2.parents.push({
  age: 40,
  firstName: 'Joseph',
  lastName: 'Doe'
});
```

All benefits from `Object.freeze` without a single freeze, cool, eh? At this point you are probably wondering how to shoot yourself in the foot with it, there should be a way.

And there is a way indeed, shooting in the foot is possible using reference types:

```typescript
const family3: DeepReadonly<Family> = family;
```

As you remember, `family` is just `Family`, so any changes to it would mutate `family3`, even though it is deep readonly.

This is the way things are.

Hope you enjoyed the article as much as I did while researching this :)

The [playground](https://www.typescriptlang.org/play#code/JYOwLgpgTgZghgYwgAgArQM4HsTIN4BQyyMwUGYAcnALYQBcyFUoA5gNxHIA2cF1dRszadicVg2QgArjQBG0UcgQALYNwAmUCCAD8jdORwBtALqcAvgQKhIsRCgBitdQE98XAA5xt4DAcwTcy5WKDgQDW9fMAx9D2JibzsQOG4AoxAzJWIaOGTU9OxM4OILS2sEHAoSF25XRmcaN2QAXnjkKJ0YxmMuYjwSMn5aSQAiACksFRBRgBoePioRxlGAESwIOeRxSQAWAAZkC1m+-EHyJcFkCfDN+d5hq7WNrZ3GAGYADiOT4lNf5ChcKRHxdfztRJ5aApNLIXoJBIDUgXARjVBQqAwgDiYQi8DAKmgWwelzG6zu2wkjAA7Idjqd+udHmiMdjcRoaFgCUT7otUStya8qcgAGzfekI-6nXL5WHwhFnZHMlYAWVZqRxwPxhKgxL5y2uAGUmgShZJqQBWH4MxVDUmq9XcTURTnc3W85VGk0qM00gBM1slAKsZWsYFcnhQqwgEE8ACUIHANDg6gAeAAqAD5WsgE0mU65U4RiMYANLIUDIADWEFcWBgyHTpkYpwA9K3kABJDAVsDbDosE3AABuEF0jcJuFyNd7yG0+ZAdVO6bLpmQEAAHpAIj2ABQyeTQZAAHyYYBYIFYJ6YrnkWG4AEpkOO88nF4WV6XTNm2x3u7Pwm2KAwncetewgGhYgnHRkGnFBuSAkC50TN86m2CJkAQ4BIBobYewAdwgbhuFORhPzXTdtw0HsAEFgLgQtQBgI8aOzF8UILVM6JA1No1jV9ONYzMfwRdsuz7YAe2wOhkC5HVZLkAArCAEDAeY4NnedUPcPhkEI4jSOQPj4w498M1XTNLEzCoqj7eAmjqP1GGMgSzMaNxszaYsOlBPwelOJE7X5a5JmmPVPWeCk3mQA5A0RJl7RC25wsSyLfWQL44qlYggQiTo-IhHzZX8hVbRRA1RnRWVnQ0bUeQWCLBXmaLaTihFAvKp4quhDV2VdHUUuCtLmuFMU2uQbKEhlHq5RtDqIrVar2Tq90GtS41sJ9EbzStCUFXm1LFpmmr+vqkkho201tv9cbJpDTgCHstwADp8piHMsmQMSsCrR7alcP1Xt8962k+sToCgLAoGsJ66iB6IMGMfZTGenYcwARnYL6Ox+v6HIB+GwSRlG0baTHsfXYCoesb7fth1xCb8V7pAwFRd286KDgBJVEoAckmDBYxUXmAXOg1efJXmCAsB8HvBqnofpwG3owZnWfZrhOf2bmgvFgWhZFrgxauCWNilmWHsqEBqnp95nJjEyFzTdy6k8mp8fYIA).

P.S. if someone knows how to pull this trick with generics in JSDoc.
P.P.S. [JSDoc conversion](https://www.typescriptlang.org/play?filetype=js#code/PQKhCgAIUgBAXAngBwKYBNUDNIAVUBOAzgPYB2UMsyBJaBSkA3kfAQJZkDmAvpFu2LwAcgEMAtqkpwadQoxZtOvSABtRrMZOnVa9BWQCu4gEaE+orlOgy98xM3zFyAbQC6fFwGMAFu1XoBKhkbpTA4OCgEDYIKBjYkABiEv6IOrL6DkxOpGTufMiiQWTwROl2DFlM0pCQhfCEZKKqAFx4hLnuNZDiog0ETa3tznmhNjx8XASiZOiFxaVhEVFwSGjMyeKpfCDhXuSs-CmqDgC8zFB1RcGlbS6XtUz8gpoSqG0ARABSJD5kHwAaNQaERvT4AERIqEBkEs70gABYAAyQHgAh7MZ5CLTw74zaFA9SvSQQqEwuFtADMAA5Uejam56ZApjM5tcSkQ2tVarV6o1mncMY8scTcbg+vzVABxaazLB9HyEGFE0EkyAfSEE2FWNoAdhRaKFmIE2LB6vF-UGMtZ4hI8EVBGVIJxpK1FMgADZaYaeZBGRjepaBZB7r7hSbRZ8ALISgbNa1yhVKwnOs0fADKW3t5J1kF1AFY6UanhHVbiY0HpbL0Lb7cngZH1Zn2Nmge7dQAmIu+-21HjgHgAbmWYB0DXEyHUDUgABUx3FMDgmAAlVCidDkE4AHm5MFqLgA0pBOJAANaoRAkHAztxtbq1GeHtyQVAADwasyIkAAFEZTIRIAAH0gVgOG4ICQMQUwSFUABKe8eQAfkgVd103RAt0fA83AAPkgBDajaLDnzfD90C-ABBAhpgwzgsAAiicIIpCULXDcyG3KiaK3cFUFQZBUPYzicJE-CbDDHk2l4-jBPQzCnyY8YcL4aSBLY9ClkiMBVjiZhVNkjiMM2VJlOgPYDngI4thOLtzm5K4Fk5EMMRLF4y0+H4-idRsNTJNtc2RbseVc001TxMgtRVF11U1HN4RpILe2Zat5huJz7N5WNBkFCSQp8i1JQTdB5TrR0Ux82L-PhfUgt9PL3PNLL42rWsHW8hrfLdXMvVqv0mVqQNJRysN6uij4K0K6sSra8qOubVttWqwsfRGkUOomuMqxtO0ZobOasx8OK9S7FaGSZfshwieVrMQAA6VKOUgc53EHSBgGASASFPcBrtSDt7vZUonpDNxXvel9qJIAgruOO6HtKFwkTcW64WBgBGMGPq+n7Yf++GiER5HUfODG3o+whaGhyIse+36TgBxz7sMIgfG-ez3WRJlS2igByH4iH4nweaZKKzR5zUeYHWDh2piHKZxm68cBogmZZtnLg5pEubc3n+cF4XLlFtVxahSWeGlkcqDWVA9L4tS0MMrdjJOUzdnAfYyEOOnEEpYHvcHIA) by [@artxe2](https://dev.to/artxe2) so I don't lose it.
