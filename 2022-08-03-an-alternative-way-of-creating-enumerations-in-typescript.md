```ic-metadata
{
  "name": "An Alternative Way of Creating Enumerations in Typescript",
  "series": null,
  "date": "2022-08-03",
  "lastModifiedDate": "2022-08-03",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial"],
  "canonicalLink": "https://dev.to/bwca/an-alternative-way-of-creating-enumerations-in-typescript-579k"
}
```

# An Alternative Way of Creating Enumerations in Typescript

(the final code of this article is available [in the playground](https://www.typescriptlang.org/play?target=99&jsx=0#code/AQUwdgrgtsDKEEsAuBnAopGBvAUMYAwgDYQBGKwAvMAOSDGZIPB-NANHnAA4CGAJiBdTUAGZE1b4AIgk5QA9mG79agMzIRbABIhOAJ1RVagUzIVAXxwB6AFQ4EYJCE0AzTgGMQhLd2C58YKSABcwFCRNKwBzAG42FEQkf3hkdEwI4xxHWUDgRzd-AjcqT2BvKD9aAEFnFkjo-yEmHGNgMxMUtKQ4AFUASQAVWF0AHlSwQIA+fOIyFGrGCvxYLl5J2mEZ4AkpWXlq5RX1LVRqgwrDCJwkAE92FzjW6gAKc8vpO3bu2ABKAG0AaxAzp+AHiB-rBOj0ALonQbpAAWGm0i2uuhBrwAdLt4REoa0UPM+LForoajQTlYbPYnC4cpp3PlCsVAsEwOFKsh8cgkjhmkNWplqdlcpR8l4fNUyiAVvgoqylrVjEA).)

Today we will take a look at `enum`, an awesome feature of `typescript` and also at some problem we can run into while using them and a way to overcome it.

Let's assume we're building some frontend application communicating with API that serves something that can make use of enums, let it be cards for example. There are 4 suits which can be represented by an `enum`:

```typescript
enum SuitsEnum {
  Clubs = '♣️',
  Spades = '♠️',
  Diamonds = '♦️',
  Hearts = '♥️',
}
```

And there's an api which serves a random card, which has a name and a suit:

```typescript
interface Card {
  name: string;
  suit: SuitsEnum;
}
```

Looks neat. Then we decide to write a unit test and not to invent test data, we decide to generate it by sending request to our API. We perform the request to imaginary `api/the-only-card-i-need-is` and get a random card:
```
{
    name: 'Ace', 
    suit: '♠️'
}
```

Good, let's put it into a variable in our test:
```typescript
const card: Card ={
  name: 'Ace',
  suit: '♠️'
}
```

As we try to do it, we immediately run into a problem `♠` is not assignable to type `SuitsEnum`, though it is the exact value which lies in `SuitsEnum.Spades`.

Not good, of course we could `@ts-ignore` it, force-case `as unknown as SuitsEnum` or substitute  `♠` with `SuitsEnum.Spades`. Yet, all these are ugly solutions and workarounds.

What could be done then? We could turn `enum` into `'♣️' | '♠️' | '♦️' | '♥️'`, but that would force us into losing some flexibility which `enum` provides.

Essentially the problem here is that `enum` combines both type and value. Yet, we could create an alternative to enum with values and type separated, which could be used to fit our card needs.

We will start with `as const` assertion to create an enumeration of card suits:
```typescript
const SUITS = <const>{
  Clubs: '♣️',
  Spades: '♠️',
  Diamonds: '♦️',
  Hearts: '♥️',
};
```

These are basically the value part of our `enum`. With them, we can build the `type`:
```typescript
type Suit = (typeof SUITS)[keyof typeof SUITS];
```

Now, anything typed with the new type `Suit` can consume values from `SUITS` object or plain strings which correspond to those values, so it's more flexible than `enum` in our case, check this out:
```typescript
const hearts: Suit = SUITS.Hearts; // ok
const spades: Suit = '♠️'; // ok
```

Which means we can use this new type for typing stuff coming from our API:
```typescript
interface Card {
  name: string;
  suit: Suit;
}

const card: Card ={
    name: 'Ace',
    suit: '♠️'
}
```

No dirty tricks, just language superpowers :)