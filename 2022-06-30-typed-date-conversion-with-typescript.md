```ic-metadata
{
  "name": "Typed Dates Conversion with Typescript",
  "series": null,
  "date": "2022-06-30",
  "lastModifiedDate": "2022-06-30",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "generics", "conversion", "date", "moment"],
  "canonicalLink": "https://dev.to/bwca/typed-dates-conversion-with-typescript-49n7"
}
```

# Typed Dates Conversion with Typescript

Links to repos are in the footer of the article.

We can't pass objects like `Date` between frontend and backend, we pass strings (or can pass numbers). Yet, sometimes 
primitives like strings or numbers are not what frontend actually needs, so they are converted into something more
suitable, like `Date`, `Moment` or whatever object representation. In javascript it might not be a big deal, but
when you try to describe your frontend entities with typescript interfaces, you immediately run into problems. Suppose
you have the following `User` interface coming from an api:

```typescript
interface User {
  registered: string;
  name: string;
  lastLoginDate: string | null;
  details: Details;
  activities: Activity[];
}

interface Details {
  birthday: string;
  place: string;
}

interface Activity {
  place: string;
  date: string;
}
```

So there are some date fields which come as strings, but as you operate with, let's say, `Date` on the frontend for 
adding/removing time or whatever, so you set up a layer of mappers for conversion of string representation of dates into
`Date` when you receive them and then `Date` back to `string` when you need to send those back to a backend. But then
what to do with types on the interfaces? Create new ones with `Date`? Use generics? Go for unions?

For some time I had an idea that such mapper should be a single function that modifies an object of any given depth and 
does conversion of dates both on the object and in the interface, eliminating the need for additional interfaces to be
created. And this is the story how I tackled the problem.

Obviously it isn't a problem to convert dates in an object from string to some object representation, the real problem
is how to type those. From typescript generic perspective, there's no telling if a string or a number is a date, however
if union type is used, then it can be narrowed down with a generic, and that's the approach that can be used.

Therefore, any date is an ambiguous union, which can be presented by a generic type
```typescript
 type AmbiguousDate<ObjectDate, PrimitiveDate> = ObjectDate | PrimitiveDate | null;
```

in which `ObjectDate` is `Date` or any other object type and `PrimitiveDate` is `number` or `string`.

Let's update our `User` interface now and make all dates there union types with `Date`:

```typescript
interface User {
  registered: Date | string;
  name: string;
  lastLoginDate: Date | string | null;
  details: Details;
  activities: Activity[];
}

interface Details {
  birthday: Date | string;
  place: string;
}

interface Activity {
  place: string;
  date: Date | string;
}
```

We've made it so ambiguous that if used inside an application it would require constant checks to narrow the types. Yet,
this can be done using a generic, which would accept:
* the type of the object being converted;
* target date type to narrow down the union;
* object date type of the union to perform check;
* primitive date type of the union to perform checks.

It would also need to traverse down the object into its nested objects/arrays and check those too, but it should remove
the ambiguity, so we'll call this generic type `DeepNonAmbiguousDate` ('cos I suck at naming).

```typescript
type DeepNonAmbiguousDate<T, TargetDateType extends AmbiguousDate<ObjectDate, PrimitiveDate>, ObjectDate, PrimitiveDate> = {
    [K in keyof T]: T[K] extends AmbiguousDate<ObjectDate, PrimitiveDate>
        ? T[K] extends NonNullable<ObjectDate | PrimitiveDate>
            ? T[K] extends PrimitiveDate
                ? T[K]
                : TargetDateType
            : TargetDateType | null
        : T[K] extends Array<infer A>
        ? Array<DeepNonAmbiguousDate<A, TargetDateType, ObjectDate, PrimitiveDate>>
        : T[K] extends object
        ? DeepNonAmbiguousDate<T[K], TargetDateType, ObjectDate, PrimitiveDate>
        : T[K];
};
```

Now we can force all dates in a user to be of `Date` type:
```typescript
const user: DeepNonAmbiguousDate<User, Date, Date, string> = {
	activities: [{
		date: new Date(),
		place: ''
	}],
	details: {
		birthday: new Date(),
		place: ''
	},
	lastLoginDate: new Date(),
	name: '',
	registered: new Date()
}
```

Looks good, generic is powerful, but there's no actual conversion, generic itself is not enough, besides there's no link
between its 2nd and 3rd argument.

Let's think about a converter for a second. It should be able to convert from object to primitive and back, without 
the actual knowledge of which primitive or object it converts into, otherwise it becomes biased and is tied to certain
types. This means we cannot really implement it to work with any types, unless we provide means for consumers to provide
their functions for conversion. This would boil over converter to an object tree walker applying provided functions to
different fields. The only certain knowledge that we have is that primitive type for a date can be either `string` or 
`number`.

```typescript
type PrimitiveDateType = string | number;
```

So to produce our converter with a factory function, we'd need 4 functions actually:
* one to check if a property is of primitive date type;
* one to check if a property is of object date type;
* one to convert primitive date to object date;
* one to convert object date to primitive date.

