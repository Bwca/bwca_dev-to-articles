```ic-metadata
{
  "name": "Decorating React Hook with Typescript",
  "series": null,
  "date": "2022-04-12",
  "lastModifiedDate": "2022-04-12",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "react"],
  "canonicalLink": "https://dev.to/bwca/decorating-react-hook-with-typescript-bd9"
}
```

# Decorating React Hook with Typescript

Decorators are awesome feature of typescript and an interesting design pattern. Too bad, in typescript the coolest decorators are rather class-oriented, so what do you do if you want to decorate something in React with its more function-style way?

The answer is higher order functions. In this tutorial we'll see how a React hook can be decorated using a higher order function and even have its return type altered with some typescript magic.

What could be the possible use cases for a decorator? Logging, caching, ~~showing off your typescript kung fu~~ etc.

For the purpose of this tutorial let us assume we have a useless hook with a unoriginal name `useStuff`.

```typescript
// ./hooks/use-stuff.hook.ts
import { useCallback, useState } from "react";

export const useStuff = (startValue: number) => {
  const [counter, setCount] = useState(startValue);

  const getStuffSync = useCallback((s: string) => "got some stuff sync", []);
  const getStuffAsync = useCallback(
    async (s: string, n: number) => Promise.resolve("got some stuff sync"),
    []
  );
  const failtToGetSomeStuffSync: () => string = useCallback(() => {
    throw new Error("no you dont");
  }, []);

  const failtToGetSomeStuffAsync: () => Promise<string> = useCallback(
    () => Promise.reject("no async for you"),
    []
  );

  return {
    getStuffSync,
    getStuffAsync,
    failtToGetSomeStuffSync,
    failtToGetSomeStuffAsync,
    setCount,
    counter,
  };
};
```

So it has a counter for no reason, a couple of synchronouse functions and a couple of asynchronous and some of them are destined to always fail. In real world scenario those could be api requests which could potentially fail, or some methods used in calculations which could throw, etc.

Now let's imagine we got tired of dealing with all those errors and decided that it would be a good idea to catch them all and simply return null if errors occur. What do we do with errors then? For simplicity let's dump them into user console.

Yet, there are four methods here and wrapping each and adding `try/catch` blocks to every one of them looks boring and repetetive. Besides it would also be good to alter return types of each method if we want to have `null` in case of errors. So changing return types in 4 places as well. Besides let's imagine this hook has been well covered with unit tests and any changes to return types would also require us to alter the tests file. Does not sound good.

However we can decorate this very hook to add all new functionality we need, meaning we add `try/catch` to each method and modify methods return types to be nullable.

First of all let us think about the interfaces that we are going to need.

The most basic one is the interface that fits any function, any hook or hook method extends it:
```typescript
// ./models/function-with-arguments.model.ts
export interface FunctionWithArguments {
  (...args: any): any;
}
```

Then we need an `Optional` generic since any hook method we are going to alter will be able to return `null` in case an error is encountered:

```typescript
// ./models/optional.model.ts
export type Optional<T> = T | null;
```

Based on these two basic types we can now create a type that can take a return function, synchronous or asynchronous and alter its return type to be optional:

```typescript
// ./models/function-with-optional-return.model.ts
import { FunctionWithArguments } from "./function-with-arguments.model";
import { Optional } from "./optional.model";

export type FunctionWithOptionalReturn<F extends FunctionWithArguments> = (
  ...args: Parameters<F>
) => ReturnType<F> extends Promise<infer P>
  ? Promise<Optional<P>>
  : Optional<ReturnType<F>>;
```

Now since we have the generic to alter functions, we can go ahead and create a generic to deal with the hook return type:

```typescript
// ./models/hook-methods-optionazed-returns.model.ts
import { FunctionWithArguments } from "./function-with-arguments.model";
import { FunctionWithOptionalReturn } from "./function-with-optional-return.model";

export type HookMethodsOptionalizedReturns<T extends FunctionWithArguments> = {
  [k in keyof ReturnType<T>]: ReturnType<T>[k] extends FunctionWithArguments
    ? FunctionWithOptionalReturn<ReturnType<T>[k]>
    : ReturnType<T>[k];
};
```

All required models are ready and we can create our decorator. It will accept a hook as an argument and produce a modified version of the passed hook, with altered methods, wrapped in `try/catch` blocks and possible `null` as return value in case of errors:

```typescript
// ./hooks/use-error-devourer.hook.ts
import { FunctionWithArguments } from "../models/function-with-arguments.model";
import { HookMethodsOptionalizedReturns } from "../models/hook-methods-optionazed-returns.model";

export const devourErrorsDecorator = <F extends FunctionWithArguments>(
  fn: F
) => {
  return (...args: Parameters<F>): HookMethodsOptionalizedReturns<F> => {
    const { ...result } = fn(...args);
    Object.entries<FunctionWithArguments>(result)
      // we've assumed only functions for typing purposes, so filter to safeguard
      .filter(([k, v]) => typeof v === "function")
      .forEach(([k, fn]) => {
        result[k] =
          fn.constructor.name === "AsyncFunction"
            ? async (...args: Parameters<typeof fn>) => {
                console.log("AsyncFunction called with ", ...args);
                try {
                  return await fn(...args);
                } catch (e) {
                  console.log("ASYNC failed");
                  return null;
                }
              }
            : (...args: Parameters<typeof fn>) => {
                console.log("Sync function called with ", ...args);
                try {
                  return fn(...args);
                } catch (e) {
                  console.log("SYNC failed");
                  return null;
                }
              };
      });
    return result;
  };
};
```

As you can see, it invokes the original hook and proceeds to modify its methods.

Now we can produce a new version of the `useStuff` hook, enhanced with our error-catching modifications:

```typescript
// ./hooks/no-error-use-stuff.hook.ts
import { devourErrorsDecorator } from "./use-error-devourer.hook";
import { useStuff as errorProneUseStuff } from "./use-stuff.hook";

export const useStuff = devourErrorsDecorator(errorProneUseStuff);
```

Pretty cool, isn't it? We've created a decorated version of a hook and altered all each methods, keeping the returned values and strongly typing everything.

Repo with the code can be found [here](https://github.com/Bwca/react-decorate-hook).