```ic-metadata
{
  "name": "Improving Dynamic Component Loader Typings in Angular Using Inputs",
  "series": null,
  "date": "2024-04-15",
  "lastModifiedDate": "2024-04-15",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "angular", "generics"],
  "canonicalLink": "https://dev.to/bwca/improving-dynamic-component-loader-typings-in-angular-using-inputs-obl"
}
```

# Improving Dynamic Component Loader Typings in Angular Using Inputs

Cover image by Google's Gemini, I have no idea why it is a skull.

When I first read about signals in Angular, I was not very amused about two ways of declaring inputs and outputs on the component. However, I figured I would figure out something interesting with those eventually, and I actually did.

So today we will be learning how to improve [Dynamic component loader](https://angular.io/guide/dynamic-component-loader) from the official docs with signal bells and whistles.

Yet, first let us define the problem. It shyly hides in `ad.service.ts` on line 31 and reads as the following:
`as {component: Type<any>, inputs: Record<string, unknown>}[]`

Why is it a problem? There is no correlation between the actual inputs of the class and the inputs field, and cast to `{component: Type<any>, inputs: Record<string, unknown>}` is purely cosmetic and serves no purpose. Go on and change one of the inputs to 'banana', typescript compiler has been rendered silent with `as`.

Understandably, decorated with `@Input`, a public property string is still a string, and there is no way to write a generic to extract inputs from a typescript class. As you might have guessed, here is where signals enter.

Let us refactor the components to use signals instead of the Input decorator:
```typescript
import { Component, input } from '@angular/core';

@Component({
  standalone: true,
  template: `
    <div class="job-ad">
      <h4>{{ headline() }}</h4>
      {{ body() }}
    </div>
  `,
})
export class HeroJobAdComponent {
  headline = input.required<string>();
  body = input.required<string>()
}
```

```typescript
import { Component, input } from '@angular/core';

@Component({
  standalone: true,
  template: `
    <div class="hero-profile">
      <h3>Featured Hero Profile</h3>
      <h4>{{ name() }}</h4>
      <p>{{ bio() }}</p>
      <strong>Hire this hero today!</strong>
    </div>
  `,
})
export class HeroProfileComponent {
  name = input.required<string>();
  bio = input.required<string>();
}
```

In terms of functionality nothing has changed, the ads are produced as they were before, yet, these properties receiving strings have changed their types to `InputSignal<string>`, which means we could write a generic that accepts a class and extracts them, unpacking the underlying input type like this:

```typescript
type ComponentInputs<T> = {
  [P in keyof T]: T[P] extends InputSignal<infer A> ? A : never;
};
```

Neat? Now let us create a factory function that produces ads inside the `AdService` class, utilizing our new generic, which extracts inputs from a component class:

```typescript
private produceAd<T>(component: Type<T>, inputs: ComponentInputs<T>): { component: Type<T>; inputs: ComponentInputs<T> } {
    return {
      component,
      inputs,
    };
}
```

Essentially this is it, we can remove the embarrassing `as` cast and populate the ad array using our factory function, which allows us to have type safety even for this dynamic adventure:

```typescript
import { Injectable, Type, InputSignal } from '@angular/core';

import { HeroProfileComponent } from './hero-profile.component';
import { HeroJobAdComponent } from './hero-job-ad.component';

type ComponentInputs<T> = {
  [P in keyof T]: T[P] extends InputSignal<infer A> ? A : never;
};

@Injectable({ providedIn: 'root' })
export class AdService {
  getAds() {
    return [
      this.produceAd(HeroProfileComponent, {
        name: 'Dr. IQ',
        bio: 'Smart as they come',
      }),

      this.produceAd(HeroProfileComponent, {
        name: 'Bombasto',
        bio: 'Brave as they come',
      }),

      this.produceAd(HeroJobAdComponent, {
        headline: 'Hiring for several positions',
        body: 'Submit your resume today!',
      }),

      this.produceAd(HeroJobAdComponent, {
        headline: 'Openings in all departments',
        body: 'Apply today',
      }),
    ];
  }

  private produceAd<T>(
    component: Type<T>,
    inputs: ComponentInputs<T>
  ): { component: Type<T>; inputs: ComponentInputs<T> } {
    return {
      component,
      inputs,
    };
  }
}
```

Try messing with inputs and observe typescript compilation error, instead of a runtime error about missing imports when used with `as` cast and `@Input` decorators.

Pretty cool, eh? See the full code on [stackblitz](https://stackblitz.com/edit/fafrvv?file=src%2Fapp%2Fhero-job-ad.component.ts,src%2Fapp%2Fhero-profile.component.ts,src%2Fapp%2Fad.service.ts,src%2Fapp%2Fad-banner.component.ts).