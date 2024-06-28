```ic-metadata
{
  "name": "Typescript Class Mixins",
  "series": null,
  "date": "2022-08-20",
  "lastModifiedDate": "2022-08-20",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial"],
  "canonicalLink": "https://dev.to/bwca/typescript-class-mixins-3hib"
}
```

# Typescript Class Mixins

So let's talk about #typescript, its superpowers and how it can be used for generating new classes on the fly and instantiating classes that do not really exist. We will be creating flying dogs and proud ducks.

Perhaps, one of the most fascinating features of #typescript is its mixins, which allow molding of classes (and not only those). In this article we will discover how mixin a base class can be used to create four different derived classes. It was quite fun to discover.

The cover image for this article is not random, to an extent it inspired the article. So let us think how we could use #typescript to create instances of a dog, fish, chicken and a duck. When you think about them, the obviously should have some sort of a super class. They also implement different ways of movement. For simplicity let us assume that some features are implemented the same way across their classes, i.e. walking is same for the dog and the chicken (though realistically chickens do not walk on four legs).

Now we can assume the superclass, which is going to be our base:

```typescript
class Animal {
  constructor(public readonly name: string) { }
}
```

Plain and simple, an animal that has a name.
Our animals implement different ways of moving around, which can be described with #typescript interfaces:
```typescript
interface IFly {
  fly(): void;
}

interface IWalk {
  walk(): void;
}

interface ISwim {
  swim(): void;
}
```

So a dog would implement `IWalk`, which a chicken would implement both `IFly` and `IWalk`.

Normally we would start creating all those classes, but mixins provide a more fun and flexible way: instead of statically extending the superclass, we can do so dynamically with mixins (which are essentially class decorators, used in a more straightforward way).
Before creating the mixins themselves, we can create a constructor generic to be used with the mixins.
```typescript
interface IConstructor<T> { new(...args: any[]): T; }
```

With the constructor generic ready, next step is to create three mixins that would enrich passed `Animal` class with new functionalities:
```typescript
function Walker<T extends IConstructor<Animal>>(animal: T) {
  return class extends animal implements IWalk {
    walk(): void {
      console.log(`I am a ${this.name} and I can walk!`);
    }
  }
}

function Swimmer<T extends IConstructor<Animal>>(animal: T) {
  return class extends animal implements ISwim {
    swim(): void {
      console.log(`I am a ${this.name} and I can swim!`);
    }
  }
}

function Flyer<T extends IConstructor<Animal>>(animal: T) {
  return class extends animal implements IFly {
    fly(): void {
      console.log(`I am a ${this.name} and I can fly!`);
    }
  }
}
```

So now to create a dog we do not need to create a separate class which extends the `Animal`, but can manufacture it by passing the class `Animal` to the `Walker` mixin:
```typescript
const dog = new class extends Walker(Animal) { }('dog');
dog.walk(); // "I am a dog and I can walk!" 
```

Same would be true for a fish and a chicken, the latter would get two methods:
```typescript
const fish = new class extends Swimmer(Animal) { }('fish');
fish.swim(); // "I am a fish and I can swim!" 

const chicken = new class extends Flyer(Walker(Animal)) { }('chicken');
chicken.fly(); // "I am a chicken and I can fly!" 
chicken.walk(); // "I am a chicken and I can walk!" 
```

Moreover, this approach allows adding even more properties/methods on the resulting instance and even overriding the extended methods, let's make a duck and override it's fly method:
```typescript
const proudDuck = new class extends Flyer(Swimmer(Walker(Animal))) {
  public override fly(): void {
    console.log('I don\'t fly as a chick, I fly like a duck!')
  }
}('duck');
proudDuck.fly(); // "I don't fly as a chick, I fly like a duck!" 
proudDuck.swim(); // "I am a duck and I can swim!" 
proudDuck.walk(); // "I am a duck and I can walk!" 
```

What is even more fascinating about this mixin approach, is that now we can even create a flying dog if we want to:
```typescript
const flyingDog = new class extends Flyer(Walker(Animal)) { }('a flying dog!');
flyingDog.walk(); // "I am a a flying dog! and I can walk!" 
flyingDog.fly(); // "I am a a flying dog! and I can fly!" 
```

And we still get to keep the types and intellisense. Amazing, isn't it? :)
P.S. the code for the article is [available here](https://www.typescriptlang.org/play?target=99&jsx=0#code/PQKhAIChwhJAbeBXAzgFwE4EM0EsD2AduAMb4AmApuAGb4bhoAW1WGeJ8l0ETaaABxQAuYMADukgHTxchANaVycqWQC2wAUngpKwNAE8BlFCQy4BaALScsKFFbW4AHnIcA3fPAoG1BjFYGlAK4KCzuPOB8giJiVO5SaPjAAEbiJFj6RiZmFta29o4ublYAzEy4KTzAkJAFKOAAgoS4aljw4ADe0KRE6BhIJEkYABRaKbIk4BiUWORE8AbghFhqlMLg-XIA5gCUXeAAvpDHkHJolBg0WCTUsABii109NIsjuxueuOQA3Ce150u11u4FgAHV2vJnuBwOJIe9Pvhvn9ToCrjc7gBlcStaGbHFqBHgL6-f5nQgXdEg2AAYT6mEGwwAPAAVAB8B0IlHEIykfLY2xE4CwhAMAG0ALofcAsn5HWo0JCEIYEYgQ+CKDCs8CUZwXQjkBq0+kDIb0JnNVrtNlskYiq3wDYs-bdGEzNBIDDEeo6vWUA0Ne1tDqtARcNYUo3qqGumGw+HSkl4uO9QgoLyUGT4bYjAAGsGFamF4AAJJ1mKEpCs1odhQbQaQRfGNQBCXO7P4p44w46nRXKvBEcDY1prLUs336w2gulphlmrWW4M2u0tYNOl09d2e73wOwNXVTwNr9rgUPh-1oI0jouxmEoAlEpN3uNkNMZrM5-OF4tlisoKtVkoWsRXIBsMmIB9WjbDseh7HpewVJUVSHR4gnHSd-WnY051NZkl2tW0g3aDc8W3L1SD3exMIDOsHTPNQw0oCMr1BNDk1oN5EyRMCXxhN90y4T88wLVZf3LCoAOrYC6zAgsIM4gwYM7ONu3lU4BLQcB5m2cAAF5lm5Sj9xo6do0uEYCPgF0jhGAByHS7NgnSpDhDV3j+Op6VoUImH0wzxGM6jDywhobzHSyT2sg5DnsmhfKcv54rCKQoMJWCvLnUgKhIRRiAMrlAp9ELaLQizzNGKzdhs2K7JIHK8sSuoGv9KRXgMDzmtwXLWrc+ROsy9BwAEDB8CQcgABFBihAqjOKv1SsWCzwvKyELKq6q8XGSZwHwdxLnMKhFKfHiOIEj9vBzOyC3mQgAB07K09rhUDbLuvkAAaBtntkRRi3IaaWychCTnsgHcqakaxsm6a2q4v4ofGqbctSx9YMRmGUb6gbNMUnYJuzfzCqCg8FunMrRgqyKHU2zpbLsrA8cIXSdKB2D2vx7NXITJLFk57Y4Y6jsgA)