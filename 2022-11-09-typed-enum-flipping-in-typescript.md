```ic-metadata
{
  "name": "Typed Enum Flipping in Typescript",
  "series": null,
  "date": "2022-11-09",
  "lastModifiedDate": "2022-11-09",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial"],
  "canonicalLink": "https://dev.to/bwca/typed-enum-flipping-in-typescript-2ce9"
}
```

# Typed Enum Flipping in Typescript

Ever had a situation where you needed to swap key-value pairs of an enum in typescript and attempts to type it resulted in ugly `Record<string, string>` sort of crutch?

Turns out key remapping that came with v4.1 make it a breeze :)

Suppose we have some rhymes

```typescript
enum Rhymes {
  Money = 'bread and honey',
  Phone = 'dog and bone'
}
```

Key remapping and template literals allow us to write a generic to swap they keys for values:

```typescript
type FlippedRecord<T> = {
  [K in keyof T as `${string & T[K]}`]: K
};
```

So now we can get some type safety while creating flipped enum maps:

```typescript
const reversedEnum: FlippedRecord<typeof Rhymes> = {
  'bread and honey': 'Money',
  'dog and bone': 'Phone'
}
```

It's boombastic, check it out in [playground](https://www.typescriptlang.org/play?#code/PQKhAIEkBtoVwM4BcBOBDJBLA9gO3AGbYrhIAWApuGilgMbRVlJIAOCAXMMACYUBuAOiTZgAIwDudNMCQBPVhR4BaCrjgBbZQWiZWrTLgDmyw8vmKEdFHqTKATHQoBOcCGAAoDxaoAxXfpKAEoUdMQ8ADwAKgB84AC84ADeHuDgANoA0uCG4ADWFHLYBOBR1AjgAAYAJEnINsbgAGSlWQC6AL6VbRzgmR4dANxeaprgQWRyGhQVKWkAsniFCeAA5GIoFGg81Lg7ZEtyqwA0qeAACge4VImrPNhGuztiS6sDXmG4yOCb-BQoCCUAFF1Bpev49IoeCEwihIj5iuNJtMEHFEnM1hstjs0HtwFdCqteqtFtcjqc0ncHk9wC9rkS1pclvYWay2ey3h0gA) :)
