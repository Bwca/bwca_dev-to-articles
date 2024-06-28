```ic-metadata
{
  "name": "Using Classes Inside React's Functional Components",
  "series": null,
  "date": "2022-08-22",
  "lastModifiedDate": "2022-08-22",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "react", "tutorial"],
  "canonicalLink": "https://dev.to/bwca/using-classes-inside-reacts-functional-components-oj3"
}
```

# Using Classes Inside React's Functional Components

This article is about smashing square shapes into round holes using the force :)

`React` encourages you to use functional approach, but what if you are stubborn and want to use classes instead? Well, if you are stubborn enough, you can.

Let's assume we are writing a counter, and come up with a class:

```typescript
export class Counter {
  private _value: number;

  constructor(initialValue: number) {
    this._value = initialValue;
  }

  public get value(): number {
    return this._value;
  }

  public increment(): void {
    this.add(1);
  }

  public decrement(): void {
    this.add(-1);
  }

  public add(n: number): void {
    this._value += n;
    console.log(`value changed, new value is: ${this._value}`);
  }
}
```
Then we go and choose a UI library and decide to use `React`, we're being naive and try to use our `Counter` class inside a functional component, creating a couple of instances:

```typescript
import { Counter } from "./counter/Counter.class";

export function App(): JSX.Element {
  const c = new Counter(100);
  const c2 = new Counter(-200);
  return (
    <div className="App">
      <section>
        <button onClick={() => c.decrement()}>decrement</button>
        {c.value}
        <button onClick={() => c.increment()}>increment</button>
      </section>
      <section>
        <button onClick={() => c2.decrement()}>decrement</button>
        {c2.value}
        <button onClick={() => c2.increment()}>increment</button>
      </section>
    </div>
  );
}
```

We hit some buttons and find out `React` does not update the UI, though in the console it's clear the values are getting updated. Now we could turn a class into a custom hook, but that'd be no fun.

Let us instead think about why the updates do not occur. The answer is simple: props did not change, component state did not change, no need to update the component. Pretty reasonable. So what we could do? Basically we need class methods to start forcing `React` component re-renders, which means they need to use some hooks.

As `Typescript` provides decorators for methods, we could use a custom decorator that would trigger component re-render when instance method is run:

```typescript
import { useState } from "react";

export function useReactChangeDetection(
  target: unknown,
  propertyKey: string,
  descriptor: PropertyDescriptor
): void {
  const [, setState] = useState<string | undefined>();
  const originalMethod = descriptor.value;
  descriptor.value = function (...args: unknown[]) {
    const result = originalMethod.apply(this, args);
    setState((prev) => (prev === undefined ? "" : undefined));
    return result;
  };
}
```

What is interesting, `React` does not allow using hooks outside functional components or other hooks, so we cannot apply the decorator directly to the `Counter` class, we need to think of something else.

Since our goal is to apply the hook-decorator to the `Counter` class, what we could do is writing a custom hook that manufactures a class extending `Counter` and applying the decorator to a given method name. Of course that requires us to write a generic that can extract the method names:

```typescript
export type ClassMethod<T> = {
    [P in keyof T]: T[P] extends (...args: any[]) => any ? P : never;
}[keyof T];
```

Now we can create our hook go generate extended classes of `Counter` superclass:

```typescript
import { useMemo } from "react";

import { ClassMethod } from "../ClassMethod.model";
import { Counter } from "./Counter.class";
import { useReactChangeDetection } from "./useChangeDetection.hook";

export const useCounterClass = (
  method: ClassMethod<Counter>,
  value: number
) => {
  class UseCounterClass extends Counter {
    @useReactChangeDetection
    public override [method](n: number): void {
      super[method](n);
    }
  }
  // eslint-disable-next-line react-hooks/exhaustive-deps
  return useMemo(() => new UseCounterClass(value), []);
};
```

Note how we override the super method and decorate it with the `useReactChangeDetection` hook, which is now perfectly fine as it is used inside a hook. Swapping the `new class Counter` with our new hook, we can even choose which class methods will trigger component update when instantiating:
```typescript
import { useCounterClass } from "./counter";

export function App(): JSX.Element {
  const c = useCounterClass("add", 100);
  const c2 = useCounterClass("decrement", -200);
  return (
    <div className="App">
      <section>
        <button onClick={() => c.decrement()}>decrement</button>
        {c.value}
        <button onClick={() => c.increment()}>increment</button>
      </section>
      <section>
        <button onClick={() => c2.decrement()}>decrement</button>
        {c2.value}
        <button onClick={() => c2.increment()}>increment</button>
      </section>
    </div>
  );
}
```

There, all the state is inside class instances and `React` has to respect the updates, outrageous, isn't it? :D