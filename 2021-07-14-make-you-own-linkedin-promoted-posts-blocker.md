```ic-metadata
{
  "name": "Make Your Own Linkedin Promoted Posts Blocker",
  "series": null,
  "date": "2021-07-14",
  "lastModifiedDate": "2021-07-14",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial"],
  "canonicalLink": "https://dev.to/bwca/make-your-own-linkedin-promoted-posts-blocker-fj5"
}
```

# Make Your Own Linkedin Promoted Posts Blocker

So the other day I was browsing Linkedin and figured out there was a number of re-occuring posts in my feed. At first I paid no attention to them, but as they hang on for days, I took a closer look and figuired out those were promoted.

Well, that's not the UX I choose, so I decided I'd write a browser extension to remove them and share a story how to make one.

So let's start with an empty typescript ('cos all the kool kids use it) project for a browser extension.

```js
mkdir linkedin-promoted-posts-eliminator
cd linkedin-promoted-posts-eliminator
npm init -y
```

Let's add a source folder to keep our typescript files and a `src/manifest.json` file with the following contents:

```js
{
    "manifest_version": 2,
    "name": "Linkedin remote promoted posts",
    "version": "1.0",

    "description": "Remove promoted stuff",

    "content_scripts": [
        {
            "matches": ["https://www.linkedin.com/feed/"],
            "js": ["index.js"]
        }
    ]
}
```

We only want our extension to have access to the feed page. Also create our entry file, `src/index.ts`.

Now, let's add typescript and tsup to transpile it.

```js
npm i -D typescript tsup
```

Also let's add a build command for transpilation and copying the manifest file to the dist folder (pardon my powershell, currently using Windows):

```js
  "scripts": {
    "build": "tsup src/index.ts --minify && @powershell copy \"./src/manifest.json\" \"./dist/manifest.json\""
  },
```
  
The time has come to think how the feed page works. Basically it loads a bunch of posts, and then adds more on scroll. So what we need to do is:

- to perform a check if there are any promoted posts present and hide them;
- start listening to page updates and run our promoted eliminator every time new posts arrive;
- not to hide same post twice, we can keep track of the hidden posts;
- not to call eliminator too often, we can create a debounced check.

That's 3 functions in total, let's get down to implementing them and start with the simplest one of them, the debounce wrapper:

```ts
// debounce.function.ts
export function debounce(fn: () => void, milliseconds: number): () => void {
    let timeout: number;

    return () => {
        if (timeout) {
            clearTimeout(timeout);
        }
        timeout = setTimeout(fn, milliseconds);
    };
}
```

Fairly simple, takes a function and returns a debounced version of it.

In order to follow page updates, we need to use MutationObserver that would trigger a given callback when more nodes are inserted into the container:

```ts
// listen-to-page-updates.function.ts
export function listenToPageUpdates(
    targetNode: HTMLElement,
    mutationCallback: () => void
): void {
    const config: MutationObserverInit = {
        attributes: false,
        childList: true,
        subtree: true,
    };

    const callback: MutationCallback = (mutationsList) =>
        mutationsList.forEach((mutation) => {
            const hasTasksListUpdated =
                mutation.type === 'childList' && mutation.addedNodes.length;
            if (hasTasksListUpdated) {
                mutationCallback();
            }
        });

    const observer = new MutationObserver(callback);

    observer.observe(targetNode, config);
}
```

Finding promoted posts was pretty tricky, but we can utilize xpath superpowers and simply look for the Promoted text in `//div[div/div/div/div/a/div/span[contains(., 'Promoted')]]` (wish they had given it a custom class to make this easier).

Adding a WeakSet to store already hidden posts we get something like this:

```ts
// hide-promoted.function.ts
const promotedPostsSet = new WeakSet();

export function hidePromoted(): void {
    const promotedPosts = document.evaluate(
        "//div[div/div/div/div/a/div/span[contains(., 'Promoted')]]",
        document
    );

    let item: Node | null;
    while ((item = promotedPosts.iterateNext())) {
        if (promotedPostsSet.has(item)) {
            continue;
        }
        (item as HTMLElement).style.display = 'none';
        promotedPostsSet.add(item);
    }
}
```

If we run into a promoted non-yet-hidden post, we set its display to none and add it to the set, simple as that.

The index.ts will serve our entry point and will contain an instantly invoked function ('cos why not).

```ts
import { listenToPageUpdates } from './listen-to-page-updates.function';
import { hidePromoted } from './hide-promoted.function';
import { debounce } from './debounce.function';

void (function main() {
    const mainContainer = document.getElementById('main');
    if (!mainContainer) {
        throw 'No main element found!';
    }
    hidePromoted();

    const debounceTimeMs = 500;
    const debouncedHidePromoted = debounce(hidePromoted, debounceTimeMs);

    listenToPageUpdates(mainContainer, debouncedHidePromoted);
})();
```

Load, eliminate all promoted posts on sight and go on obliterating them as they load :D

Now all that's left to do is to build the extension with

```js
npm run build
```

Open chrome, go to extensions, enable developer mode, select load unpacked, navigate to the dist folder which was created and load your freshly baked promoted posts eliminator.

Full [repo](https://github.com/Bwca/linkedin-antiprom) for anyone interested.

Enjoy ^_^
