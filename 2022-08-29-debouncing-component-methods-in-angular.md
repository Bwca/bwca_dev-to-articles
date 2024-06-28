```ic-metadata
{
  "name": "Debouncing Component Methods in Angular",
  "series": null,
  "date": "2022-08-29",
  "lastModifiedDate": "2022-08-29",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "angular", "tutorial"],
  "canonicalLink": "https://dev.to/bwca/debouncing-component-methods-in-angular-34im"
}
```

# Debouncing Component Methods in Angular

Using component methods is generally unwise for known reasons, yet sometimes an application can have third party components that require using component methods in template to react to outputs.

In today's article we are going to explore a way of debouncing component's method execution, so it does not fire too often, without resolving to rxjs tools.

For the sake of simplicity we will be using a simple component that implements a counter.

```typescript
import { Component } from '@angular/core';

@Component({
  template: `<p>{{ calledTimes }}</p>
    <p><button (click)="increment()">increment</button></p>`,
  selector: 'app-root',
})
export class AppComponent {
  private _calledTimes = 0;

  public get calledTimes(): number {
    return this._calledTimes;
  }

  public increment(): void {
    this._calledTimes++;
  }
}
```

Everything is rather straightforward: there is a property that is incremented each time a handler in template is called. To show things down we could use rxjs' `Subject`s with `debounceTime`, but such solution would be limited to a component that implements it and not transportable to other components which might need debouncing their method calls.

What we can do as an alternative, is to create a parameterized decorator which would accept a debounce time in milliseconds and utilize `Promise` to postpone method execution.

```typescript
export function debounce(ms: number) {
  return (
    target: unknown,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ): void => {
    let timer: ReturnType<typeof setTimeout>;
    const originalMethod = descriptor.value;

    descriptor.value = function (...args: any[]) {
      new Promise((resolve) => {
        if (timer) {
          clearTimeout(timer);
        }

        timer = setTimeout(() => {
          resolve(originalMethod.apply(this, ...args));
        }, ms);
      });
    };
  };
}
```

What is cool about this approach is that now we have a separate debouncing mechanism which we can apply to any method of any component. So if we wanted to limit increment to 2 seconds, we could simply decorate the method:

```typescript
import { Component } from '@angular/core';

import { debounce } from './debounce/debounce.decorator';

@Component({
  template: `<p>{{ calledTimes }}</p>
    <p><button (click)="increment()">increment</button></p>`,
  selector: 'app-root',
})
export class AppComponent {
  private _calledTimes = 0;

  public get calledTimes(): number {
    return this._calledTimes;
  }

  @debounce(2000)
  public increment(): void {
    this._calledTimes++;
  }
}
```

Simple as that :)