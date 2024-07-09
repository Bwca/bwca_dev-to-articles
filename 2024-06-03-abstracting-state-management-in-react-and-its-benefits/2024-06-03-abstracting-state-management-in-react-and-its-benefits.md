```ic-metadata
{
  "name": "Abstracting State Management in React with Typescript, and its Benefits",
  "series": null,
  "date": "2024-06-03",
  "lastModifiedDate": "2024-07-09",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "angular"],
  "canonicalLink": "https://dev.to/bwca/abstracting-state-management-in-react-with-typescript-and-its-benefits-8mk"
}
```

# Abstracting State Management in React with Typescript, and its Benefits

State management is something unavoidable on the UI, be it local component scoped, or global, which in some enterprise applications can grow to gargantuan sizes.

The topic I would like to dwell upon today is tight coupling, which can be created by a state management library, how state management can be abstracted and what benefits and drawbacks it has. Perhaps it should be noted that for the applications that are not maintained in the long run or are short-lived, it does not really matter which state management tool is used or if there is an abstraction at all. Yet, those should be obvious.

As a part of this article we will take a look at `Redux` in a `React` application, the coupling it creates, how it can be abstracted and tested and how the state management tool can be swapped once the abstraction has been created, so the public api of the state management remains the same for the app, but the actual provider of it changes.

Firstly let us create a `React` application and add `Redux`:
```bash
npx create-react-app demo_react-abstracting-state-management --template typescript && install @reduxjs/toolkit react-redux
```

