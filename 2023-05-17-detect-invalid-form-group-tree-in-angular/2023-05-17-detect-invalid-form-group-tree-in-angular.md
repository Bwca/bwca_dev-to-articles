```ic-metadata
{
  "name": "Detect Invalid FormGroup Tree in Angular",
  "series": null,
  "date": "2023-05-17",
  "lastModifiedDate": "2023-05-17",
  "author": "Volodymyr Yepishev",
  "tags": ["angular", "debugger"],
  "canonicalLink": "https://dev.to/bwca/detect-invalid-formgroup-tree-in-angular-1ed"
}
```

# Detect Invalid FormGroup Tree in Angular

In this article I am sharing the product of my laziness.

TL/DR source code and small demo can be found [here](https://github.com/Bwca/script_detect-invalid-form-group-in-angular).

Eventually every frontend developer who works with forms faces a problem of an invalid form. A classic scenario would be a bug from QAs with submit button being disabled for no apparent reason. You go to the code to find out the condition for the submit button to be disabled is the form being invalid. 

Reasonable, now why is it invalid? It is invalid because one of its controls did not pass validation, but the fun begins when the form has nested form groups. Essentially you are facing a tree and need to find an infected leaf from which the disease spreads to the trunk.

Imagine the following interface representing an object in form, for the sake of simplicity let's imagine all fields are required (I had ChatGPT come up with the interface, got to take advantage of the AI before they rebel):

```typescript
interface Person {
  name: string;
  age: number;
  gender: string;
  contact: {
    phone: string;
    address: {
      street: string;
      city: string;
      state: string;
      postalCode: string;
    };
  };
  education: {
    degree: string;
    university: {
      name: string;
      location: {
        country: string;
        city: string;
      };
    };
  };
  employment: {
    company: string;
    position: string;
    department: {
      name: string;
      supervisor: {
        name: string;
        email: string;
      };
    };
  };
  hobbies: string[];
  friends: Person[];
}
```

Now, an invalid `email` invalidates `supervisor`, which invalidates `department`, which invalidates `employment`, which invalidates the whole object. 

And debugging the problem you'd start from top and go down to the `email`. This can be quite boring, so I decided to write a script to automate it.

The general idea is: knowing the selector of a component that has a form group, it should find the form group in it (developer is too important to be bothered), then it should traverse all its controls, and their child controls, going recursively until all controls are visited, creating a tree of invalid controls and dropping it into console for inspection.

What's actually cool about it, is that such script can be made a [bookmarklet](https://en.wikipedia.org/wiki/Bookmarklet), meaning you store it as a bookmark and run when you need it. The only drawback being, it relies on Angular debug object, which is available only in developer mode.

So I came up wit the following: click the bookmarklet, enter component selector, and have a tree of invalid form controls presented in the browser console for easy debug.

```javascript
javascript: (() => {
  try {
    const componentSelector = prompt(
      'Please enter component selector to check invalid form groups'
    );
    const domeElement = document.querySelector(componentSelector);
    const invalidTree = Object.entries(ng.getComponent(domeElement))
      .filter((i) => !!i[1].controls)
      .reduce((a, b) => ({ ...a, [b[0]]: findAllInvalidControls(b) }), {});
    console.log(invalidTree);
  } catch (e) {
    console.error('Could not check form groups for the following reason: ', e);
  }

  function findAllInvalidControls(fg) {
    return (fg[1].controls ? Object.entries(fg[1].controls) : [fg])
      .reduce((a, b) => {
        const item = { name: b[0], control: b[1] };
        /* Hit FormArrayControl */ 
        if (Number.isInteger(+item.name)) {
          item.name = `${fg[0]}[${item.name}]`;
        }
        if (item.control.invalid) {
          if (item.control.controls) {
            item.nestedControls = Object.entries(item.control.controls)
              .map(findAllInvalidControls)
              .filter((i) => Object.values(i).length)
              .reduce((a, b) => {
                const [name, value] = Object.entries(b)[0];
                return { ...a, [name]: value };
              }, {});
          }
          a.push(item);
        }
        return a;
      }, [])
      .reduce(
        (a, b) => ({
          ...a,
          [b.name]: b.nestedControls
            ? { control: b.control, nestedControls: b.nestedControls }
            : b.control,
        }),
        {}
      );
  }
})();
```



You can do pretty amazing things with javascript :)