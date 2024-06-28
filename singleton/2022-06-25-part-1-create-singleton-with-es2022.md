```ic-metadata
{
  "name": "Create Singleton with ES2022",
  "series": {
    "name": "Singleton",
    "part": 1
  },
  "date": "2022-06-25",
  "lastModifiedDate": "2022-07-01",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "javascript"],
  "canonicalLink": "https://dev.to/bwca/create-singleton-with-es2022-51ic"
}
```

# Create Singleton with ES2022

Create a singleton? Why? For the glory of OOP, of course!

ES2022 brings new cool feature of static initializers, which means now it's even easier to create
singletons in JS/TS.

- make constructor private so ~~scrubs~~ consumers don't instantiate your class;
- create a private static vip-one-of-a-kind instance;
- initialize the vip-one-of-a-kind instance with new ES2022 bells and whistles static block;
- serve the private static vip-one-of-a-kind instance using public static getter

```typescript
class Foo {
    private static instance: Foo;

    static {
        Foo.instance = new Foo();
    }

    public static get getInstance(): Foo {
        return Foo.instance;
    }

    private constructor() { }
}
```

Looks more neat to me than the old way:

```typescript
class Foo {
    private static instance: Foo;

    public static get getInstance(): Foo {
        if(!Foo.instance){
            Foo.instance = new Foo();
        }
        return Foo.instance;
    }

    private constructor() { }
}
```

Though they both are consumed in the same way
```typescript
const bar = Foo.getInstance;
```

OOP fans rejoice :)