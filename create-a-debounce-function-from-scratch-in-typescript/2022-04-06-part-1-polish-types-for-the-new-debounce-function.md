```ic-metadata
{
  "name": "Polish Types for the New Debounce Function",
  "series": {
    "name": "Implementing Debounce with Typescript",
    "part": 1
  },
  "date": "2022-04-06",
  "lastModifiedDate": "2022-04-06",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial"],
  "canonicalLink": "https://dev.to/bwca/polish-types-for-the-new-debounce-function-5e0n"
}
```

# Polish Types for the New Debounce Function

There is always a room for improvement, so let's take a look what can be improved in the debounce function we've created in the previos article:

```typescript
// debounce.function.ts

export function debounce<A = unknown, R = void>(
  fn: (args: A) => R,
  ms: number
): [(args: A) => Promise<R>, () => void] {
  let timer: NodeJS.Timeout;

  const debouncedFunc = (args: A): Promise<R> =>
      new Promise((resolve) => {
          if (timer) {
              clearTimeout(timer);
          }

          timer = setTimeout(() => {
              resolve(fn(args));
          }, ms);
      });

  const teardown = () => clearTimeout(timer);

  return [debouncedFunc, teardown];
}
```

It does the job, but looks clunky:
- timer type tied to Nodejs;
- does not allow multiple primitive arguments, i.e. two numbers;
- return type is hard to read.

We'll start with the easiest one, the timer type. Instead of typing it using `NodeJS.Timeout` we could type it in a more sly way with `ReturnType`:

```typescript
let timer: ReturnType<typeof setTimeout>;
```

So timer is whatever `setTimeout` returns, no arguing that.

Now perhaps to the most interesting part: allow passing to the debounce function any amount of arguments of any type instead of one stictly typed object.

To get there first we need to understand an interface that is applicaple to any function in typescript, if we gave it a name, and let's say, called it `FunctionWithArguments`, it would look the following way:

```typescript
// ./models/function-with-arguments.model.ts

export interface FunctionWithArguments {
  (...args: any): any;
}
```

This single interface will allow us to eliminate the necessity to type separately the argument type and the return type in `debounce<A = unknown, R = void>`, we could go straight to a type that expects a function instead of argument + return type, which would look like this: `debounce<F extends FunctionWithArguments>(fn: F, ms: number)`.

So we get a function `F` which is an extension of `FunctionWithArguments`, how would a debounce function interface look like then? It would take the aforementioned function and utilise types `Parameters` and `ReturnType` generics to unpack whatever arguments and return type the `F` function carries:

```typescript
// ./models/debounced-function.model.ts

import { FunctionWithArguments } from './function-with-arguments.model';

export interface DebouncedFunction<F extends FunctionWithArguments> {
  (...args: Parameters<F>): Promise<ReturnType<F>>;
}
```
As you can see, `DebouncedFunction` accepts any function, and produces a function which is its async version, without the need to explicitly pass arguments and return types.

Having dealt with the first two points, it is time now to make return type of `debounce` a bit more readable.

`[(args: A) => Promise<R>, () => void]` basically equals to `Array<DebouncedFunction<F> | (() => void)>`, so we can strictly type it by creating a separate interface:

```typescript
// ./models/debounce-return.model.ts
import { DebouncedFunction } from './debounced-function.model';
import { FunctionWithArguments } from './function-with-arguments.model';

export interface DebounceReturn<F extends FunctionWithArguments> extends Array<DebouncedFunction<F> | (() => void)> {
  0: (...args: Parameters<F>) => Promise<ReturnType<F>>;
  1: () => void;
}
```

There we go, a strictly typed tuple.

Putting it all together we get a better typed debounce function, which no longer requires passing argument and return type explicitly, but infers them from the passed function insead:

```typescript
// debounce.function.ts
import { DebouncedFunction, DebounceReturn, FunctionWithArguments } from './models';

export function debounce<F extends FunctionWithArguments>(fn: F, ms: number): DebounceReturn<F> {
  let timer: ReturnType<typeof setTimeout>;

  const debouncedFunc: DebouncedFunction<F> = (...args) =>
    new Promise((resolve) => {
      if (timer) {
        clearTimeout(timer);
      }

      timer = setTimeout(() => {
        resolve(fn(...args as unknown[]));
      }, ms);
    });

  const teardown = () => {
    clearTimeout(timer);
  };

  return [debouncedFunc, teardown];
}
```

Try it live [here](https://www.typescriptlang.org/play?#code/JYOwLgpgTgZghgYwgAgGIFcQLMA9iAdWDAAsBBKAc3QFsJwBnZAbwChlkAKAOl7ioYAuZHBABPAJTDRYgNysAvq1ahIsRCgAiEAEa5MSACYYsOfAB5UyCAA9IIQ0xPY8hYuSq16YBgD4W7Fy83PyUQsgACvxwdGoMlr5SkVC4NMAMEOYAShBg6FAgACpiAA6ZqL6+8koq4NDwSMjaegYQOXkFltZ29I5oBmZupBTUdIz+tvZ9FFBwYubN+lgQxgOuCcgAPlycEsgAvP4AbrjAhhL+bBwADMI8fALCUbOx0PEVe4fJqemZ7flFUrlSryDgARjun2Op0M1WUMDW+GQhl0SyQXUmvSciKGHlG3j8nBgIGEqAANMgaOEQLQdNAkotWv9OhUAhwADa5ZA4OhQYTMwFlcxgIG4GDIDJgQrAOj6MBVZQcBD4BhgZGo1qrLDCRnLLUuCys-ZBB5hKGBDggCAAd2+aQynE4UAgDFw7KOEChbI4PuA4s4PPp3p9PoQnP40tl6DAAZl9NBIeQNUT3LjUAOEtykYgcsdXquKeQztd7ogRJA9xCAhETEwAGsQLhrSAANoAXQkEgTIYUFKpXYtSYHgWVIFV3Ig-EMTZAGd2B0ug7Dk6g2dzgagA44CnkgWdHVnLZRLT1zgpkCnM7bcNYo-HcEMhjnIAhyBpNDpUApIAATMJ35+XovsgADUb4-rud5qkeGp6mQj5thmx5omWD6GBSYLXNcw4wLg6acJyao2Bm1yyMgxHmMgmGkeRIEgXsBbIZq8GGJwYIUj+EjcKQ9CcHeboQNw7K4JQEiKEAA).

The repo is [here](https://github.com/Bwca/merry-solutions__debounce).

Packed as npm package [here](https://www.npmjs.com/package/@merry-solutions/debounce).