```ic-metadata
{
  "name": "Remove Double Imports with Regular Expression",
  "series": null,
  "date": "2022-03-29",
  "lastModifiedDate": "2022-03-29",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "react"],
  "canonicalLink": "https://dev.to/bwca/remove-double-imports-with-regular-expression-47af"
}
```

# Remove Double Imports with Regular Expression

One thing I learned about imports is that they can become really messy when dealing with git conflicts. The more imports there is, the more time is required to sort them out. Imagine merging a branch and ending with something like:

```typescript
import { Baz } from 'baz';
import { Foo } from 'foo';

import { 
    BOHICA, 
    FUBAR, 
    FUBU, 
    SNAFU, 
    SUSFU, 
    TARFU } from 'slang';

import { Baz } from 'baz';
import { 
    BOHICA, 
    FUBAR, 
    FUBU, 
    SNAFU, 
    SUSFU, 
    TARFU } from 'slang';
    
import { Baz } from 'baz';
```

That's over 20 lines, nothing critical, but once you need to split screen/scroll to remove duplicates, it's no fun. Need some superpowers to remove duplicates. You just need to use ~~ancient elven magic~~  regular expression and an IDE with replace function that supports it.

Check out this ~~spell~~:
```regex
(import [^;]+;\n)(?=(.*\n)*\1)
```

Paste it into search field, enable regex and run replace, whoosh! All import duplicates are gone, you might have some extra blank lines, but a prettier will fix that with a single hotkey comman.

```typescript
import { Foo } from 'foo';

import { 
    BOHICA, 
    FUBAR, 
    FUBU, 
    SNAFU, 
    SUSFU, 
    TARFU } from 'slang';
    
import { Baz } from 'baz';
```

Pretty neat, eh?