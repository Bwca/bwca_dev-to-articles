```ic-metadata
{
  "name": "Implementing React Check Permissions. Part 1: Check Permissions Function",
  "series": {
    "name": "Implementing React Check Permissions",
    "part": 1
  },
  "date": "2021-10-30",
  "lastModifiedDate": "2021-10-30",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "react"],
  "canonicalLink": "https://dev.to/bwca/implementing-reach-check-permissions-part-1-check-permissions-function-136l"
}
```

# Implementing React Check Permissions. Part 1: Check Permissions Function

You can follow the code in this article in the [commit](https://github.com/Bwca/implementing-react-check-permissions/commit/067058c0129a4d64ed73db3ddc354bfed2d1f56d) in the repo I made for the series.

Continuing our series on implementing permission checking tools in React application, in this article we take a look at the function that will drive the whole process.

But first let us create a React application for our check-permissions module to reside. Do not forget to add prettier not to be messy :<

```bash
npx create-react-app implementing-react-check-permissions --template=typescript
```

Now let's create a folder for our permission-validation module with subfolder for our check-permissions function.

```bash
mkdir src/permissions-validation
mkdir src/permissions-validation/check-permissions
```

Here we are going to have three files:

```bash
check-permissions.function.spec.ts
check-permissions.function.ts
index.ts
```

File for the actual function, file for tests and an index file for export convenience.

Its purpose is to validate if use has required permissions and return the result as a boolean value. The function signature of the function will be the following (I am a sucker for typescript, please pardon my weakness):

```typescript
export function checkPermissions(
    currentPermissions: string[],
    requiredPermissions?: string[] | string | null,
    checkAll?: boolean
): boolean;
```

We definitely want to have the current permissions the use has, yet, as for required permissions we'll be handling even cases when there are no required permissions. We also provide a boolean flag `checkAll` for cases when we want to ensure use has one of the required permissions, but not necessary all. By default we'll be checking all required permissions, but we are flexible enough to check just some.

The whole function body is pretty small, under 20 lines of code:

```typescript
// check-permissions.function.ts
export function checkPermissions(
    currentPermissions: string[],
    requiredPermissions?: string[] | string | null,
    checkAll = true
): boolean {
    /** No permissions required, so it's cool, saves us the trouble
        and allows for early exit.
     */
    if (!requiredPermissions) {
        return true;
    }
    /** If there is only one required permission, wrap it in an array
        for further convenience.    
     */
    if (!Array.isArray(requiredPermissions)) {
        requiredPermissions = [requiredPermissions];
    }

    /** Check every or some, dead simple. */
    if (checkAll) {
        return requiredPermissions.every((p) => currentPermissions.includes(p));
    }

    return requiredPermissions.some((p) => currentPermissions.includes(p));
}
```

Now, looking at the actual function you might be asking, why are we using arrays instead of sets, since both `requiredPermissions` and `currentPermissions` are probably always sets of unique values. The reason for using arrays is pretty trivial, the sheer size of arrays holding permissions and required permissions are usually so small that there is little to no benefit bothering converting them to sets.

Who knows, perhaps converting both arrays to sets and checking them could even take more time than iterating over two small arrays. I have not tested it, but I digress.

Good, we now have a function, let us now add some tests to ensure it actually works as we expect.

We will have small four test cases written with AAA methodology for readability:

- checking permissions when there are no required pemissions provided;
- positive check when required permissions are present in current permissions;
- negative check when some required permissions are missing;
- positive check when we check for only one of the required permissions.

So we end up with the following tests file:

```typescript
// check-permissions.function.spec.ts
import { checkPermissions } from "./check-permissions.function";

describe("Testing permission checking function", () => {
    it("Result should be positive if no required permissions provided", () => {
        // Arrange
        const currentPermissions: string[] = [];

        // Act
        const hasPermissions = checkPermissions(currentPermissions);

        // Assert
        expect(hasPermissions).toBeTruthy();
    });

    it("Result should be positive if required permissions are present in current permissions", () => {
        // Arrange
        const requiredPermission = "some-view-permission";
        const currentPermissions = [requiredPermission];

        // Act
        const hasPermissions = checkPermissions(currentPermissions, requiredPermission);

        // Assert
        expect(hasPermissions).toBeTruthy();
    });

    it("Result should be negative if not all required permissions are present", () => {
        // Arrange
        const requiredPermission = ["some-view-permission", "some-other-permission"];
        const currentPermissions = [requiredPermission[0]];

        // Act
        const hasPermissions = checkPermissions(currentPermissions, requiredPermission);

        // Assert
        expect(hasPermissions).toBeFalsy();
    });

    it("Result should be positive if not all required permissions are present when checkAll parameter is set to false", () => {
        // Arrange
        const requiredPermission = ["some-view-permission", "some-other-permission"];
        const currentPermissions = [requiredPermission[0]];

        // Act
        const hasPermissions = checkPermissions(
            currentPermissions,
            requiredPermission,
            false
        );

        // Assert
        expect(hasPermissions).toBeTruthy();
    });
});

```

At this point you might be asking why bother with a function if you can go and create a hook right away. You could start with a hook of course, yet, hooks work only in components, while a function is so universal you can use it anywhere. And we will use it in our hook in the next article of the series :)