Since there's 4 params, we can pack them into a payload object:

```typescript
interface RecursiveDeepDateConverterFactoryPayload<PrimitiveDate extends PrimitiveDateType, ObjectDate> {
    converterToJsDate: (d: PrimitiveDate) => ObjectDate;
    converterFromJsDate: (d: ObjectDate) => PrimitiveDate;
    checkTypeofDate: (d: unknown) => boolean;
    checkTypeofPrimitive: (d: PrimitiveDate) => boolean;
}
```

The converter we're producing can also be represented by a generic

```typescript
type RecursiveDateConverter<PrimitiveDate, ObjectDate> = <T, TargetDateType extends PrimitiveDate | ObjectDate>(
    obj: T
) => DeepNonAmbiguousDate<T, TargetDateType, ObjectDate, PrimitiveDate>;
```

So it will accept an object and a target date type for conversion in generic, while the conversion itself will be done
using the functions provided to it.

All what's left is to create the factory function:

```typescript
function createRecursiveDateConverter<PrimitiveDate extends PrimitiveDateType, ObjectDate>({
    checkTypeofDate,
    checkTypeofPrimitive,
    converterFromJsDate,
    converterToJsDate,
}: RecursiveDeepDateConverterFactoryPayload<PrimitiveDate, ObjectDate>): RecursiveDateConverter<PrimitiveDate, ObjectDate> {
    return function recursiveDateConverter<T, TargetDateType extends PrimitiveDate | ObjectDate>(
        obj: T
    ): DeepNonAmbiguousDate<T, TargetDateType, ObjectDate, PrimitiveDate> {
        for (const k in obj) {
            if (!Object.prototype.hasOwnProperty.call(obj, k)) {
                continue;
            }

            const value: unknown = obj[k];

            if (Array.isArray(value)) {
                value.map(recursiveDateConverter);
                continue;
            }

            if (typeof value === 'object') {
                if (checkTypeofDate(value)) {
                    // @ts-ignore
                    obj[k] = converterFromJsDate(value);
                    continue;
                }
                // @ts-ignore
                obj[k] = recursiveDateConverter(value);
            }

            if (checkTypeofPrimitive(<PrimitiveDate>value)) {
                // @ts-ignore
                obj[k] = converterToJsDate(value);
            }
        }
        return obj as DeepNonAmbiguousDate<T, TargetDateType, ObjectDate, PrimitiveDate>;
    };
}
```

So it creates a converter walking down the object tree, doing checks and conversions with provided functions. Which means
it is quite flexible, it doesn't make assumptions about primitive date being string or an object date being `Date`.

Let's try to use it on a `User` with string dates:
```typescript
const user: User = {
	activities: [{
		date: '1991-12-12',
		place: ''
	}],
	details: {
		birthday: '1991-12-12',
		place: ''
	},
	lastLoginDate: '1991-12-12',
	name: '',
	registered: '1991-12-12'
}
```

Creating a converter, for the sake of simplicity we assume dates are any string with 8 digits with hyphens:
```typescript
const deepDateConverter = createRecursiveDateConverter<string, Date>({
    checkTypeofDate: (v: unknown) => v instanceof Date,
    checkTypeofPrimitive: (v: string | number) => typeof v === 'string' && /^\d{4}(-\d{2}){2}$/.test(v),
    converterFromJsDate: (d: Date) => d.toISOString(),
    converterToJsDate: (d: string) => new Date(d)
});
```

Now we can convert all string dates of an object (not necessary a `User`) from strings to `Date`:
```typescript
const convertedUser = deepDateConverter<User, Date>(user);
```

And then we can convert them back if we want strings:
```typescript
const convertedUser2 = deepDateConverter<typeof convertedUser, Date>(convertedUser);
```

Pretty cool, eh? :D

The caveat here is conversion always goes from one direction to another and type is hint is set using generic, so these
are two different things: pass wrong date type to generic, get wrong hints :)

