```ic-metadata
{
  "name": "Split Collection Based on Enum Value with Typescript",
  "series": null,
  "date": "2021-06-28",
  "lastModifiedDate": "2024-07-09",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial"],
  "canonicalLink": "https://dev.to/bwca/split-collection-based-on-enum-value-with-typescript-4g3b"
}
```

# Split Collection Based on Enum Value with Typescript

Enumerations are an awesome feature of typescript, you can create a set of distinct cases with numeric or string-based enums.

For example, if you're working on a game of cards and figure out you could actually use `enum` to define the suits:

```ts
enum Suit {
  Clubs = '♣️',
  Spades = '♠️',
  Diamonds = '♦️',
  Hearts = '♥️',
}
```

Now, if you have a collection of objects and each object in the collection has a value represented by an enum, you can actually write a pretty neat function to split it into smaller collections, each limited to a certain `enum` value. In our case we could split a deck of cards into clubs, spades, diamonds and hearts correspondingly.

For the sake of simplicity our deck will have only 4 cards, namely 4 aces, one from each suit, here's our small deck:

```ts
interface Card {
  name: string;
  suit: Suit;
}

const deck: Card[] = [
  {
    name: 'Ace',
    suit: Suit.Clubs
  },
  {
    name: 'Ace',
    suit: Suit.Diamonds
  },
  {
    name: 'Ace',
    suit: Suit.Hearts
  },
  {
    name: 'Ace',
    suit: Suit.Spades
  }
];
```

With typescript, we will be looking forward to obtaining somehow strongly typed object that would be of the following interface:

```ts
{
  '♣️': Card[],
  '♠️': Card[],
  '♦️': Card[],
  '♥️': Card[],
}
```

Basically we will have `enum` values as keys and each will hold corresponding array with cards of its type.

So, let's invent a function for producing such object. But before that we would need a generic type that can take `enum` and a value type and produce a type for the object, which would have its properties based on enum values. Let's make it as abstract as possible:

```ts
type GroupedValues<T extends PropertyKey, V> = {
  [K in T]: V
}
```

Now we're ready to create our grouping function with the following signature:

```ts
function groupItemsBy<T, P extends PropertyKey>(items: T[], groupingKey: keyof T, enu: Record<string, string>): GroupedValues<P, T[]>
```

It takes two generics, the first being the type of the items in the collection we are about to, and the second is the `enum`. As arguments, we will pass the collection, the name of the field, by which we are going to do the grouping, and the `enum` itself, which is basically a `Record<string, string>` type object.

Now what's left is to grab `enum` values, use them to make an empty object with arrays to store items from our collection, and then to iterate our collection pushing each item to the corresponding array which has the same name as the passed property value:

```ts
function groupItemsBy<T, P extends PropertyKey>(items: T[], groupingKey: keyof T, enu: Record<string, string>): GroupedValues<P, T[]> {
  const result: GroupedValues<P, T[]> = Object.values(enu).reduce((a: GroupedValues<P, T[]>, b) => {
    a[b as P] = [];
    return a;
  }, {} as GroupedValues<P, T[]>);

  items.forEach(i => {
    const key = i[groupingKey] as unknown as P;
    result[key].push(i);
  });

  return result;
}
```

Now we can easily split our deck by suit:

```ts
const { "♣️": clubs, "♦️": diamonds, "♥️": hearts, "♠️": spades } = groupItemsByProperty<Card, Suit>(deck, 'suit', Suit);
```

Doesn't look difficult at all, does it? What's cool is that we've got intellisense all the way :)

Oh, you can also check it out in the [typescript playground](https://www.typescriptlang.org/play?#code/KYOwrgtgBAymCWAXKBvAUFKBhANmARgM5QC8UA5IMZkg8H-kA0GsADgIYAmwxZ5gBmS0OYAIvBYQA9iDZcKgMzJ+jABLAWAJ0TTygUzJ5AXzRp4IRMBUAzFgGNg2VW1SMQo4AC4ohRCsMBzANyNCCIiucEh+emgWEu5QHBYA1q5YtgDaALqkUMmM6JiYjhAuFACCVvSMmAFIwYEAdLgEhIw6Ava5UPmF5CXAZW2VQbC1wqISUk0tObkdrl2lLRWB1Ug1Sqrq49nl7U4z3b25-UuINTCsHI2Yeql++pEg0ShQAEQ0T64WeER0z3JvMSLiSSEb5PbR-AAWyjUwOefD+hDOnCgOgyXhUYjATAAksYIIQAEIATwACuimCZEISADxJFRsb4hRAAPgAFLE4t9yP16IMkABKG53QhiHDAGo4MReFkfBoCiJREViiVStgA0aEOVCxXiyUsyFrDV+LWinVShHsThy-RoUxgEAWRDwCRQNEY7G4gkkskU6kAFW+xKgwAAHsYgVBSWJyWpCQBpYCE1lIYB41y+tLfV2Y7zxwmuOIJsSmKD+oPgVwAJWAkTpVPcnhAXm+9e8TL5rgA4ujMcA2AA1Fh4ThU4nfdOpJmtKBC5AqThgHADLtu3sDoeEEdjtKTsgAeXwACtq8cAG6DsCcFmgMB8mpzthgKwslksTvd8n98-D0cl7fffB8qQk6TJgLDJPgUAsMQxLpGQaR+G0c6IGAKggJBCHIt8KAolBUDLj2n7rpuv4TlamDJniNSmGIKgAKKWOCLLwEBU6YDOUAFoSGTwMkWZMDmCbpLhdpxCAYgAO5obhxIYZgc4BIuyScakNRMGAhCMfAcqXGRUBIShaHyQuiBhPolLknh76rl+G6+kGoagFIEbejGubfH2O5TsksZQIYJapK4fZoDoQA).
