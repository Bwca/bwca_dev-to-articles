```ic-metadata
{
  "name": "Caching Decorator",
  "series": {
    "name": "Decorator Memoization",
    "part": 0
  },
  "date": "2022-06-06",
  "lastModifiedDate": "2024-06-26",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "decorators"],
  "canonicalLink": "https://dev.to/bwca/caching-decorator-52e1"
}
```

# Caching Decorator

Let's imagine we've got a class that does some heavy computations, since my imagination is limited, I will suggest a class that does count (you can introduce something cooler in the comments, so I can stop embarrassing myself):

```typescript
class CountdownCalculator {
  public countDown(n: number): number {
    let count = 0;
    while (count < n) {
      count += 1;
    }
    return count;
  }
}
const countdownCalculator = new CountdownCalculator();
```

So obviously the bigger number we pass, the more time it will be working. At some point we might want to memoize its results, so it case the method receives a number it has calculated previously, it simply returns the result right away.

Let's call the method a couple of times and see how it goes:

```typescript
for (let x = 0; x++ <= 1;) {
  const start = performance.now();
  const result = countdownCalculator.countDown(500000000);
  const end = performance.now();

  console.log(`Result is: ${result}. Execution time of the first invocation: ${end - start} ms`);
}

// [LOG]: "Result is: 500000000. Execution time of the first invocation: 157.19999992847443 ms" 
// [LOG]: "Result is: 500000000. Execution time of the first invocation: 156 ms"
```

We could modify the method/class itself so it memoizes the arguments, or we could use the cool decorator feature typescript offers us and create a decorator for that:

```typescript
function memoize(
  target: unknown,
  propertyKey: string,
  descriptor: PropertyDescriptor
): void {
  const cachedResults = new Map<number, number>();

  const originalMethod = descriptor.value;

  descriptor.value = function (...args: unknown[]) {
    const num = args[0] as number;
    if (cachedResults.has(num)) {
      return cachedResults.get(num)
    }
    const result = originalMethod.apply(this, args);
    cachedResults.set(num, result);
    return result;
  };
}
```

The implementation is naive and simplified, i.e. it doesn't deal with such problems as multiple arguments or objects passed as arguments. But anyway.

Now we can decorate our goofy method with the caching decorator and see it in action (the whole code is available [in the sandbox btw](https://www.typescriptlang.org/play?#code/GYVwdgxgLglg9mABAWwKbLjAXqgFAWAChFEoBDAJwHNUoAuRcAazDgHcwAaIkgBwri9UFKAE8A0qlEMAzlAowwVbsUQATVDIgLeUOBQYAFAUJGiAIpu0xd+ogEoGAN0xrEAbx6IICOd7IQABaoagBKmiAANlAyiAC8iGCobIgAsmS8ADxgIMgARsKcibkFFAB8uPYA3ERePmB++jBUimSRqbSBcG4JGlo6ehQAdE5tIKg1hF591rbDo5Hj8YigkLAIiLhD25RUMgzMrBwA2gC69h5eJPV+OcjLuzLHAAyniGSxd6WTJCQwwJsIAFgmEItEZENAh9cHd7BdPKpfogKLQQBQkECgiFwjIojEhjQoDDcvYrogAL5km5QZFgmkJJotMBtDpQLpqIYZXiRUS4NkwGRFR7VKnA7F0iEyWjE5BFFG46IixG0qBopDyvE-CmTSlTQgQSIfWIAYTg4CganYYGNbQgUTIg0uqgAAmgMNhUF5eCA8pEYBBvGawFBzFaYQwvsJHMV8sInUjIrRA+bls8tSQ2IEYInAUGaZlEvCydc84gANQJACM6YpZJRqvRyeDWt1uqI1KbFqtNsidsNjoSSRSpvNlo4Pb7Dv0lUmRGA+k2iZpAA9U1VEMuy2XEJkq1Ui6oO3JKPTEKZ5xRkGRIKghkcZ3VfDSNdFlj5R93bfbBkN38HQxwuAAKzPKBYGgUqJYNDSqBgD0Z7CBeV43ne7APnqUEyHAiZDJEcBULgAAGOJ4ogAoMAAJO4L5QOSQyIAAosuqB2usSCwGgiBwACbKoCsMAUH4iguECbGUe4sFuAAtIgx4iOSKAyIRSptnqRBAA)):

```typescript
function memoize(
  target: unknown,
  propertyKey: string,
  descriptor: PropertyDescriptor
): void {
  const cachedResults = new Map<number, number>();

  const originalMethod = descriptor.value;

  descriptor.value = function (...args: unknown[]) {
    const num = args[0] as number;
    if (cachedResults.has(num)) {
      return cachedResults.get(num)
    }
    const result = originalMethod.apply(this, args);
    cachedResults.set(num, result);
    return result;
  };
}

class CountdownCalculator {
  @memoize
  public countDown(n: number): number {
    let count = 0;
    while (count < n) {
      count += 1;
    }
    return count;
  }
}

const countdownCalculator = new CountdownCalculator();

for (let x = 0; x++ <= 1;) {
  const start = performance.now();
  const result = countdownCalculator.countDown(500000000);
  const end = performance.now();

  console.log(`Result is: ${result}. Execution time of the first invocation: ${end - start} ms`);
}

// [LOG]: "Result is: 500000000. Execution time of the first invocation: 156.40000009536743 ms" 
// [LOG]: "Result is: 500000000. Execution time of the first invocation: 0 ms" 
```

Cool, eh? :D