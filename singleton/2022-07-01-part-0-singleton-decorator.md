```ic-metadata
{
  "name": "Singleton Decorator",
  "series": {
    "name": "Singleton",
    "part": 0
  },
  "date": "2022-07-01",
  "lastModifiedDate": "2022-07-01",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial"],
  "canonicalLink": "https://dev.to/bwca/singleton-decorator-526h"
}
```

# Singleton Decorator

It turns out it's actually possible to apply singleton pattern using decorators in typescript.

Since decorator is essentially a wrapper-function, we can use it to return a fake anonymous class and use its constructor
to trap an instance of the decorated class in a closure variable, which we can reuse later when somebody tries to invoke
the class constructor again:
```typescript
function Singleton<T extends new (...args: any[]) => any>(ctr: T): T {

    let instance: T;

    return class {
        constructor(...args: any[]) {

            if (instance) {
                console.error('You cannot instantiate a singleton twice!');
                return instance;
            }

            instance = new ctr(...args);
            return instance;
        }
    } as T
}
```

Now we can use it to decorate any class to make it a singleton:
```typescript
@Singleton
class User {
    constructor(private name: string) { }

    public sayName(): void {
        console.log(`My name is ${this.name}`);
    }
}

let user = new User("Bob");
let secondUser = new User("not Bob");
user.sayName(); // "Bob"
secondUser.sayName(); // still "Bob"
```

[Playground](https://www.typescriptlang.org/play?jsx=0#code/GYVwdgxgLglg9mABAZRmA5gGwKZQQHgBVFsAPKbMAEwGdExsB3RACgDoOBDAJ3RoC5EnMAE8A2gF0AlIgC8APiGj5LaN0GEpGxAG8AUHsSIcURGhpRhEbBoDcBo91whuSCJk406+o0YgILbhBoOG52Ll4BJXFpXQdfIxhgVnNLSGwZHwTffzAaOBw2bG5uUJYAcgBNOBBECGEwOFNU4VhOCiFEGjQsXAREKEYYawBCcql7bN8nKBckFvTJ7IBfeISF6zl6JjqoMI42Hj4Jw2yZubM8tOslo1W7oTpCPVW9AAFUDBMEPXdPOgAqjRinE-AE9sE8GEAA7cGAAN3a2HonAAtjYunseplEK8jNCQAAjTDDLqcEQAOTR2BYWkQ8LgMCooJyAQK2DYmDg6BYAAMALIiFHosx0AAkOigAAsYDQ2GBqcteSc7i8DCZECBgdwtgxmEDiiwAEQAITghKNJw1wNyVANOtk2312uNjVMZotJy1xTYNHJVPRtPsNoQdu1vv91KDQA)

Interesting, eh?