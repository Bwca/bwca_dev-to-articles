```ic-metadata
{
  "name": "Create a Debounce Function from Scratch in Typescript",
  "series": {
    "name": "Implementing Debounce with Typescript",
    "part": 0
  },
  "date": "2021-08-26",
  "lastModifiedDate": "2022-04-07",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial"],
  "canonicalLink": "https://dev.to/bwca/create-a-debounce-function-from-scratch-in-typescript-560m"
}
```

# Create a Debounce Function from Scratch in Typescript

Debounce function is a neat tool that can help you every time you need to deal with a large amount of events over time that you need to process, i.e. when you are dealing with user typing something in a text input, scrolling, clicking, etc.

Let's see how to implement a simple debounce function with typescript that is capable of turning a simple function into its debounced version. This can be done with promises and setTimeout, so do not really need any specific dependencies.

Supposing each function we will be debouncing corresponds to the following interface:

```ts
(args: A) => R
```

It has an argument and a return type. I am using one argument here because it is really easy for use with generics and quite flexible, since if you have more than one you could simply wrap them in an object. Or could go with `(argOne: A, argTwo: B) => R`, but I digress.

Since we are going to turn a synchronous function into a debounced asynchronous version of itself, its interface would change the following way:

```ts
(args: A) => Promise<R>
```

Our delayed function needs to know how long it is supposed to wait before executing, so we need to provide a delay time in milliseconds, therefore a factory to generate debounced functions would be a function matching the following interface:

```ts
function debounce<A = unknown, R = void>(
    fn: (args: A) => R,
    ms: number
): (args: A) => Promise<R>;
```

Takes in a types for accepted argument and return, a function and delay time, and crafts debounced version of the function. Looks good, but there's something missing, namely there is no way to stop function execution once it has been called. This potentially could lead to situations when the object/element waiting for the function to execute has already been destroyed, which is no good. Let's add another return value, a function to terminate the execution and wrap it into a tuple:

```ts
function debounce<A = unknown, R = void>(
    fn: (args: A) => R,
    ms: number
): [(args: A) => Promise<R>, () => void];
```

Good. Now need the body that would create a closure with a state and return a promise that resolves to our function call:

```ts
// debounce.ts
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

So we have a factory that return a tuple with two functions, the first one which promises to call the originally passed function after the given amout of time. If we call it again, it gives us another promise and never fulfills the previous one. Which is kind of sad... the second function simply clears the timer and no promise is ever fulfilled.

So there you have it, a debounced function that can actually resolve to a value after certain amount of time. Or never resolves if terminated.

If we want to use it with React, we could wrap it into a hook with the following interface:

```ts
<A = unknown, R = void>(
    fn: (args: A) => R,
    ms: number
): ((args: A) => Promise<R>)
```

So we still accept generics for arguments and function return type, but this time we can hide the teardown function and instead put it into useEffect:

```ts
import { useEffect } from "react";

import { debounce } from "./debounce";

export const useDebounce = <A = unknown, R = void>(
    fn: (args: A) => R,
    ms: number
): ((args: A) => Promise<R>) => {
    const [debouncedFun, teardown] = debounce<A, R>(fn, ms);

    useEffect(() => () => teardown(), []);

    return debouncedFun;
};
```

So if the hook is destroyed, the function never executes.

Cool, eh?
