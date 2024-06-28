```ic-metadata
{
  "name": "Iframe Microfrontends: Angular Shell",
  "series": {
    "name": "Iframe Microfrontends",
    "part": 2
  },
  "date": "2022-05-31",
  "lastModifiedDate": "2022-05-31",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "microfrontends", "react", "angular"],
  "canonicalLink": "https://dev.to/bwca/iframe-microfrontends-angular-shell-pmj"
}
```

# Iframe Microfrontends: Angular Shell

The repo is [here](https://github.com/Bwca/demo__iframe-micro-frontends).
The commit for this part of the tutorial is [here](https://github.com/Bwca/demo__iframe-micro-frontends/commit/daf1b2606ad059cdc84749cf9a33832c17a0fad5) :)

Before we start coding the `Angular` shell, let's first think about what we are going to need.

We will need a component to provide `iframe` and mount our `React` application, it's a feature, so it deserves its own module, and since lazy loading is a cool feature, it'll be lazy too! There's something twisted about lazy loading an `iframe`, which in return will load another application. Anyway, I digress.

So then, we also need a service to communicate with the Bored API in Angular and another service, which will handle the messaging between the `iframe` and our shell application. As you might have already guessed, we're going to use `postMessage` to throw messages between our microfrontends.

Let's start with the module:
```node
npm run nx -- g m frame --route frame --module app.module
```

Once it's created, let's update `app.module.ts` so all paths redirect to it:

```typescript
// ./apps/angular-shell/src/app/app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { RouterModule } from '@angular/router';

import { AppComponent } from './app.component';
import { NxWelcomeComponent } from './nx-welcome.component';

@NgModule({
  declarations: [AppComponent, NxWelcomeComponent],
  imports: [
    BrowserModule,
    RouterModule.forRoot(
      [
        {
          path: 'frame',
          loadChildren: () =>
            import('./frame/frame.module').then((m) => m.FrameModule),
        },
        {
          path: '**',
          redirectTo: 'frame',
        },
      ],
      { initialNavigation: 'enabledBlocking' }
    ),
  ],
  providers: [],
  bootstrap: [AppComponent],
})
export class AppModule {}

```

Also while we're at it, let's purify ~~with fire~~ `app.component.ts` and remove everything unnecessary from it.

```typescript
// ./apps/angular-shell/src/app/app.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  template: `<router-outlet></router-outlet>`,
})
export class AppComponent {}
```

Good, now onto creating the `ActivityService` which will live inside our `FrameModule`:
```node
npm run nx -- g s frame/services/activity  --skipTests=true
```

Now, let's update `FrameModule` a bit: first of all we need to add `HttpClientModule` to the imports, since our `ActivityService` will require it to communicate with the api, also let's add the newly created service to the providers (we will not provide it in root).

```typescript
// ./apps/angular-shell/src/app/frame/frame.module.ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Routes, RouterModule } from '@angular/router';
import { HttpClientModule } from '@angular/common/http';

import { FrameComponent } from './frame.component';
import { ActivityService } from './services/activity.service';

const routes: Routes = [{ path: '', component: FrameComponent }];

@NgModule({
  declarations: [FrameComponent],
  imports: [CommonModule, RouterModule.forChild(routes), HttpClientModule],
  providers: [ActivityService]
})
export class FrameModule {}
```

Time to update our `ActivityService`, for the sake of sadness let's add a filter to the api request so it only requests activities for one participant.

```typescript
// ./apps/angular-shell/src/app/frame/services/activity.service.ts
import { HttpClient, HttpParams } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';

import { ActivityItem } from '@demo--nx-iframe-microfrontends/models';

@Injectable()
export class ActivityService {
  constructor(private http: HttpClient) {}

  public getActivity(): Observable<ActivityItem> {
    const params = new HttpParams().set('participants', 1);
    return this.http.get<ActivityItem>(
      'http://www.boredapi.com/api/activity',
      {
        params,
      }
    );
  }
}
```

Time to produce probably one of the most important services, the `MessageService`. It is going to react to messages coming from the `iframe`, pipe them to api requests from `ActivityService` and send them back via `postMessage` to `iframe`'s `contentWindow`. Since it is going to be a service, it will not be watching `DOM` for events, but provide methods to set the `iframe` for messaging and a method which accepts `MessageEvent` bubbling from the `iframe`. It will be component's duty to watch the events and pass them to the service to handle, but later about it, let's create the service:

```node
npm run nx -- g s frame/services/message  --skipTests=true
```

Update the service with the following:

```typescript
// ./apps/angular-shell/src/app/frame/services/message.service.ts
import { Injectable, ElementRef, OnDestroy } from '@angular/core';
import { debounceTime, Subject, Subscription, switchMap } from 'rxjs';

import { ActivityService } from './activity.service';

@Injectable()
export class MessageService implements OnDestroy {
  private incomingMessage$$ = new Subject<MessageEvent>();
  private targetWindow: ElementRef<HTMLIFrameElement> | null = null;
  private subscription: Subscription | null = null;

  constructor(private activityService: ActivityService) {
    this.subscribeToMessages();
  }

  public ngOnDestroy(): void {
    this.subscription?.unsubscribe();
  }

  public set target(targetWindow: ElementRef<HTMLIFrameElement>) {
    this.targetWindow = targetWindow;
  }

  public requestActivity(event: MessageEvent): void {
    this.incomingMessage$$.next(event);
  }

  private subscribeToMessages(): void {
    this.subscription = this.incomingMessage$$
      .pipe(
        debounceTime(100),
        switchMap(() => this.activityService.getActivity())
      )
      .subscribe((v) => {
        this.targetWindow?.nativeElement.contentWindow?.postMessage(v, '*');
      });
  }
}
```

As you can see we utilize `Subject` to turn messages into a stream of observables, then pipe them to `getActivity` requests and post results to the `iframe`. No rocket science. Note how the service implements `OnDestroy` for unsubscription, this is because we intend to provide it on the component level, which will allow us to get access to this lifecycle hook.

Time to update our `iframe` component, but before that let's modify `environment`, so it contains the url to our `React` app. That's where we would normally store such url.

```typescript
// ./apps/angular-shell/src/environments/environment.ts
export const environment = {
  production: false,
  iframeUrl: 'http://localhost:4200',
};
```

Now we are ready to update `FrameComponent`. So what's the plan for it? It should contain only 1 element, the `iframe`, pass reference to it to the `MessageService` and alert it every time it detects the `message` event. For these we will utilize:
* `DomSanitizer` to sanitize the environmel url and throw it into `iframe`'s src;
* `ViewChild` decorator to obtain reference to the `iframe`;
* `HostListener` decorator to listen to the events;
* `AfterViewInit` hook to detect when the `iframe` is available in DOM.

And of course we are going to remove all styles, so it looks as ~~ugly~~ minimalistic as possible.

```typescript
// ./apps/angular-shell/src/app/frame/frame.component.ts

import {
  AfterViewInit,
  Component,
  ElementRef,
  HostListener,
  ViewChild,
} from '@angular/core';
import { DomSanitizer } from '@angular/platform-browser';

import { environment } from '../../environments/environment';
import { MessageService } from './services/message.service';

@Component({
  template: `<iframe
    #childWindow
    [src]="iframeUrl"
    width="400px"
    height="400px"
  ></iframe>`,
  providers: [MessageService],
})
export class FrameComponent implements AfterViewInit {
  @ViewChild('childWindow')
  public readonly iframe!: ElementRef<HTMLIFrameElement>;

  public readonly iframeUrl = this.sanitizer.bypassSecurityTrustResourceUrl(
    environment.iframeUrl
  );

  constructor(
    private messageService: MessageService,
    private sanitizer: DomSanitizer
  ) {}

  public ngAfterViewInit(): void {
    this.messageService.target = this.iframe;
  }

  @HostListener('window:message', ['$event'])
  private message(event: MessageEvent) {
    this.messageService.requestActivity(event);
  }
}
```

As you update everything you note that it doesn't work yet: `React` works as a standalone application and does not delegate anything. Fixing this will be addressed in the next post of the series, which is going to be the last one :)