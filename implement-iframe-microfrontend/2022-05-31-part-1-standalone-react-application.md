```ic-metadata
{
  "name": "Iframe Microfrontends: Standalone React Application",
  "series": {
    "name": "Iframe Microfrontends",
    "part": 1
  },
  "date": "2022-05-31",
  "lastModifiedDate": "2024-07-09",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "microfrontends", "react", "angular"],
  "canonicalLink": "https://dev.to/bwca/iframe-microfrontends-standalone-react-application-341f"
}
```

# Iframe Microfrontends: Standalone React Application

The repo is [here](https://github.com/Bwca/demo__iframe-micro-frontends).
The commit for this part of the tutorial is [here](https://github.com/Bwca/demo__iframe-micro-frontends/commit/5f23628a6f4b7f46213c3001ca6bcd4a9f518416) :)

Time to do some actual coding. In this post we'll complete the `React` app we previously created so it can function independently.

However, prior to that let's setup the `models` library: let us go to `./libs/models/src/lib`, remove the spec file, rename `models.ts` to `activity-item.model.ts` and update its contents with the `ActivityItem` interface, which corresponds to the entity returned by the [Bored API](https://www.boredapi.com/) we will be using.

Yep, we're using ~~ES4~~ the almighty Typescript.

```typescript
// ./libs/models/src/lib/activity-item.model.ts
export interface ActivityItem {
  activity: string;
  type: string;
  participants: number;
  price: number;
  link: string;
  key: string;
  accessibility: number;
}
```

Time to create our React component to display activity, in a most ugly way possible.
```node
npm run nx -- g @nrwl/react:component activity --project=react-app --skipTests=true --export=false
```

Onward to our newly created component, let's do some cleaning and add the logic to display an `Activity`. There's no need for props interface or default export, so we'll just removed them. We also remove styles, there's no place for beauty in our app. In the end our component should look like this:

```typescript
// apps/react-app/src/app/activity/activity.tsx
import { useState } from 'react';

import { ActivityItem } from '@demo--nx-iframe-microfrontends/models';

export function Activity() {
  // eslint-disable-next-line @typescript-eslint/no-unused-vars
  const [activity, setActivity] = useState<ActivityItem | null>(null);

  return (
    <div>
      <h3>Welcome to Activity!</h3>
      {activity &&
        Object.entries(activity).map(([k, v]) => (
          <p key={k}>
            <strong>{k}</strong>: {v}
          </p>
        ))}
    </div>
  );
}
```

Don't worry about the linter, the disabling comment is merely a temporary measure.

Our next target is `app.tsx`, it is going to be altered, so it only serves as means of navigating to our new component:

```typescript
// ./apps/react-app/src/app/app.tsx
import { Navigate, Route, Routes } from 'react-router-dom';

import { Activity } from './activity/activity';

export function App() {
  return (
    <Routes>
      <Route path="/activity" element={<Activity />} />
      <Route path="*" element={<Navigate to="/activity" replace />} />
    </Routes>
  );
}
```

Don't forget to update `App` import in `main.tsx` to a named one, since we are removing the default. All `spec` files and `nx-welcome.tsx` can be removed, they are not needed for this tutorial.

Let's now create a hook that can provide us with a function to request an activity from the Bored API. Of course, we could import a function directly, but in future we are going to perform the iframe check and that's why using a hook to import a function is better in our case: we are going to hide the logic where does the function comes from, so the component itself is not aware if it is inside an iframe or not.

```node
npm run nx -- g @nrwl/react:hook use-activity-provider --project=react-app --skipTests=true --export=false
```

So we have the hook, let's think about the interface of a function it is supposed to return. So we have two cases: 
* the application runs on its own and requests activity by itself;
* the application runs inside an iframe and asks its parent to request activity.

Both these can be reduced to a function interface, which requires no arguments and resolves to a promise with `ActivityItem`, which we will call `GetActivity` and place in `./apps/react-app/src/app/models/get-activity.model.ts`:
```typescript
// ./apps/react-app/src/app/models/get-activity.model.ts
import { ActivityItem } from '@demo--nx-iframe-microfrontends/models';

export interface GetActivity {
  (): Promise<ActivityItem>;
}
```

So now we need to implement an utility function that corresponds to this interface and will be used when the application is opened independently. Let's place it inside `use-activity-provider`, so it is concealed from the rest of the application:

```typescript
// apps/react-app/src/app/use-activity-provider/use-activity-provider.ts
import { ActivityItem } from '@demo--nx-iframe-microfrontends/models';

export async function fetchActivity(): Promise<ActivityItem> {
  const result = await fetch('http://www.boredapi.com/api/activity/');
  if (result.status === 200) {
    return result.json();
  }
  throw new Error('somethign went wrong');
}
```

Fairly simple use of fetch. Our provider hook is ready to provide it:
```typescript
// apps/react-app/src/app/use-activity-provider/use-activity-provider.ts
import { GetActivity } from '../models/get-activity.model';
import { fetchActivity } from './fetch-activity.util';

export function useActivityProvider(): GetActivity {
  return fetchActivity;
}
```

Though at this point `useActivityProvider` looks like something useless and unnecessary, it is crucial for us, as this is the place where we will get to pick our strategy to request activities in future.

Finally we can return to the `Activity` component and add some logic for requesting and displaying activity in the ugliest way possible:

```typescript
// apps/react-app/src/app/activity/activity.tsx
import { useCallback, useState } from 'react';

import { ActivityItem } from '@demo--nx-iframe-microfrontends/models';

import { useActivityProvider } from '../use-activity-provider/use-activity-provider';

export function Activity() {
  const [activity, setActivity] = useState<ActivityItem | null>(null);
  const getActivity = useActivityProvider();
  const handleGetActivity = useCallback(
    () => getActivity().then(setActivity),
    [getActivity]
  );

  return (
    <div>
      <h3>Welcome to Activity!</h3>
      <button onClick={handleGetActivity}>get some activity!</button>
      {activity &&
        Object.entries(activity).map(([k, v]) => (
          <p key={k}>
            <strong>{k}</strong>: {v}
          </p>
        ))}
    </div>
  );
}
```

It's ugly, and it works, that's all that matters, and that's the end for this part. In the next part we'll work in the `Angular` shell app.