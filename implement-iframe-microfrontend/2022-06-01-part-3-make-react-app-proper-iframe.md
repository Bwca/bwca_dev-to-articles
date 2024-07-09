```ic-metadata
{
  "name": "Iframe Microfrontends: Make React App Proper Iframe",
  "series": {
    "name": "Iframe Microfrontends",
    "part": 3
  },
  "date": "2022-06-01",
  "lastModifiedDate": "2024-07-09",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "microfrontends", "react", "angular"],
  "canonicalLink": "https://dev.to/bwca/iframe-microfrontends-make-react-app-proper-iframe-12g9"
}
```

# Iframe Microfrontends: Make React App Proper Iframe

The repo is [here](https://github.com/Bwca/demo__iframe-micro-frontends).
The commit for this part of the tutorial is [here](https://github.com/Bwca/demo__iframe-micro-frontends/commit/568de18c7e34ada4c55ac38ad7fbf9201b026cbd) :)

It is time for our final part, in which we will add functionality to the `React` app we have, so it can determine if it is a standalone app and use its own means of fetching activities, or if it is a part of something else, and delegate the request to the parent window.

The key point here is our `use-activity-provider` module. The only thing this folder exports to the rest of the application is the `useActivityProvider` hook, which returns a function, which corresponds to `GetActivity` interface. The rest is concealed under the hood of the module. What that means is that we simply need to craft another function for communicating with the parent window, which would correspond to `GetActivity` interface and then return it from our `useActivityProvider` hook in cases when our `React` app detects it is inside an iframe.

Sounds simple, right?

In order to do that we will need two more hooks inside `use-activity-provider` module, which will be working under its hood. The first one will do nothing but receiving messages which come down from the parent window, and the other one will serve as an adapter to pipe these messages to the familiar `GetActivity` interface, which the rest of the application is expected.

Finally,  `useActivityProvider` will be granted the logic to tell if the app is standalone or inside an inframe, and will get to pick which one of the two functions returning `Promise` to provide to the application.

These two new hooks deserve a module of their own, since they encapsulate a good chunk of logic, so we'll be placing them inside `use-ask-for-activity` folder in `use-activity-provider`.

We'll start with the simpler hook, the one which receives activities from messages:
```node
npm run nx -- g @nrwl/react:hook use-activity-from-message --directory=app/use-activity-provider/use-ask-for-activity --project=react-app --skipTests=true --export=false --flat
```

Now let's populate the file with logic. We will utilize `useEffect`, `useCallback` and `useState` hooks:

```typescript
// ./apps/react-app/src/app/use-activity-provider/use-ask-for-activity/use-activity-from-message.ts
import { useState, useCallback, useEffect } from 'react';

import { ActivityItem } from '@demo--nx-iframe-microfrontends/models';

export function useActivityFromMessage(): ActivityItem | null {
  const [activity, setActivity] = useState<ActivityItem | null>(null);

  const logMessage = useCallback((event: { data: ActivityItem }) => {
    setActivity(event.data);
  }, []);

  useEffect(() => {
    window.addEventListener('message', logMessage);
    return () => {
      window.removeEventListener('message', logMessage);
    };
  }, [logMessage]);

  return activity;
}
```

Looks fairly straightforward, doesn't it? We add a listener and every time activity comes down (for the sake of simplicity we are not performing any checks here, i.e. if it is really `ActivityItem`, etc.), we throw it into `useState` and send it further to whoever is using the hook. This hook has no idea how the activity is further delivered and that's the marvel of it.

Now we need our last hook, which will provide means for requesting activity from the parent window and return the result which it will obtain from our recently created `useActivityFromMessage`.

I suck at naming, so I will call it `useAskForActivity` :)

```node
npm run nx -- g @nrwl/react:hook use-ask-for-activity --directory=app/use-activity-provider/use-ask-for-activity --project=react-app --skipTests=true --export=false --flat
```

This one is going to be a bit more tricky: we will need it to return a promise, but we would have to manually resolve it with the result coming from `useActivityFromMessage`. Luckily we can easily obtain a reference to `resolve` of a `Promise` and keep it preserve using `useRef` hook :)

```typescript
// ./apps/react-app/src/app/use-activity-provider/use-ask-for-activity/use-ask-for-activity.ts
import { useEffect, useRef } from 'react';

import { ActivityItem } from '@demo--nx-iframe-microfrontends/models';

import { GetActivity } from '../../models/get-activity.model';
import { useActivityFromMessage } from './use-activity-from-message';

export function useAskForActivity(): GetActivity {
  const activity = useActivityFromMessage();

  const megares = useRef<(activity: ActivityItem) => void>();

  useEffect(() => {
    if (activity) {
      activityResolver.current?.(activity);
    }
  }, [activity]);

  return (): Promise<ActivityItem> => {
    window.parent.postMessage(
      {
        message: 'plz give some activity, bro?',
      },
      '*'
    );
    return new Promise<ActivityItem>((res) => {
      activityResolver.current = res;
    });
  };
}
```

So as you see when the returned function is invoked by a consumer, it will message parent window, create a new `Promise`, store its `resolve` to `useRef` resolver and trigger it once activity comes from `useActivityFromMessage`!

All what's left is to tweak `useActivityProvider` to determine whether our app is standalone or `iframe`, we could use window location for the check and then return the correct version of `GetActivity` implementation:

```typescript
// ./apps/react-app/src/app/use-activity-provider/use-activity-provider.ts
import { GetActivity } from '../models/get-activity.model';
import { fetchActivity } from './fetch-activity.util';
import { useAskForActivity } from './use-ask-for-activity/use-ask-for-activity';

export function useActivityProvider(): GetActivity {
  const askForActivity = useAskForActivity();
  const isStandaloneApplication = window.location === window.parent.location;

  return isStandaloneApplication ? fetchActivity : askForActivity;
}
```

So now you have it, `http://localhost:4201/` run `Angular` application with `React` inside an iframe requesting `Angular` to do http requests, and at the same time there's a standalone `React` app `http://localhost:4200/` which functions independently.

Cool, eh? :)