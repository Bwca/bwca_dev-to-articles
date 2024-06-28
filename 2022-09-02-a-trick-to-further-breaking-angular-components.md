```ic-metadata
{
  "name": "A Trick to Further Breaking Down Angular Components",
  "series": null,
  "date": "2022-09-02",
  "lastModifiedDate": "2022-09-02",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "angular"],
  "canonicalLink": "https://dev.to/bwca/a-trick-to-further-breaking-down-angular-components-1i8a"
}
```

# A Trick to Further Breaking Down Angular Components

The usual Angular way of dealing with things is placing business logic in services and presentation logic in components. It is a good approach, as all the heavy lifting is delegated to services which are imported and re-used by different components. Yet, sometimes components start acquiring logic of their own, with time developing private methods involved with data processing on component level.

This sometimes becomes a problem with unit tests, as private methods and state are not testable, unless some dirty hacks are performed. Besides, unit testing a component requires mounting a module to declare and all dependent modules, which in turn increases the time required to run unit tests.

In this article I want to share an approach to further breaking Angular components into parts, so they are easier to create in unit tests and more flexible.

Let's take a look at a simple counter component:
```typescript
@Component({
  template: `<p>{{ calledTimes }}</p>
    <p><button (click)="increment()">increment</button></p>`,
  selector: 'app-root',
})
export class AppComponent {
  protected _calledTimes = 0;

  public get calledTimes(): number {
    return this._calledTimes;
  }

  public increment(): void {
    this._calledTimes++;
  }
}
```

It is pretty straightforward: click the button, counter increments by one. Is there a way to further break it down to pieces? There is, think of `AppComponent` as of class `App`, decorated by `Component` to create a new class, which extends the `App`. So let's extract the class and call it a `Counter`:

```typescript
export class Counter {
    protected _calledTimes = 0;

    public get calledTimes(): number {
        return this._calledTimes;
    }

    public increment(): void {
        this._calledTimes++;
    }
}

@Component({
    template: `<p>{{ calledTimes }}</p>
    <p><button (click)="increment()">increment</button></p>`,
    selector: 'app-root',
})
export class AppComponent extends Counter { }
```

What benefits does it bring us? Well, we have just made a perfectly agnostic class, not bound by the Angular, so it can be unit tested without mounting any ngModules. And since it is merely a class, it is free to be extended and decorated. We can create new components based on it or extend it to a service.

What if we wanted a new component that does the same counting, by increments by a 100 instead of 1? Well, fairly easy to do it by extending the counter and overriding the increment method:

```typescript
@Component({
  template: `<p>{{ calledTimes }}</p>
    <p><button (click)="increment()">increment</button></p>`,
  selector: 'app-increment-by-100',
})
export class IncrementBy100Component extends Counter {
  public override increment(): void {
    this._calledTimes += 100;
  }
}
```

So the main idea the *Component derived class deals with gluing the class to the template while having no state of its own. It is like a combo of a stateful and stateless component.