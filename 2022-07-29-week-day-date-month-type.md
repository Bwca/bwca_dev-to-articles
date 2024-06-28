```ic-metadata
{
  "name": "Weekday - Date - Month type",
  "series": null,
  "date": "2022-07-29",
  "lastModifiedDate": "2022-07-29",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial"],
  "canonicalLink": "https://dev.to/bwca/weekday-date-month-type-26j4"
}
```

# Weekday - Date - Month type

Suppose out of the sudden you are dealing with an object, which besides well-known fields has a number of fields which are named after the day of the week, date and month, and contain some value related to that particular day. I.e. let's say you keep a record how many times you're happy to use `Typescript` instead of pure `Javascript`:

The final code is available [in the playground](https://www.typescriptlang.org/play?#code/C4TwDgpgBAIghiAzgeQGYBUAWEDqEIDWUAvFAOQCyA9gHZlQA+56ArhPU2XnY85ix3IAxAE4BLQWQDKcYJKks6AbgBQK0JChYxI0MhoR4SajWCZEJcgCk4PThTgjJDkJKssANpICCLAOaSyADGcrxkMBBBZKoa0Nq6IEaIJmYWpGTeYE5h7nbkUhBgkgByVABu0WqxUPoQWCJU-pjFYgaWAIy8AEy8AMy8ACy8AKy8AGy8AOy8ABy8AJwx4NC19Y1+mOgA7hCmIC1tpAAGACQA3gAMvJ1MXQC+56uYDU0HEHdHS5pPLxvxoJYfutNjs9m8wr0LpVqkCmv8QLVAQY1nDMDoAUwAES9dqY1TqZawBBoPCEeDACApTAAHiSAEkaFSoBAAB4UmgAEwsiGA4hofgANFAqRZWeyuVAeXy-AA+SynM5JNBYXD4AgPRUIRAMqkakUffHVIzkym0MyWIwktUmqnU2F-UGgN5CshCCAAIzIcqYltQpIINrNNPtmzRCSF8KSIu9RIRfutslNpmDyOewPhtQjYb0BijQcQMsNhONiapACVIlQRBzqeg5aQziooFAANoECAgKCtWOB5MAXQA-AAuLSqO741oUkSoOBBaAAGTEPJwYjM3jKEBEcD8EFrcrFuwlJYp5cr1b3UEbzfKm7gHg8I5oLAAtu7N2O1AB6T+SuDPsAeBAKhBLQPJQJgVBbAAEnAYBgCA6BUAAqogdTLIgQTiGAwBQUuwBViAI6Lsuq6YOut47tST6vpu9aXk2zZaPwXTzO4D5QAyqCtKuIACgx14blu94jpx3GgCodxAA).

```typescript
const howHappyToUseTypescriptHistory = {
    Thu29Jul: Infinity,
    overall: Infinity
}
```

So far good, type is inferred, but what will happen on Friday? Need to add another record, recalculate the average, inferred type changes. Is there a way to type such construction at all?

Turns out, there is. `Typescript` v4 provides superpowers to do that.

Let's start with creating type for weekdays:

```typescript
type DaysOfTheWeek = 'Mon' | 'Tue' | 'Wen' | 'Thu' | 'Fri' | 'Sat' | 'Sun';
```

Now types for months, in accordance with how many days they have (omitting February):
```typescript
type ThirtyOneDaysMonths = 'Jan' | 'Mar' | 'May' | 'Jul' | 'Aug' | 'Oct' | 'Dec';
type ThirtyDaysMonths = 'Apr' | 'Jun' | 'Sep' | 'Nov';
```

All months can have 29-30-31 days top, so that's 3 sets of types using `Typescript`'s template literals:
```typescript
type OneThroughNine = 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9;
type OneThroughTwentyNine = `${0 | 1 | 2}${OneThroughNine}`;
type OneThroughThirty = OneThroughTwentyNine | '30';
type OneThroughThirtyOne = OneThroughThirty | "31";
```
At this point you've probably guessed that we're simply going to use a generic now to glue all these strings together into our `DayDateMonth` format:

```typescript
type DayOfWeekDateMonth<DaysInMonth extends string, Months extends string> = `${DaysOfTheWeek}${DaysInMonth}${Months}`;
type DayDateMonth = DayOfWeekDateMonth<OneThroughTwentyNine, 'Feb'> | DayOfWeekDateMonth<OneThroughThirty, ThirtyDaysMonths> | DayOfWeekDateMonth<OneThroughThirtyOne, ThirtyOneDaysMonths>;
```
With the type for our strings ready, we can define a generic record to store data for different days:
```typescript
type DayDateMonthRecord<T> = {
    [key in DayDateMonth]?: T;
};
```

A mapped type may not declare its own properties or methods, so in order to add `average` or any other field, we would need to extend `DayDateMonthRecord<T>`:

```typescript
interface ListWithAverage<T> extends DayDateMonthRecord<T> {
  average: number;
}
```

Finally, we can type the initial object:
```typescript
const howHappyToUseTypescriptHistory: ListWithAverage<number> = {
    Thu29Jul: Infinity,
    overall: Infinity
}
```

Now we have strict typing with intellisense if needed :)