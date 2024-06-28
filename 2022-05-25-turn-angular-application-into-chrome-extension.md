```ic-metadata
{
  "name": "Turn Angular Application into Chrome Extension",
  "series": null,
  "date": "2022-05-25",
  "lastModifiedDate": "2022-05-25",
  "author": "Volodymyr Yepishev",
  "tags": ["angular", "tutorial", "chrome", "extension"],
  "canonicalLink": "https://dev.to/bwca/turn-angular-application-into-chrome-extension-2mkn"
}
```

# Turn Angular Application into Chrome Extension

It's pretty easy actually, this is going to be a short one.

Step 1: create Angular application (duh), i.e.
```bash
npx @angular/cli new angular-chrome-extension
```

Step 2: add a manifest file to the source folder:
```json
{
  "manifest_version": 3,

  "name": "My App Extension",
  "description": "A basic chrome extension built with Angular",
  "version": "0.1",
  "action": {
    "default_popup": "index.html",
    "default_title": "Open the popup"
  },
  "content_security_policy": {
    "script-src": "self",
    "object-src": "self'"
  }
}
```

Step 3: add the manifest file to the build assets in `angular.json`:
```json
"architect": {
        "build": {
          "builder": "@angular-devkit/build-angular:browser",
          "options": {
            ...
            "assets": ["src/favicon.ico", "src/assets", "src/manifest.json"],
            ...
          },
```

Step 4: build
```bash
npm run build
```

Now you have an unpacked Chrome extension in `dist/angular-chrome-extension`, which you can load with developer mode on, enjoy :)

P.S. [repo with code](https://github.com/Bwca/demo__angular-chrome-extension)