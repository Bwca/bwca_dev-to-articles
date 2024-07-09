```ic-metadata
{
  "name": "How I Decided to Write React cli and How it Went",
  "series": null,
  "date": "2021-08-08",
  "lastModifiedDate": "2024-07-09",
  "author": "Volodymyr Yepishev",
  "tags": ["angular"],
  "canonicalLink": "https://dev.to/bwca/how-i-decided-to-write-react-cli-and-how-it-went-4f19"
}
```

# How I Decided to Write React cli and How it Went

Perhaps I am spoiled beyond salvation by angular cli, which is an incredible tool that lets you create components with as little as some keystrokes, but every time I need to create a React component, I found the process quite tedious and repetitive.

- create a component file;
- create a separate file to hold the interface of its props;
- create an index file to export the component from its folder;
- create a stylesheet;
- create file for tests;
- create a story maybe?

That is a lot of files in the first place, however you are still supposed to add some crucial boilerplate some of these files before you can proceed working on your component.

Can we fix it? Yes, we can, thought I and wrote a first pretty basic cli command for generating component/props/index files from a given path when invoked. I put it on the github and invoked via npx. It did its job, but it lacked flexibility. The boilerplate it used for generating files was basically set in stone, so I could use it only for generating components only according to some predefined pattern.

I obviously needed a better tool to generate my react components. A tool more flexible and extensible. But how to make such flexible tool that would not dictate me how my components are made? I turned to logic-less templates and came up with the idea of having a templates folder filled with sub-folders named as entities that the cli tool would generate, i.e. `component`, `hook`, `story`, etc.

```txt
📂───templates
│   │
│   📂component
│   │  │───📃component.tsx.mustache
│   │  │   ...
│   │
│   📂hook
│   │  │───📃hook.ts.mustache
│   │  │   ...
```

At that point it became obviously that such tool can be completely framework-agnostic and could generate any number of files using given templates folder and sub-folder name. What was needed to be figured out was the arguments it would accept.

I came up with two required, one being the path of the item to generate, i.e. `components/MyNewComponent` and the second the `itemType`, which corresponds to the name of a subfolder in the templated directory. I also decided it would be cool to have two optional ones, `templatesRoot` to provide a custom folder with templates and `nameCase`, so you can pass a path as `components/my-new-component` and still get `MyNewComponent` as the component name for the react component.

It looked great, and it did not seem to be bound to a certain framework anymore. With mustache, I could come up with a any template and pass any number of key-value pairs to my tool for substitutions, i.e. I could make a text file template

```text
// ./templates/random/random.txt.mustache
Hello, {{ name }}! I am {{ reaction }} to {{ verb }} you!
```

And generate a file to green Bob with one command, passing arguments like:

```bash
some-random-stuff/happy-to-see-bob-file --itemType=random  --name=Bob --reaction=happy --verb=see
```

Now that was even cooler than what I originally expected :)

The cli tool I made was no longer bound to preset templates, it was not even bound to react as I originally imagined. I turned it into an npm [package](https://www.npmjs.com/package/@merry-solutions/cli) called @merry-solutions/cli that can be invoked without installation with some preset templates and made a [demo](https://github.com/Bwca/demo__cast) cra app with it. The command utilized itself is called "cast", because ~~open source naming sucks~~ the the process reminded of of casting something with molds. The only hardcoded thing is, it generates items inside a `src` folder, but I intend to delegate setting target folder to user-set argument in the next update.

Now I can create react components and hooks with a single command even without installing the package, i.e. (since I don't pass template folder path, defaults will be used, and there are defaults for component and hook)

```node
npx @merry-solutions/cli Header --itemType=component
```

So what's the moral of the story here? Make tools, it's cool, and sometimes you can make something that is even more useful than you might anticipate at first ^_^
