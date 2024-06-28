```ic-metadata
{
  "name": "Create a Custom eslint Rule with Typescript",
  "series": null,
  "date": "2021-04-02",
  "lastModifiedDate": "2021-07-11",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "eslint", "npm"],
  "canonicalLink": "https://dev.to/bwca/create-a-custom-eslint-rule-with-typescript-4j3d"
}
```

# Create a Custom eslint Rule with Typescript

Creating custom eslint rules can often come handy. I enjoy using typescript, so I decided I'd share a way how to use its superpowers to create custom eslint rules.

## Step one

Create a repo and initialise an npm package in it:

```bash
mkdir my-awesome-eslint-rule && cd my-awesome-eslint-rule && npm init -y
```

Make sure your package name starts with `eslint-plugin-`, 'cos _this is the way_, i.e.:

```json
{
    "name": "eslint-plugin-my-awesome-rule",
    ...
```

Install the dependencies (we won't touch @types/estree in this tutorial, but it's quite handy for types when writing eslint rules, so I keep it here):

```bash
npm i -D  @types/eslint @types/estree @types/node tsup typescript
```

Basically we're installing three packages with types definitions, typescript and a _the simplest and fastest way to bundle your TypeScript libraries_ :)

## Step two

We'll be seriously developing, so we'll have a `src` and `dist` folders, for the source and produced output respectively, so let's configure our `package.json` accordingly:

```json
{
    "name": "eslint-plugin-my-awesome-rule",
    "main": "./dist/index.js",
    "files": [
    "dist/**"
    ],
    ...
```

Now let's create the `src` folder with an `index.ts` file as the entry point to our future rule:

```typescript
module.exports = {
    rules: {
        // our rules will go here
    },
};
```

## Step three

Let's create a ~~silly~~ simple rule in our `src` folder. Note, that a rule is a function that accepts `context` and returns rule listener. Both these types we can get from the `eslint`.
For the sake of demonstration let's create a rule that questions every import:

```typescript
// ./src/my-aswesome-rule.ts
import { Rule } from "eslint";

export function myAwesomeRule(context: Rule.RuleContext): Rule.RuleListener {
    return {
        ImportDeclaration(node) {
            context.report({
                node,
                message: 'Are you sure about this?'
            })
        }
    }
}
```

Now let's import it into our entry file and plug it in:

```typescript
// ./src/index.ts
import { myAwesomeRule } from './my-aswesome-rule';

module.exports = {
    rules: {
      'my-silly-rule': {
        create: myAwesomeRule,
      },
    },
};
```

## Step four

Let's transpile it to js to make it usable. Add a build command to the scripts section in the `package.json` file. We'll be using tsup, which is absolutely awesome for this:

```json
...
"scripts": {
    "build": "tsup src/index.ts --no-splitting --minify"
},
...
```

Run the command to get a minified tiny version of your rule in `dist` folder.

## Step five

Install and enjoy, you can install it directly from the `dist` folder, run `npm pack` to create an archive with your new rule or publish it to npm with `npm publish`.
After installing, don't forget to plug it into your `.eslintrc`, by adding it to the plugins section and activating its rule:

```json
{
    "plugins": ["my-awesome-rule"],
    "rules": {
      "my-awesome-rule/my-silly-rule": "warn"
    }
  ...
  
```

Have fun :)