### Links
[Typescript playground](https://www.typescriptlang.org/play?jsx=0#code/C4TwDgpgBAggtgIwJYHMCuB7NBnAIgQ2AgB4B5BAKwgGNgCiAaKABQCck4lgkA3CeiAD4oAXijkqtAVAA+Ldp259pcgHZoANhoDcAKF2hIUXBAhgAchlXxk6LHkIkAKkyf5WKCHUdPw0CAAeRKoAJtiwiKiYOAJklDTejPIcXLz8joJMEgkCTGwpSulEwmIA3rpQlVAA2gDSUEiqUADWECAYAGZQTgC6AFzddT1QgcFhEbbRDkRxkokQeQqpyhkVVesA-IO1w6MQoeGWquaaGvgIGiTZUo6yyYppAoJr669QW05DI0H74-kPKyILzeII+Q2BINeAzcHi8Al8kAhIOh7k88wR0DUpyRlWhXz2B1grFY+BAxEaHQgrFgz0hWxgxNJxBMZiONii9liMFcqLhPj8WXiNyS-2WRSEtOR212P0JGCFwBx72MpgsVnZdhijmInx2PNh6IF4gVuXuYqeSrxOz0AF89AY-GbCvDHWJsMB2KoUHd1IgqfbGkRWB18NRoAAlGhoVjYR6qgQAYSsfFYQYAYqHgBhWCBmKSNBh8CFiKLnbcCX8lmWiBjBXMnlByutqMmqUGnBgAFLTCADAAUIQGpcejgAlKJhNd5npm63U1S06wMHBuwJ+4PjfWxxOnSOiDOqtQABY0ZoYzprqADgZoVTNVQYADuqnHImECAwGEu+FUB8qx9Pc8OmHPh1yHKs9wgV930-b9f10G19EMCMoxjSCk1UFMgxLCDAQWTccgyUQoB1fU0RdIwK3CEDxTuKcnj7YF5QoaFdGglVWXVSJNR7Ujul5Q1IDrQiRVw8VBHtDpb1oJArCgahWAgRxI2oaNYzwjCsKpHCCkg74xmosSKPw+iMj7JtDxPagzz8C9HAYYEAOsoCaIc2dMLbBclxXHs3MPOd2y7XyEIGFS1LjMxEwChdM2zXN80LYsaNNUzilHULUPU8VNM81gdIBcVhOFIRG2BRTgGjJopNUGS5MU1S0I06K8pcfiDWM-TfkM3S8Lok0zKVZjWPWdKOLVaxuKmWJWphcj+SEgjisWHrxNKyEOmzK8W1Ud0WgaJpmPHCzIUqJAuj7ABCKcADowCXLNkOuo98GwUhnzYDBIFTEBruofAtD7ZimGaUcjqVSFtu4dQID-E7EPB-yduAKAeH+tBeygW97yfJoxGY6pmh6e0TqqM6rwZEkfqQbAKdJPtUY0dHQbWknXgZ9HrrgfAwD7erwqajz51YUdYdZ+SrCh9HRZBeGxbJvtkM6FG0egEQ1agAByZiEg1sGxfWeWnJsyA7KIemVeZ479fWAB6G2oAAAWAbAAFpUAfRSEZJ-HCeI7atNYRdl1XRxzcZqDpetyHGilr3XkQ63Kjtx3nbdlAPYgOP1h94YxD5xrsuasOmcj+P9Dl86jZcsS+3y80MnZqC9cT5Ondd93s0zxOoBzv3mo7EOzcbkW44TmWlXKyqe8oKAXrGtlJs5bUZoE4yivmZaCqeWG7QQ-RdEDKkQzDKAAFVsCpFnFJQamgwgDcVCgd1PRQA9VHwOAMefxpX7WM53QADIYBvqoS8j9v5eh9KcA8IQvD4CQBobAAwTDAHgYgg8mZeCpAgEg2AMkeBcBANUIme8D6qCDMfaAKC0HhAssgVMR4QikmQbcOQEDf6VDAGcMMAx2G2n0IfYMoZoAwHwYQlmXDhG8I9D-GBjgWFEDuHw0h21do4CpAMc+l8yi6AAJCYIIdwHBAxqjlF0bophRABgawAIwAE47E2JdjYgATM4lxGsHLmMkTwzWGs9E2h6F42BqCEG4LMbo+hwBGHMM1vYxx7j3GeL0bonxGMNb+N0TaLx-9gBAJAZeWxDinGuKSV49+n9rHJN0dfW+VJ77WPiSUtxrj-Hw1UcjWBkVHA5SFn7RSylMroWasQdhTAGJWyrrZDol56Y3jvA+Z87EeD7XdD+MMStciOSssbCAnQaL9h4NIl+UC-TCx3IrLoKy1ZiA1uwjWUAABkjyoA2wAHoAB0QilAACw2j7C7L5pQXE2lHMCm0AASG210iDunpqOPy-5mpBx8rMh+243xQBCDCjAABJAAyqQfFMivR9gRY5fuQU0XHJ-uxVQEBHzGFDiEUcCER66A6eLQWRAQhaOpGILpYAorcu0ny8ZZl1HCz0Kor8EBroFhQH2AABv9DQWLHDhHcNARZTKrFQAhaUao-tPK8ovqwa6tT3T1JCEwY185TVUmuiEmh10okxJALa6KDrzW5PyY0U0dqeV8uugY7B2BqgAAYehOscNGiAKYQB9iQDuZNjQ1k1T2V0AQo4bRKtZRyqwu1A33z5S44igrhUB2IJcrlWlvXiuKH2Yt3qR4ysuPK4ByrVXqthbPRSUAdWXgNUar1paLUQBvlaxSNra0mrHc6sJrqkAMKYR62d9qx2+uAf6+y66g1mpcSGsRRjw1RpjUQONCak0XOmQ0UQ6s7kkpQLrXNo4gA)

[NPM package](https://www.npmjs.com/package/@merry-solutions/typed-date-converter)

[Github repo](https://github.com/Bwca/package__merry-solutions__typed-date-converter)