Following the [quick start](https://react-redux.js.org/tutorials/typescript-quick-start) we can add a counter state, a component to render it, and get something similar to what is found in [this commit](https://github.com/Bwca/demo_react-abstracting-state-management/commit/dc8bf22b6a947b0dc74edc3e0a6be9fe1bf0f703).

Our counter slice is the following, it is almost identical to the one from the `Redux` quickstart:
```typescript
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

import type { RootState } from '../../store';

// Define a type for the slice state
interface CounterState {
    value: number;
}

// Define the initial state using that type
const initialState: CounterState = {
    value: 0,
};

export const counterSlice = createSlice({
    name: 'counter',
    // `createSlice` will infer the state type from the `initialState` argument
    initialState,
    reducers: {
        increment: (state) => {
            state.value += 1;
        },
        decrement: (state) => {
            state.value -= 1;
        },
        // Use the PayloadAction type to declare the contents of `action.payload`
        incrementByAmount: (state, action: PayloadAction<number>) => {
            state.value += action.payload;
        },
    },
});

export const { increment, decrement, incrementByAmount } = counterSlice.actions;
```

The noticeable difference is that we are placing `Redux`-related items in `store/redux`, because we intend to have abstraction in `store` and `store/redux` will be one of the implementations. The end goal is to provide an abstract exports from `store`, in a way that consumers are no longer coupled with a specific state management library.

Apart from a significant amount of boilerplate, which is not critical, what problem does it introduce to the application? The `Counter` component is now coupled with `Redux`, and so is our whole application, since it is wrapped in the `Redux` provider. While this is not a problem if your tech stack is married to `Redux`, this somewhat is limiting and coupling: changes to `Redux` in the long run could cause introducing changes to `Counter` component or other consumers, since all components utilizing imports from `Redux` will be coupled with it.

See the `Counter` component:
```typescript
import { decrement, increment, useAppDispatch, useAppSelector } from '../../store/redux';

export const Counter = () => {
    const count = useAppSelector((state) => state.counter.value);
    const dispatch = useAppDispatch();

    return (
        <div>
            <div>
                <button
                    aria-label="Increment value"
                    onClick={() => dispatch(increment())}
                >
                    Increment
                </button>
                <span>{count}</span>
                <button
                    aria-label="Decrement value"
                    onClick={() => dispatch(decrement())}
                >
                    Decrement
                </button>
            </div>
        </div>
    );
};
```

Were we to migrate to some other state management tool, the scope of change would be large as well, since `Redux` would have roots all over the place. You could consider it a soft form of vendor lock, when you could migrate, but the effort is not worth it, it is beneficial for libraries to impose such bounds, but not necessarily beneficial for your application, especially if a better alternative appears on the horizon.

Looking at the exposed state, we can deduct some abstractions: a model for state, a model for hook to access state and methods to modify it and a model for the state provider, which is a wrapper for our application.

Observe the [following commit](https://github.com/Bwca/demo_react-abstracting-state-management/commit/bf69400fcfe6bd7998c8d9a3d57bc14fba16f4b4).

So the state and managing methods can be expressed with the following interface:

```typescript
// src/store/models/counter.state.ts
export interface CounterState {
    count: number;
    increment: () => void;
    decrement: () => void;
}
```

The hook delivering them would have the following functional interface:
```typescript
// src/store/models/counter-state-hook.model.ts
import { CounterState } from './counter.state';

export interface CounterStateHook {
    (): CounterState;
}
```

And, finally, the provider:
```typescript
// src/store/models/state-provider.model.ts
import { ReactElement, ReactNode } from 'react';

export interface StateProvider {
    (args: { children?: ReactNode }): ReactElement;
}
```

Evidently, the `Redux` items, currently present in our application hardly conform to these abstractions, and we would need to create an adapter to make things compliant, so let us create one, which will encapsulate `Redux` state management:

```typescript
// src/store/redux/features/counter/counter-hook.ts
import { useCallback } from 'react';

import { useAppDispatch, useAppSelector } from '../../hooks';
import { decrement, increment } from './counter-slice';
import { CounterStateHook } from '../../../models';

export const useCounterHook: CounterStateHook = () => {
    const count = useAppSelector((state) => state.counter.value);

    const dispatch = useAppDispatch();

    const inc = useCallback(() => {
        dispatch(increment());
    }, [dispatch]);

    const dec = useCallback(() => {
        dispatch(decrement());
    }, [dispatch]);

    return {
        count,
        decrement: dec,
        increment: inc,
    };
};
```

Also, the `Redux` provider now needs to be adapted to conform to the provider interface:
```typescript
// src/store/redux/StateProvider.tsx
import { Provider } from 'react-redux';

import { store } from './store';
import { StateProvider } from '../models';

export const StateContextProvider: StateProvider = ({ children }) => {
    return <Provider store={store}>{children}</Provider>;
};
```

With these in place, we can now provide abstract exports from `store` which conceal the actual implementation of the underlying state management library.

Here goes our new state management provider:
```typescript
// src/store/state.provider.ts
import { StateProvider as Provider } from './models';
import { StateContextProvider as ReduxStateContextProvider } from './redux';

export const StateProvider: Provider = ReduxStateContextProvider;
```

And the hook:
```typescript
// src/store/counter.hook.ts
import { CounterStateHook } from './models';
import { useCounterHook as ReduxUseCounterHook } from './redux';

export const useCounter: CounterStateHook = ReduxUseCounterHook;
```

Which in turn means we can use them in `Counter`:
```typescript
import { useCounter } from '../../store';

export const Counter = () => {
    const { increment, decrement, count } = useCounter();

    return (
        <div>
            <div>
                <button aria-label="Increment value" onClick={increment}>
                    Increment
                </button>
                <span>{count}</span>
                <button aria-label="Decrement value" onClick={decrement}>
                    Decrement
                </button>
            </div>
        </div>
    );
};
```

The `Counter` no longer has any idea which library manages the state, or even if it s a library at all, which is good.

Now, provider for the app, and a small cleanup for imports:
```typescript
// src/index.tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';

import reportWebVitals from './reportWebVitals';

import { App } from './App';

import { StateProvider } from './store';

const root = createRoot(document.getElementById('root') as HTMLElement);
root.render(
    <StrictMode>
        <StateProvider>
            <App />
        </StateProvider>
    </StrictMode>
);

// If you want to start measuring performance in your app, pass a function
// to log results (for example: reportWebVitals(console.log))
// or send to an analytics endpoint. Learn more: https://bit.ly/CRA-vitals
reportWebVitals();
```

Some state provider, no details, also good.

Now, before we proceed to migrate to a different state management tool, what we could do is write tests covering the public api of our `store` module, namely the `useCounter` hook, which we can do using `React Testing Library`:

```typescript
import { act } from 'react';
import { renderHook } from '@testing-library/react';

import { useCounter } from './counter.hook';
import { StateProvider } from './state.provider';

describe('Tests for the counter state', () => {
    test('should increment counter', () => {
        const { result } = renderHook(useCounter, { wrapper: StateProvider });

        act(() => {
            result.current.increment();
        });

        expect(result.current.count).toBe(1);
    });

    test('should decrement counter', () => {
        const { result } = renderHook(useCounter, { wrapper: StateProvider });
        const current = result.current.count;

        act(() => {
            result.current.decrement();
        });

        expect(result.current.count).toBe(current - 1);
    });
});
```

Once we swap the implementation, the test will show us if everything is working, and we do not need to re-visit consumers.

As a poof of concept for state management tool migration, I chose `Zustand`, because all the kool kids write articles about it and I had never used it before. Apparently it is pronounced `/ˈʦuːʃtant/` and the name itself is taken from German, but don't take my word for it.

```bash
npm i zustand
```

So we create a folder `zustand` next to the `redux`, it is going to be another implementation of our abstraction. It has no providers, so we would have to implement and empty one to comply with our abstraction (observe the [commit](https://github.com/Bwca/demo_react-abstracting-state-management/commit/1129c0070e20d89f9748ce32986e58a8d9177814)):
```typescript
// src/store/zustand/StateProvider.tsx
import { StateProvider } from '../models';

export const StateContextProvider: StateProvider = ({ children }) => {
    return <>{children}</>;
};
```

However, its store creating function actually accepts and interface, and we can pass the existing one we created earlier, `CounterState`, which means no extra adapters, it will just work (which is amazingly convenient):

```typescript
// src/store/zustand/counter.hook.ts
import { create } from 'zustand';
import { CounterState } from '../models';

export const useCounter = create<CounterState>((set) => ({
    count: 0,
    increment: () => set((state) => ({ count: state.count + 1 })),
    decrement: () => set((state) => ({ count: state.count - 1 })),
}));
```

All what is left is to swap implementations for the hook:
```typescript
// src/store/counter.hook.ts
import { CounterStateHook } from './models';
import { useCounterHook as ReduxUseCounterHook } from './redux';
import { useCounter as ZustandUseCounterHook } from './zustand';

export const useCounter: CounterStateHook = ZustandUseCounterHook; // || ReduxUseCounterHook;
```

And the state provider:
```typescript
// src/store/state.provider.ts
import { StateProvider as Provider } from './models';
import { StateContextProvider as ReduxStateContextProvider } from './redux';
import { StateContextProvider as ZustandStateProvider } from './zustand';

export const StateProvider: Provider = ZustandStateProvider; // || ReduxStateContextProvider;
```

We can run the tests now and make sure switching the underlying library, which does the heavy-lifting for state management had no impact on the `store` module's public api, which means the consuming components need no changes.

So to sum it up, what are the benefits? Looser coupling for components and state management, more flexibility, easier migration when changing the state management library. What are the drawbacks? Some abstraction overhead and need to write some interfaces to define public api and potentially adapters to make state management library comply.

That's it, have fun, the repo is [here](https://github.com/Bwca/demo_react-abstracting-state-management) :)