```ic-metadata
{
  "name": "Reducing the Boilerplate for Services Utilizing Subjects in Angular",
  "series": null,
  "date": "2024-05-06",
  "lastModifiedDate": "2024-07-09",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "angular", "rxjs"],
  "canonicalLink": "https://dev.to/bwca/reducing-the-boilerplate-for-services-utilizing-subjects-in-angular-44fp"
}
```

# Reducing the Boilerplate for Services Utilizing Subjects in Angular

If you are familiar with Angular, chances are, you are most probably familiar with using some type of `Subject` to hold a state in a service. Perhaps, the most well known example would be [Parent and children communicate using a service](https://angular.io/guide/component-interaction#parent-and-children-communicate-using-a-service) example from the official documentation. It established rather a pattern, which one can observe navigating through various projects, and the recurring theme is a service that contains a private `Subject` and exposes a public `Observable` for subscription.

In this article we will dive into this phenomenon and take a look at how it can be somewhat simplified.

Let us take a close look at the `MissionService`:

```typescript
import { Injectable } from '@angular/core';
import { Subject } from 'rxjs';

@Injectable()
export class MissionService {

  // Observable string sources
  private missionAnnouncedSource = new Subject<string>();
  private missionConfirmedSource = new Subject<string>();

  // Observable string streams
  missionAnnounced$ = this.missionAnnouncedSource.asObservable();
  missionConfirmed$ = this.missionConfirmedSource.asObservable();

  // Service message commands
  announceMission(mission: string) {
    this.missionAnnouncedSource.next(mission);
  }

  confirmMission(astronaut: string) {
    this.missionConfirmedSource.next(astronaut);
  }
}
```

What is the recurring pattern here? Essentially we maintain a private state, using a `Subject`, and expose a public method to alter the state and a public getter to observe the state. We do not want to expose the `Subject` itself, so the consumers cannot fiddle with the state and do something outrageous like closing it.

What problem does this approach have? As you might have noticed, for each state we need 2 properties and a method, due to the `Observable` being a converted `Subject`, we also have to put a private property before the public one, since otherwise we are referring to a property before its declaration. Also, we have to get creative with naming and come up with 3 variations expressing the same state concept, i.e. for mission announcement we have `missionAnnouncedSource`, `missionAnnounced$` and `announceMission`, which is juggling 2 words around with a prefix and Finnish notation.

Out there you could also discover similar approaches, but using `$$` as prefix for the `Subject`, so it would be `missionAnnounced$` and `missionAnnounced$$` or using underscore to differentiate between the private and public properties, i.e. `_missionAnnounced$` and `missionAnnounced$`.

This is a pattern, but can it be simplified in a way, so that a developer does not need to come up with 3 additional fields on a class every time a state is added to a service using a `Subject`?

I have been thinking about it every time I saw this approach, and I think, I came up with a trick. If essentially what we are doing is setting a state of a `Subject` and retrieving it as `Observable`, this could be performed using a getter/setter pair of methods on a function, which would expect state name as a parameter, opposed to creating a setter method per each piece of state. We would simply need a couple of Typescript generics to produce them.

Consider the following generic to provide a list of Subjects on a class:

```typescript
type SubjectsRecord<T> = {
  [K in keyof T]: T[K] extends Subject<any> ? T[K] : never
}
```

Once a `Subject` is extracted, it can be cast into `Observable` using a different generic:
```typescript
type ObservableValue<T extends Observable<any>> = T extends Observable<infer A> ? A : never
```

So far easy, but yet we encounter a problem: Subjects should be public, as generics cannot iterate of private members, this seems to throw a wrench into our idea, as our class would expose the Subjects. However, this does not have to be the case if we make them public, but place inside an object, assigned to a protected property.

Let us name it `sources` as a tribute to the `Source` suffix used in the `MissionService`. With these we are ready to create an abstract stateful service, that will enforce its children to have a pattern where to keep the Subjects and expose two public methods for setting and getting a state by the state name.

```typescript
import { Observable, Subject } from 'rxjs';

abstract class StatefulSubjectService<T extends Record<string, Subject<any>>> {
  // We pass the interface for sources to get intellisense for the methods
  protected abstract sources: T

  // State name equals one of the property names in sources, which has a Subject
  public getState$<K extends keyof SubjectsRecord<T>>(k: K): Observable<ObservableValue<SubjectsRecord<T>[K]>> {
    return this.sources[k];
  }

  // State name equals one of the property names in sources, which has a Subject, and value type is extracted from its type
  public setState<K extends keyof SubjectsRecord<T>>(k: K, v: ObservableValue<SubjectsRecord<T>[K]>): void {
    this.sources[k].next(v);
  }
}
```

Note that any extending service would have to communicate the interface for the `sources` property, so the intellisense for `getState$` and `setState` can work properly, while `sources` is protected, so it can be used in a generic, but cannot be exposed to some consumer to fiddle with the extending service.

Were the `MissionService` to extend this abstract service, it would look like this:

```typescript
@Injectable()
export class MissionService extends StatefulSubjectService<{
  missionAnnounced: Subject<string>,
  missionConfirmed: Subject<string>,
}> {
  protected sources = {
    missionAnnounced: new Subject<string>(),
    missionConfirmed: new Subject<string>(),
  }
}
```

Adding more states would only result into adding a subject to the interface passed to the underlying `StatefulSubjectService` and its creation inside the `sources` field.

The interface passing to the extended service could in theory be reduced to `class MissionService extends StatefulSubjectService<MissionService['sources']>`, but that constitutes a Typescript violation due recursive self reference, so technically it is a heresy.

Simple as that, a pair of getter/setter and a record of Subjects. No need to juggle words, create affixes or change public/private property order.

Of course, there is a slight api change, so instead of
```typescript
missionService.announceMission('some mission');
```

one would use the generic setState method:
```typescript
missionService.setState('missionAnnounced', 'some mission')
```

Thence, there is some trade back as you see, and I leave the decision if it is a good idea to you.

My thirst for investigating this topic has been quenched for now :D