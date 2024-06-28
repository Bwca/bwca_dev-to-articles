```ic-metadata
{
  "name": "Iframe Microfrontends: Intro and Setup",
  "series": {
    "name": "Iframe Microfrontends",
    "part": 0
  },
  "date": "2022-05-30",
  "lastModifiedDate": "2022-05-30",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "microfrontends", "react", "angular"],
  "canonicalLink": "https://dev.to/bwca/iframe-microfrontends-intro-and-setup-1g7h"
}
```

# Iframe Microfrontends: Intro and Setup

This is going to be split into several articles.

There's a bunch of ways of implementing microfrontends and all the kool kids on the block do it. Well, let's get our hands dirty and see how a microfrontend can be implemented using perhaps one of the ~~ugliest~~ simplest way possible, using `iframe`s. What are the cons of such approach? You get a page inside a page, with all additional requests that come out of it. Want a button in an `iframe` built with `Vue`? Have fun loading as many Vue runtimes as there are buttons on your page. If it's just a button, you'd more likely be better off with web components, but I digress. The advantages of `iframe`s are obvious too: rock solid isolation, so it is safe to load even Angular inside one, it won't monkey patch your main window or react in any way to something that's happening outside the `iframe`. And since sometimes the number of files loaded is not really an issue (i.e. your developing a desktop app and not a web app), `iframe`s can be a viable choice for building microfrontends (you could prove me wrong though).

## What we're building
So, what we're going to build is an `Angular` application which interacts with [The Bored API](https://www.boredapi.com/) and uses a `React` application inside an `iframe` to display results. The requests will be triggered by clicking the button in the `React` app. Moreover we will make the `React` app a standalone application, so it can function even when accessed as a separate application. It is going to determine if it's loaded as a module of the shell or a separate application.

## The tools
We're going to use [Nx](https://nx.dev/), it is an indredible tool for monorepo, which fits our needs, since it can work with both `React` and `Angular`, and will allow us to share code using libraries.

## Let's do this!

First of all we're going to create an empty nx workspace for developing applications:
```bash
npx create-nx-workspace@latest demo__nx-iframe-microfrontends --preset=apps
```

Our next step is adding `Angular` and `React` plugins and `concurrently` package, so we can run two applications simulaneously:
```bash
npm install -D @nrwl/angular @nrwl/react concurrently
```

Also let's update `scripts` section so we can use `nx` in the command line:
```json
// package.json
"scripts": {
    ...
    "nx": "nx",
```

Having added `nx` to scripts and with plugins ready, we can now proceed to creating the Angular application, which will serve as a shell:
```bash
npm run nx -- g @nrwl/angular:app angular-shell --style=scss --routing --prefix=app
```
 `React` application to display our bored-api request results:
 ```bash
npm run nx -- g @nrwl/react:app react-app --style=scss --routing
 ```

 And a library which will be used to share models between the two frontend apps:
 ```bash
npm run nx -- g @nrwl/js:library models
 ```

 With both applications ready, it is time to update the `scripts` section of `package.json` once again, so they can be run at the same time using the `concurrently` package:
 ```json
"scripts": {
    "start": "concurrently --kill-others \"nx serve react-app\" \"nx serve angular-shell --port=4201\"",
 ```

 So we'll have `React` on port 4200, which is default and `Angular` on 4201.

 That's it for the first part, in the next one we'll work on `React` app and prepare it.