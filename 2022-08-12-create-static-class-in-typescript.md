```ic-metadata
{
  "name": "Create a Static Class in Typescript",
  "series": null,
  "date": "2022-08-12",
  "lastModifiedDate": "2022-08-12",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial"],
  "canonicalLink": "https://dev.to/bwca/create-a-static-class-in-typescript-5ao1"
}
```

# Create a Static Class in Typescript

You can't have `static` classes in `Typescript` out of the box, but there are several approaches to make a class that cannot be instantiated. This article is about them.

## 1. Make constructor private
```typescript
class JustAClass {
  private constructor() { }
}
```

Intellisense will alert if you try to instantiate it.

## 2. Throw error in class constructor
```typescript
class Static {
  constructor() {
    throw new Error('No, you don\'t!');
  }
}
```

Throwing an error inside the constructor gets the job done, moreover other classes can now inherit this one to become static themselves:
```typescript
class JustAClass extends Static {
  constructor() { super(); }
}
```

Though no intellisense to help unlike in the first way.

## 3. Use abstract instead
```typescript
abstract class JustAClass {}
```

Intellisense is still going to alert you when trying to instantiate it, though it looks like a misuse of the `abstract` keyword.

## 4. Create a decorator to make any class static
This approach combines method 2 with typescript's decorators feature. We create a function that pumps out anonymous class which inherits a given class, and overrides its constructor call with throwing an error. And yes, it breaks all the rules and does not call the super like a bad boi, so we silence the compiler with `@ts-ignore`.
```typescript
function Static<T extends new (...args: any[]) => any>(ctr: T): T {
  return class extends ctr {
    // @ts-ignore
    constructor(...args: any[]) {
      throw new Error('no way dude');
    }
  }
}

@Static
class JustAClass {}
```

While not providing the safety of intellisense, this approach is quite declarative.

That's the four ways I discovered :)