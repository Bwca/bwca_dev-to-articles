```ic-metadata
{
  "name": "Runtime Environment Config in Angular, but without Dependency Injection",
  "series": null,
  "date": "2024-06-08",
  "lastModifiedDate": "2024-07-09",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "angular"],
  "canonicalLink": "https://dev.to/bwca/runtime-environment-config-in-angular-but-without-dependency-injection-kno"
}
```

# Runtime Environment Config in Angular, but without Dependency Injection

The link to the repo for the code used in the article is provided at the end.

Configuring Angular to use environment variables packed in a json file, which is later pulled and provided by a dedicated service is a topic covered by several articles found on web. Yet, this approach sets a certain restriction on how the environment is provided, so it is no longer a ts import, but rather something provided inside Angular dependency injection system.

In this article we will take a look how to rewire Angular to use a json environment file, but without a separate service to provide it, in fact we are going to explore a way to preserve the environment.ts files for the dev experience. In the end we will be looking at environment.ts, which is transformed into environment.json, so the familiar way of using env files is preserved, at the same time new benefits of having a runtime configuration file is added. Which means migrating to the json file would not trigger any changes in the consuming components, which can be quite convenient if environment is used in many places.

Let us consider we have a typed environment, which has the following model, nesting is for demo purposes, to make it look more complicated:

```typescript
// src/environments/models/environment.model.ts
export interface Environment {
    api: string;
    something: {
        completely: {
            different: string;
        };
    };
}
```
And there is a consuming component that merely displays some values:
```typescript
// src/app/app.component.ts
import { Component } from '@angular/core';

import { environment } from '../environments/environment';

@Component({
    selector: 'app-root',
    standalone: true,
    template: `<p>
        Api: {{ environment.api }}
        <br />
        And now for something completely different:
        {{ environment.something.completely.different }}
    </p>`,
})
export class AppComponent {
    protected readonly environment = environment;
}
```

Let us add an extra environment `environment.development.ts`, which is going to be a copy of `environment.ts`. We will be utilizing it for development, while `environment.ts` will be to preserve the imports.

This is our starting point, now we need to rewire the application to use environment configuration from a json file instead with minimal changes to `src/app` and no changes to `AppComponent`.

The first things that needs to be done is converting the environment typescript file into json. We could leverage `npx` and `ts-node` with some edgy inline magic to achieve that. Consider adding the following command to the scripts in `package.json` (I am using Windows, the amount of backslashes could different on a better OS):

```json
"generate-env": "npx ts-node -O \"{\\\"module\\\":\\\"commonjs\\\"}\" -e \"const fs = require('fs'); const path = require('path'); const { environment } = require(path.join(process.cwd(), './src/environments/', (process.argv[1] || 'environment.development.ts'))); fs.writeFileSync(path.join(process.cwd(), './src/assets/environment.json'), JSON.stringify(environment));\""
```

Looks like a screenshot from war crimes in programming YouTube video, I know. Essentially it is an inlined javascript snippet, here is how it looks formatted:
```javascript
const fs = require('fs'); 
const path = require('path'); 

const { environment } = require(
    path.join(process.cwd(), 
        './src/environments/', 
        (process.argv[1] || 'environment.development.ts')
    )
);

fs.writeFileSync(
    path.join(
        process.cwd(), 
        './src/assets/environment.json'
    ), 
    JSON.stringify(environment)
);
```

As you can see, nothing special happens, just a given environment file is converted into json and placed inside `/src/assets/environment.json`. If no environment file name is passed, default to be used is `environment.development.ts`.

Updating the `start` and `build` commands we get the following:
```json
"start": "npm run generate-env && ng serve",
"build": "npm run generate-env -- environment.production.ts && ng build",
```

Now the json environment file is going to be generated every time the app is served in development mode or built for production.

Generated file is not yet used by anything, it has to be loaded in the app first. For the purpose of loading and consequently providing it, we will add a special class, `EnvironmentLoader`, which will have only static methods and properties. Think of it as a static class.

```typescript
// src/environments/utils/environment-loader.util.ts
import { Environment } from '../models/environment.model';

export class EnvironmentLoader {
    private static env: Environment;

    public static get environment(): Environment {
        return EnvironmentLoader.env;
    }

    public static async loadEnvironment(): Promise<void> {
        const response = await fetch('/assets/environment.json');
        try {
            EnvironmentLoader.env = await response.json();
        } catch (e) {
            console.log('Could not load config, oh no!');
        }
    }
}
```

It uses fetch, so it is independent of Angular and does not need to depend on `HttpClientModule`. It has to run before the application is fully bootstrapped, so we have to use the `APP_INITIALIZER` token and create a provider for it:

```typescript
// src/environments/providers/provide-environment.provider.ts
import { APP_INITIALIZER, Provider } from '@angular/core';

import { EnvironmentLoader } from '../utils/environment-loader.util';

export const providerEnvironment: () => Provider = () => ({
    provide: APP_INITIALIZER,
    useFactory: () => () => EnvironmentLoader.loadEnvironment(),
    multi: true,
});
```

With the provider in place, it is time to plug it into the application config:

```typescript
// src/app/app.config.ts
import { ApplicationConfig } from '@angular/core';

import { providerEnvironment } from '../environments/providers/provide-environment.provider';

export const appConfig: ApplicationConfig = {
    providers: [providerEnvironment()],
};
```

So far so good. We have the mechanism to create a static json configuration file, we have means to fetch and store it before the application get bootstrapped, now comes the most interesting part: to wire up environment.ts to use the values from the json file without introducing changes to the consumers.

Every time a getter is fired on `environment` object from `environment.ts`, it should get its value from the appropriate field of `EnvironmentLoader.environment`, as those come from the json file. If you are thinking about Proxy, you are on the right path, but plain Proxy would not do as we have an object with several nesting levels. What we need is not just a one Proxy wrapper, but a factory, which could call itself and craft as many proxies on the fly, as we need, every time it encounters an object as a value, when a getter fires.

```typescript
// src/environments/utils/create-proxy.util.ts
import { EnvironmentLoader } from './environment-loader.util';

export function createProxy<T extends object>(target: T, path = ''): T {
    return new Proxy(target, {
        get: function (obj, prop: string) {
            const fullPath = path ? `${path}.${prop.toString()}` : prop;

            const value = fullPath
                .split('.')
                .reduce((a, c) => a[c], EnvironmentLoader.environment as any);

            if (value && typeof value === 'object') {
                return createProxy(value, fullPath.toString());
            }
            return value;
        },
    });
}
```

What is happening over there? Every time a getter fires, we calculate the path to the property and reach out to `EnvironmentLoader.environment` to get the value, if it is an object, we return another Proxy, passing the path along the way, once we reach the primitive value, we return it. This is how we counter paths like `environment.something.completely.different`.

This is all the heavy-lifting to be done, the only thing left is to update `environment.ts` and set it to proxy from our factory:

```typescript
// src/environments/environment.ts
import { Environment } from './models/environment.model';
import { createProxy } from './utils/create-proxy.util';

export const environment: Environment = createProxy({} as Environment);
```

It is done now, the app is wired to a static configuration json file without any changes to consumers and no extra dependency injection. Just some ts/js magic. Now we can modify configuration after building the app, without having to rebuilt it. Something quite useful when you do not know where you deploy beforehand.

Build once, run anywhere, eh? :)

P.S. the [poc repo](https://github.com/Bwca/demo_runtime-environment-config-in-angular-but-without-dependency-injection).