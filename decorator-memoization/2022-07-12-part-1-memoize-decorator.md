```ic-metadata
{
  "name": "Memoize Decorator",
  "series": {
    "name": "Decorator Memoization",
    "part": 1
  },
  "date": "2022-07-12",
  "lastModifiedDate": "2022-07-12",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "decorators"],
  "canonicalLink": "https://dev.to/bwca/memoize-decorator-5aoo"
}
```

# Memoize Decorator

The caching decorator from the previous article was fine, but there was a problem with it: it was too specific, it assumed only one primitive argument for the decorated method, so it's abilities were limited. It also had no mechanism of clearing the cache, so results would be stored until the application is torn down.

Let's think of a better solution, that would be more flexible. Something that:
* does not make assumptions about the key used for storing memoized result;
* has a mechanism for clearing the cache;
* can have debugging capabilities;
* can use either Map or Weakmap for storage.

Since in `js` we can pass function as arguments, it allows us to have great flexibility: we can pass a function to extract unique id from arguments, which we later can use as a key to memoize a result, instead of using an argument as a key, which was the case with the previous decorator.

Hence, the payload for our new memoize decorator can be summed up in the following interface:

```typescript
// ./models/memoize-payload.model.ts
export interface MemoizePayload {
  // The function that will determine a unique id for the provided arguments set, determined by used
  extractUniqueId: (...args: any[]) => any;
  // A flag to use WeakMap
  doUseWeakMap?: boolean;
  // If regular map is used, you can set timeout to clear its contents, optional
  clearCacheTimeout?: number;
  // For debug purposes you can pass an exta function for logging all actions
  debugReporter?: (message: string, state?: Map<any, unknown> | WeakMap<object, unknown> | unknown) => void;
}
```

Since now we're passing arguments to the decorator, it will no longer be a mere function, but a factory one, producing a decorator. The decorator function itself can be typed as:
```typescript
// ./models/decorator.model.ts
export type MemoizeDecorator = (target: unknown, propertyKey: string, descriptor: PropertyDescriptor) => void;
```

Let's assume two use cases:
* with timed cache clearing, this version doesn't need the boolean flag `doUseWeakMap`;
* without cache clearing with optional WeakMap, this version does not need `clearCacheTimeout` argument.

Bearing that in mind, using function overload in typescript we can produce two signatures for the factory:

```typescript
export function memoize(args: Omit<MemoizePayload, 'doUseWeakMap'>): MemoizeDecorator;
export function memoize(args: Omit<MemoizePayload, 'clearCacheTimeout'>): MemoizeDecorator;
```

Now for the implementation of the memoize decorator factory itself, with some debug messages passed to the optional `debugReporter`:

```typescript
// ./memoize.decorator.ts
import { MemoizeDecorator } from './models/decorator.model';
import { MemoizePayload } from './models/memoize-payload.model';

export function memoize(args: Omit<MemoizePayload, 'doUseWeakMap'>): MemoizeDecorator;
export function memoize(args: Omit<MemoizePayload, 'clearCacheTimeout'>): MemoizeDecorator;

export function memoize({ extractUniqueId, clearCacheTimeout, doUseWeakMap, debugReporter }: MemoizePayload): MemoizeDecorator {
  return (target: unknown, propertyKey: string, descriptor: PropertyDescriptor): void => {
    let cacheTeardownTimer: ReturnType<typeof setTimeout>;

    let cache = initCache(doUseWeakMap);

    const startTeardownTimeout = !clearCacheTimeout
      ? null
      : () => {
          if (cacheTeardownTimer) {
            debugReporter?.('Clearing the cache timeout timer');
            clearTimeout(cacheTeardownTimer);
          }
          debugReporter?.(`Cache to be cleared in ${clearCacheTimeout}ms`);
          cacheTeardownTimer = setTimeout(() => {
            debugReporter?.('Clearing the current cache of', cache);
            cache = initCache(doUseWeakMap);
            debugReporter?.('Cache cleared: ', cache);
          }, clearCacheTimeout);
        };

    const originalMethod = descriptor.value;

    descriptor.value = function (...args: unknown[]) {
      startTeardownTimeout?.();

      const uniqueId: any = extractUniqueId(...args);
      debugReporter?.('Looking for a value with unique id of ', uniqueId);

      if (cache.has(uniqueId)) {
        const cachedResult = cache.get(uniqueId);
        debugReporter?.('Returning cached result', cachedResult);
        return cachedResult;
      }

      debugReporter?.('No cached result found');
      const result = originalMethod.apply(this, args);

      debugReporter?.('Storing a new entry in cache: ', { uniqueId, result });
      cache.set(uniqueId, result);
      debugReporter?.('Cache updated', cache);

      return result;
    };
  };
}

function initCache(doUseWeakMap?: boolean) {
  return doUseWeakMap ? new WeakMap<object, unknown>() : new Map<any, unknown>();
}
```

