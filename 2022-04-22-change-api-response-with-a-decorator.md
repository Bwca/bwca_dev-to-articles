```ic-metadata
{
  "name": "Override Method Return with a Decorator",
  "series": null,
  "date": "2022-04-22",
  "lastModifiedDate": "2022-04-22",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial"],
  "canonicalLink": "https://dev.to/bwca/override-method-return-with-a-decorator-1bii"
}
```

# Override Method Return with a Decorator

Decorators are experimental feature in typescript, so writing and relying on them might not be a good idea. Even google warns about [using them](https://google.github.io/styleguide/tsguide.html#decorators). You've been warned now.

Okay, now let's goof around with them and see how they can be used to modify API responses. For the purpose of this article, we will be using [Star Wars api](https://swapi.dev/) and [Angular](https://angular.io/), it's already using a pile of decorators on its own, so...

Let's add a service that would request a hero from the api by id, take a quick glance at a response and convert it to an interface, it looks something like this:

```typescript
// src/app/models/hero.model.ts
interface Hero {
  name: string;
  height: string;
  mass: string;
  hair_color: string;
  skin_color: string;
  eye_color: string;
  birth_year: string;
  gender: string;
  homeworld: string;
  films: string[];
  species: string[];
  vehicles: string[];
  starships: string[];
  created: string;
  edited: string;
  url: string;
}
```

So the service would look something like this:
```typescript
// src/app/services/star-wars-api/star-wars-api.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

import { Hero } from 'src/app/models/hero.model';

@Injectable({
  providedIn: 'root',
})
export class StarWarsApiService {
  private readonly api = 'https://swapi.dev/api';

  constructor(private http: HttpClient) {}

  public getHero(id: number): Observable<Hero> {
    const url = `${this.api}/people/${id}`;
    return this.http.get<Hero>(url);
  }
}
```
Looks good, but let's imagine that for some reason we want to have type `Date` in `created` and `edited` fields.

Let's build a function that can traverse an object and convert strings to dates:

```typescript
// src/app/utils/convert-string-to-date-recursively.util.ts
export function convertStringToDateRecursively<T>(
  obj: any,
  checkDate: (s: string) => boolean
): T {
  for (const k in obj) {
    if (!Object.prototype.hasOwnProperty.call(obj, k)) {
      continue;
    }

    const value = obj[k];

    if (Array.isArray(value)) {
      value.map((el) => convertStringToDateRecursively(el, checkDate));
      continue;
    }

    if (typeof value === 'string' && checkDate(value)) {
      obj[k] = new Date(value) as any;
      continue;
    }

    if (typeof value === 'object') {
      obj[k] = convertStringToDateRecursively(value, checkDate);
    }
  }
  return obj;
}
```

So we accept any object, iterate over its fields and convert any `string`, which is verified to be a date string by `checkDate` to `Date`. Providing a separate function to check if a string is a date provides better separation of concerns, so our converter deals only with conversion.

This function could be used in a pipe to convert `Hero` to another interface that has `created` and `edited` as `Date` types, but we're risky, so we'll use decorator to do the conversion on the fly and just directly change the types of those fields in the `Hero` interface. Not a good idea for an enterprise application, but for pet-project or goofing with decorators, it'll do.

A method decorator is basically a function that accepts the `target`, `propertyKey`, and `descriptor` as argument and then can perform anything with them, so it can be used to override the behavior of the existing class method. For our purpose we are going to pipe into the `Observable` returned by the original method and `map` it to the `convertStringToDateRecursively` function we have created, in order to convert string dates to `Date` type. To determine whether a string is a date I'll use a function from `iso-datestring-validator` package, it simply checks if a string is valid ISO-8601 date and returns boolean, so it fits our needs.

```typescript
// src/app/decorators/map-strings-to-dates.decorator.ts
import { map, Observable } from 'rxjs';

import { isValidISODateString } from 'iso-datestring-validator';

import { convertStringToDateRecursively } from '../utils/convert-string-to-date-recursively.util';

export function mapStringsToDates(
  target: unknown,
  propertyKey: string,
  descriptor: PropertyDescriptor
): void {
  const originalMethod = descriptor.value;
  descriptor.value = function (...args: unknown[]) {
    return (<Observable<unknown>>originalMethod.apply(this, args)).pipe(
      map((r) => convertStringToDateRecursively(r, isValidISODateString))
    );
  };
}
```

Pretty simply, how what's left is to decorate our method with it:

```typescript
// src/app/services/star-wars-api/star-wars-api.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

import { Hero } from 'src/app/models/hero.model';
import { mapStringsToDates } from 'src/app/decorators/map-strings-to-dates.decorator';

@Injectable({
  providedIn: 'root',
})
export class StarWarsApiService {
  private readonly api = 'https://swapi.dev/api';

  constructor(private http: HttpClient) {}

  @mapStringsToDates
  public getHero(id: number): Observable<Hero> {
    const url = `${this.api}/people/${id}`;
    return this.http.get<Hero>(url);
  }
}
```

Now we can update the `Hero` interface and put `Date` as the type for `created` and `edited` fields.

That's about it, now you've got the fire to play with :D

P.S. [repo with code](https://github.com/Bwca/demo__override-method-return-with-decorator) :)