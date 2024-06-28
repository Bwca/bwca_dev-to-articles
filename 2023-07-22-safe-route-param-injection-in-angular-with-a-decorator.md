```ic-metadata
{
  "name": "Safe Route Param Injection in Angular with a Decorator",
  "series": null,
  "date": "2023-07-22",
  "lastModifiedDate": "2023-07-28",
  "author": "Volodymyr Yepishev",
  "tags": ["angular", "tutorial", "routing", "generics"],
  "canonicalLink": "https://dev.to/bwca/safe-route-param-injection-in-angular-with-a-decorator-1lk7"
}
```

# Safe Route Param Injection in Angular with a Decorator

The intro and last passage of this article were provided by ChatGPT, who(?) was the first one to read the article and provide feedback.

The final code for this article is available at [github](https://github.com/bwca/fork_hall-of-heroes) (since code fragments here are screenshots)

Route changes can have serious consequences in large Angular applications, and sometimes cause ripple effects and hidden bugs. When working with route parameters, it's crucial to ensure safety and maintainability in the codebase. In this article, we will delve into a powerful technique that leverages TypeScript's type-checking capabilities and a custom decorator to enhance the safety of extracting route parameters in Angular.

Traditionally, developers have relied on loosely defined strings for route parameters, which can lead to potential bugs and errors. In this article, we'll address this concern by introducing a more robust approach. We'll explore how TypeScript's static typing can be utilized to extract parameters accurately and enable intelligent code completion.

To further elevate safety, we'll implement a custom decorator that leverages the power of observables. This decorator will seamlessly inject route parameters into properties, ensuring a clean and type-safe approach throughout our Angular components.

By the end of this article, you'll have a deep understanding of this technique and be equipped with the knowledge to implement safe route param injection in your Angular projects. Let's embark on this journey to enhance safety and maintainability in our Angular applications.

Checking the classic Angular hall of heroes tutorial one can observe a code fragment dealing with getting hero id from route params in hero details component, which reads as follows:

```typescript
// src/app/hero-detail/hero-detail.component.ts
  constructor(
    private route: ActivatedRoute,
    private heroService: HeroService,
    private location: Location
  ) {}

  ngOnInit(): void {
    this.getHero();
  }

  getHero(): void {
    const id = parseInt(this.route.snapshot.paramMap.get('id')!, 10);
    this.heroService.getHero(id)
      .subscribe(hero => this.hero = hero);
  }

```

With the app-routing.module.ts providing the following information about routes:

```typescript
// src/app/app-routing.module.ts

const routes: Routes = [
  { path: '', redirectTo: '/dashboard', pathMatch: 'full' },
  { path: 'dashboard', component: DashboardComponent },
  { path: 'detail/:id', component: HeroDetailComponent },
  { path: 'heroes', component: HeroesComponent }
];
```

What are the points which can be improved here? The first one is, of course, loose strings in paths: there is no explicit connection between the :id in the path to the details component and the id grabbed from the params in the component. Should it become 'heroId', something migh break silently.

Let us discover how typescript can be used to increase the safety of extracting params from route, so only the existing params can be extracted, and also, how a decorator can be used for this purpose. Therefore, what we are aiming at, is having heroId$ as an Observable, injected into the hero details component via decorator.

Since a decorator is a function, it is a bit tricky to have it access Router or ActivatedRoute, however, it is possible: if APP_INITIALIZER were to be used to pass those references to our decorator, this could do the trick. However, there is a problem with this approach, as property decorators would fire prior to APP_INITIALIZER, which means we need the references before we can get them. To counter this we need something that can be available right away for subscription and which would serve as a channel to pass url params, so we need a BehaviorSubject, which will be used to broadcast params from the Router, which is to be injected later with APP_INITIALIZER. Consider the following code fragment:

```typescript
import { NavigationEnd, Router } from '@angular/router';
import { BehaviorSubject } from 'rxjs';
import { filter, map } from 'rxjs/operators';

const params$ = new BehaviorSubject<Record<string, string>>({});

export function routeInitializer(router: Router): () => Promise<any> {
  return () =>
    new Promise<void>((resolve) => {
      router.events
        .pipe(filter((event) => event instanceof NavigationEnd))
        .subscribe(() => {
          params$.next(
            router.routerState.snapshot.root.firstChild?.params ?? {}
          );
        });

      resolve();
    });
}
```

Now we have a broadcasting channel which we can use inside a function, even before it starts receiving any events from the Router, we would just need to keep our decorator function in this file, so it has the access to it.

The time has come for some typescript kung fu, w are going to write a generic to extract parameters from a string, we know that paths cannot start with a forward slash and parameters are preceded by a column, therefore a generic to extract them would look as follows:

```typescript
type PathParameter<T extends string> = T extends `:${infer P}/${infer R}`
  ? P | PathParameter<`${R}`>
  : T extends `${infer _}/${infer R}`
  ? PathParameter<`${R}`>
  : T extends `:${infer P}`
  ? P
  : unknown;
```
Works like a charm, now we have intellisense on our side:

```typescript
const param: PathParameter<':path/:userId/:id/:name/profile/:profileId'> = 'id';
```

The next thing would be swapping loose strings for consonants and using those in paths instead:

```typescript
// src/app/routes.const.ts
export const ROUTES = <const>{
  DASHBOARD: 'dashboard',
  DETAIL: 'detail/:id',
  HEROES: 'heroes',
};

// src/app/app-routing.module.ts
const routes: Routes = [
  { path: '', redirectTo: ROUTES.DASHBOARD, pathMatch: 'full' },
  { path: ROUTES.DASHBOARD, component: DashboardComponent },
  { path: ROUTES.DETAIL, component: HeroDetailComponent },
  { path: ROUTES.HEROES, component: HeroesComponent },
];

```

What is left now is to write a simple decorator function to do the param injections into a property, which it will receive from the params$ BehaviorSubject, created earlier:

```typescript
export function param<T extends string>(parameter: PathParameter<T>) {
  return (target: any, propertyKey: string) => {
    target[propertyKey] = params$.asObservable().pipe(
      map((p) => p[<string>parameter]),
      filter(Boolean)
    );
  };
}
```

Viola, now we have a safe decorator for parameter injections, which can be used in the hero details component as following:

```typescript
  @param<typeof ROUTES.DETAIL>('id')
  heroId$!: Observable<number>;

  constructor(private heroService: HeroService, private location: Location) {}

  ngOnInit(): void {
    this.getHero();
  }

  getHero(): void {
    this.heroId$
      .pipe(
        first(),
        switchMap((id) => this.heroService.getHero(id))
      )
      .subscribe((hero) => (this.hero = hero));
  }
```

Simple, isn't it?

With this newfound knowledge, you are now empowered to take your Angular applications to new heights of safety and maintainability. Happy coding!