Looks good, but better test it. We can use `jest` for that, create a class with a method that does some calculations, running it several times and checking how much times does it take to run a decorated methods vs non-decorated.

First let's create a helper function to check the time:

```typescript
function checkRuntime<T extends (...args: any) => any, C>(fun: T, iterations: number, countdown: C): number {
  const start = performance.now();
  for (let x = 0; x++ <= iterations; ) {
    fun(countdown);
  }
  const end = performance.now();
  return end - start;
}
```

It basically runs a function a number of times and tells how long it took, nothing special.

Since I have poor imagination, I will use a similar countdown class I used in the previous article:

```typescript
interface CalculationPayload {
  id: number;
  n: number;
}

class ObjectCountdownCalculator {
  @memoize({
    extractUniqueId: (a) => a.id,
  })
  public static decoratedCountDownWithMap(a: CalculationPayload): number {
    return ObjectCountdownCalculator.countdown(a);
  }

  @memoize({
    extractUniqueId: (a) => a,
    doUseWeakMap: true,
  })
  public static decoratedCountdownWithWeakMap(a: CalculationPayload): number {
    return ObjectCountdownCalculator.countdown(a);
  }

  public static nonDecoratedCountDown(a: CalculationPayload): number {
    return ObjectCountdownCalculator.countdown(a);
  }

  private static countdown({ n }: CalculationPayload): number {
    let count = 0;
    while (count < n) {
      count += 1;
    }
    return count;
  }
}
```

So what's going on here? There's a private method doing all the work and 3 public ones calling it, two of them are decorated. With 3 public methods invoking the same internal one, we can run them and compare results to see how fast our memoization decorator is compared to regular method call.

Let's assume it's 5 times faster if there are 5 iteration calls:
```typescript
it('The decorated version of the countdown with regular map should be at least 5 times faster', () => {
  // Arrange
  const countdown: CalculationPayload = {
    id: 1,
    n: 500000000,
  };
  const iterations = 5;

  // Act
  const decoratedVersionTime = checkRuntime(ObjectCountdownCalculator.decoratedCountDownWithMap, iterations, countdown);
  const nonDecoratedVersionTime = checkRuntime(ObjectCountdownCalculator.nonDecoratedCountDown, iterations, countdown);

  // Assert
  expect(Math.floor(nonDecoratedVersionTime / decoratedVersionTime)).toBeGreaterThanOrEqual(5);
});
```

And it actually is that fast. What's interesting, is that the `WeakMap` version turned out to be only 4 times faster. I wonder why though.

Anyway, that's about it.

