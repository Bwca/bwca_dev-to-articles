```ic-metadata
{
  "name": "Crafting a Global State Hook in React",
  "series": null,
  "date": "2024-05-18",
  "lastModifiedDate": "2024-05-18",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "react", "state"],
  "canonicalLink": "https://dev.to/bwca/crafting-a-global-state-hook-in-react-4hb4"
}
```

# Crafting a Global State Hook in React

As usual, the cover image by AI, this time from Gemini.

The article features typescript generics, run before you can.

Have you ever wanted `useState` to work across different components, without passing props, using context or throwing a state management tool on top? In this article we will take a look how to craft such state hooks, which would operate in the global scope using `rxjs` or even a plain javascript object, so no extra libraries are involved.

Let us start with an abstraction to outline what we want. A classical `useState` application looks something like:
```typescript
  const [count, setCount] = useState(0);
```

What can be inferred from this example? We invoke a function that produces an array of a setter and sort-of-a-getter, which we manually have to name. Those also infer the type from the initial value passed to the `useState` hook. 

Since the scope of hook is limited to the invoking component and its children, each invocation requires an initial value, which would not be the case for a hook operating in the global scope, as it would require only one initial value passed to it. That would be the moment the global hook is created, so we would need a function that accepts an initial value and produces a hook, which would return getter/setter pair once invoked, we are talking about a factory here.

Yet, if we are already using a factory, could we do better than returning a getter/setter inside an array? Knowing the state name, we could produce the names ready and pack them inside an object, ready for destructuring, so the end user does not have to type. We would have to type it somehow though, since like all the kool kids, we are using typescript.

Consider the following generic:
```typescript
type GetterSetterPair<T, S extends string> = {
    [K in [S, `set${Capitalize<S>}`][number]]: K extends `set${Capitalize<S>}` ? (s: T) => void : T;
}
```

What is going on here? We take a type and a state name, and turn them into an object with getter/setter properties, using `Capitalize` utility type and some array unpacking to union. So, passed `GetterSetterPair<number, count>`, we would get `{ count: number; setCount: (s: number) => void }`. Rather convenient, is it not?

This leads us to the following signature for our hook function, since it requires no initial state:
```typescript
() => GetterSetterPair<T, N>
```

Where `T` is the type of the value stored in state and `N` is the string to generate getter/setter pair. As you might have noticed, this function is not a generic, though it returns one, so where do the types come from? Those would have to be provided by the actual hook factory, which would have the following signature:

```typescript
type StateHookFactory = <T, N extends string>(stateName: N, initialState: T) => () => GetterSetterPair<T, N>;
```

Everything is easier when you start with the interface, is it not? So how would a global hook work with `rxjs` implementation. As you might have guessed, we could create a closure with the factory function and return a simple hook utilizing `useState` and updating it by subscribing to a subject, namely like this:

```typescript
import { useCallback, useEffect, useState } from 'react';

import { Subject } from 'rxjs';

import { GetterSetterPair, StateHookFactory } from '../models';

export const subjectStateHookFactory: StateHookFactory = <T, N extends string>(stateName: N, initialState: T) => {
    const subject$ = new Subject<T>();

    return (): GetterSetterPair<T, N> => {
        const [state, set] = useState<T>(initialState);

        useEffect(() => {
            const subscription = subject$.subscribe((s) => set(s));
            return () => {
                subscription.unsubscribe();
            };
        }, []);

        const setState = useCallback((s: T) => {
            subject$.next(s);
        }, []);

        return { [stateName]: state, [`set${stateName.charAt(0).toUpperCase()}${stateName.slice(1)}`]: setState } as GetterSetterPair<T, N>;
    };
};
```

This way the closure keeps the state inside the subject and no matter how many hooks are invoked, though each of them would have their own `useState`, they all would still be in sync due to the subscription and updates in the `useEffect`.

Observing this implementation, one could wonder if something similar can be done with regular tools available in js, without using a library, since we are using a closure and it is a well known way to keep the state. The real question here would be how to telegraph that state to the hooks and how make them listen to it.

It indeed can be done using a `Set` to keep track of subscribing hooks, and a `Proxy` to update subscribers to the state changes:

```typescript
import { useCallback, useEffect, useState } from 'react';

import { GetterSetterPair, StateHookFactory } from '../models';

export const objectStateHookFactory: StateHookFactory = <T, N extends string>(stateName: N, initialState: T) => {
    const listeners: Set<(s: T) => void> = new Set();
    const currentState = new Proxy<{ state?: T }>(
        { state: initialState },
        {
            set(target: typeof currentState, property: string, value: T) {
                target.state = value;
                listeners.forEach((l) => l(value));
                return true;
            },
        },
    ) as { state: T };

    return (): GetterSetterPair<T, N> => {
        const [state, set] = useState<T>(currentState.state as T);

        useEffect(() => {
            listeners.add(set);
            return () => {
                listeners.delete(set);
            };
        }, []);

        const setState = useCallback((s: T) => {
            currentState.state = s;
        }, []);

        return { [stateName]: state, [`set${stateName.charAt(0).toUpperCase()}${stateName.slice(1)}`]: setState } as GetterSetterPair<T, N>;
    };
};
```

Essentially it is very similar to what happens with the `rxjs` example, with subscription being replaced by adding and removing setters to the listeners `Set`.

What is cool about these examples, as both factories implement the same interface, meaning they are interchangeable, i.e. could be assigned to a variable typed with the interface, based on a condition:

```typescript
import { subjectStateHookFactory } from './subject-state-hook-factory';
import { environment } from '../environment';
import { objectStateHookFactory } from './object-state-hook-factory';
import { StateHookFactory } from './models';

export const stateHookFactory: StateHookFactory =
    environment.REACT_APP_STATE_ENGINE === 'object' ? objectStateHookFactory : subjectStateHookFactory;
```

I have compiled a repo with a small app generating a random Shakespeare quote using these two hook factories as a [proof of concept](https://github.com/Bwca/demo_crafting-a-global-state-hook-in-react).

Rather easy, eh?