You can find the code as 
* [an npm package here](https://www.npmjs.com/package/@merry-solutions/memoize-decorator);
* [as source code there](https://github.com/Bwca/package__merry-solutions__memoize-decorator);
* or [in the playground](https://www.typescriptlang.org/play?jsx=0#code/JYOwLgpgTgZghgYwgAgLIQLYHtgC8IAKcAngDZZwAmyA3gFDLID0TyAKgBYowCuICYYFhDIwHOGGQB3YKVLJKESFAygUcZH2ABHHimDUYWKKK7IADlCwA3AxGpwoAcx4YI4AM7IPSgDQKlaFUQe2QAI2JNH0oGZAgADzAoRDAAVRAdPQBJSgAuZAAKADoSxycPfLgQYgBtAF0ASmQAXgA+ZCriAG5YlmQAQWQYUjgnUSwolAB1CDgAa1Q4c1jKLFSfGfnF8wB+fLCsLFJZkB7GPqyYZCgIFxGTDCXkYC8eaP9iLB5kBCrvJVEwDcX0kYAmCGOjmeYC8CGEkE8-iw5kEwjgpFiENmUAAwoguGwgRAQXtkCBXGFoGdmKwAGLGAJhHhjcw8KDmLA+Lyfb6-ETmOAeLx-BJgDS8fiokRGEzkJxOUBjdHyFJCEAeFYQJlOABKEA5UGUpIKbiFowg+Q8SUV-itEggpO2AB5Ov4+HMQFgpCB2gAfZCbBZLJ1YMIAKwgAjdIA9Xp9yH97s93qabWQ1hwlB6AF86HQwMRzCh0Ng8BAACKR4wSBnNQpi5xKfJJuP+SzI6AFgDSEGIlutICc-kUHgQUGAKOM+QIViLhuIldH48nUFT7QzBh6dAlAjVyDcpfwBTKFWQAHlVGAnSWcPgiGQKJR-AByVbrCCB7bP1oNfI3suVnCyRglAPQ7lK+6YLeEDHs4p4XsAV7-neJDkFQL5Yo4eIIASRIgt+v5oFBAFVsBxhbuBe4HtBBQ0HEiTJAI6SZBAOT+JhuL4hAhLAjwYDDmsGyzEG5jDlqzJ6gayjINmf7EShD5UIRyEVqRNYmPQjA3GAbIiAUDZOE2mgxsmIBtrOnbED2fbeAOQ4BEuE4gdOFnzouY5OcYhEbtQaaaYwyDHJIvw4dx2KrN6PHQPkeo6VAIBsIWEBOgWRZYFcPhgFFIKtFuAWBQCIVmHWoCIdhXAFG+QlbEsDR5QFcLqpIdqGmw4VxtlfEtMgACEHHldxeF8bE+U7GSPByCNAX5AUa60FN+XAFcBRFWFjgRQlRKrvN+W7YyEn6sYRpFAUz44pC46DqYKCrYCvGgltz51Qtu0cZ1YArVxbXrR1W3PXtjC5gD+26odhrQDsJ0AAYDeM4Q3RdoSgMgAAkND9V9Q1gNmGAeFD-0A6t31QBtUUmHWmXvQUs0tO0-nA4o2qSUdEMnWdF2KtdPxsjc4A-FxyDpc+7FcQTwO3SVGRgANlWCR+wnbGLAOMwdUms6dsMcfY+TC-zoVK-l2bsRdA3vQbMn1YwjVWoL44KiA6LoGIWC+Q5HkrkU1jonoltu8uIGe97KB1pRwiFCURQns2JlxvUTT0wFLVZe1kVY5Ds2+1bwg21ouisXkHTVN1oqMWkGR5zkxSlHB5sq6DatQOnz4ADKHHMnMyh06ZB9IiEcMZLHPNQ6XILrufZJQz0vUthSrUU4geAU4-5w08cvVnTV61wlB6h4E2SHWc+GR9y85ObjB18z4ON2zsW6Zzq3UDce+kGAuuP7v+-n9cSi6Vv9if1ftSQ2eY9qXzBsdU6AA5cEXEn4QBfpIIwfBKBPWAQ1bOkhn7726sYYA9tHZKA4C7SO5hzCkGIPpDgLx-AninmA8S9cWY31OgAZRApzDQIQpBxHAFASIyNVo638HRU+T4f6IJkubOemUl7lwnv4bBr9a6MKvpAs6AseDmEoPaVBIt9aZx-nFEQSiwDoOzNSCxdBczbj4LuMOpVpZcVlu+T8SxSQHCOCcNeWlf7xQUHLNx5hkBjW4QGBWwZQwRijMZWM3pWg03yGE50rpYmmQSc9GxocRChQQHMHUfBBBuCdGweiCJKBeCrpHOClRqhzVSTiBJEp8hsH8IhaAEg1SnnJBgSkUB2JfHABtfIOJCI9L6TtH4mDbKOAPhYaAMpHj8AgEUZMGdYidwKEFZA8RuoAAYug7IANRHOQE6EqyhOnZy6D4gKEoVqDLABtAmQMpmb3cK7OciyqhIFWV6dZvjjG8OoAAWhmYaHMeZQDKHgEgZAeJSAIAmlckA940LUH8gYJJFIqSxBANi3puKbEQkFF4M84ZIzS0eRtBFSKRggUmQAAWomWWiU0S4pGYhXAux56lFAML4WI2YGixFZGEUgwAEAzMEFKxQQFdE4keeWOMUw+7bGPCM9EdKUVosfGMnFGkpraT-uS6JVLCk0q1cigOcILVxl5ZY0ByBmXyRggnDlTF5H5xmnAepgr8pVXljVcw+Qkh6H9VI0VPBxWSulbGuV1ZICUEVXa70qqxBBI1fCq19K1S6qUgSiZCdjX+NNZSlNQy4y0utcYIotrK3egdUKp1YqJVSrtDKskwhAKJvsBWsAyrG1wE1Yi61ebUJ6sLdASZgKTUUoEP2y1o76W1vrU8+1vrHWivHF7SAcapVro2rRMkMkR3aqlPmyeU7DX5W2Wu-Z6CpDUOOLPR5ZyyS3Pyveo5dYACM5ijV+JyY8x1xLpmHrjGesdwhL3dUxQXX9Eb8XIAAKx7PQxh9DgqrHW0kO04CXTuooa3LhgI8qk0ADVoAeDVFFbquT8mFKJAUMtC7qVVpzepIoCbgJ9qVSqtVSw2mXKlB4AZqaQDPVI56EAPbeOUCo1AGjwg6OHy4Hkgp4BmOsfNQ2kA1aV1QD+bJtSSb+2DrMtCDponxN6antbLxRQ5QFChjx3RinlObTcPkNGbnKPUdo0SbM+MegOeOE5rATgXMybk+5gLKmiQ+ZoDF0z9gPOBbcMFuqQA).