```ic-metadata
{
  "name": "Implementing React Check Permissions. Part 2: Check Permissions Hook",
  "series": {
    "name": "Implementing React Check Permissions",
    "part": 2
  },
  "date": "2021-11-01",
  "lastModifiedDate": "2021-11-01",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "react"],
  "canonicalLink": "https://dev.to/bwca/implementing-react-check-permissions-the-hook-3m20"
}
```

# Implementing React Check Permissions. Part 2: Check Permissions Hook

You can follow the code in this article in the [commit](https://github.com/Bwca/implementing-react-check-permissions/commit/3209059e00190392b838913d15c6fc62f4f7ba34) in the repo I made for the series.

Continuing our series on implementing permission checking tools in React application, in this article we take a look at the check permissions hook and how to implement it.

The hook is actually the place where the logic of obtaining current permissions can be placed, then it can expose a method for checking permissions in a way that does not require components to bother with getting active user permissions from the profile or whatever.

Let us create a folder called `models` and place the types for our future hook:

```typescript
// ./permissions-validation/models/use-check-permissions.ts
export type UseCheckPermissions = () => UseCheckPermissionsMethods;

export interface UseCheckPermissionsMethods {
    checkPermissions: (
        permissions?: string[] | string | null,
        checkAll?: boolean
    ) => boolean;
}
```

Our future hook is going to be of type `UseCheckPermissions`, a function that accepts no parameters, but returns an object with a method for permission validation.

At this point you might start thinking how do are we supposed to design our hook for obtaining active permissions when we have neither a user profile nor a slightest idea how and where these current permissions are going to be stored. The best part about it is that we don't have to know. Otherwise our `permission-validation` module would become coupled with permission storing mechanism used in the application. This is something we should, can and will avoid.

Functional approach and factory method come to the rescue here. Instead of actually implementing a hook which would know a way how to obtain current permissions, we will make a factory to produce it and pass a function for retrieving current permissions to it. This way the hook will have no idea where do permissions come from, which is great.

So let's add a type for a function that would give us current user permissions:

```typescript
// ./permissions-validation/models/get-permissions.ts
export type GetPermissions = () => string[];
```

Now an index file in models folder for the convenience of exports and we are ready to build our hook factory!

```typescript
// ./permissions-validation/models/index.ts
export * from "./get-permissions";
export * from "./use-check-permissions";
```

Our hook factory is going to live in `create-check-permissions-hook` folder next to an index file for exports and a file with tests.

```typescript
// ./permissions-validation/create-check-permissions-hook/create-check-permissions-hook.function.ts
import { checkPermissions } from "../check-permissions";
import { GetPermissions, UseCheckPermissions } from "../models";

export function createCheckPermissionsHook(
    getCurrentPermissions: GetPermissions
): UseCheckPermissions {
    return () => ({
        checkPermissions: (
            permissions?: string[] | string | null,
            checkAll = true
        ): boolean => checkPermissions(getCurrentPermissions(), permissions, checkAll),
    });
}
```

So we expect to be given a function for obtaining current user permissions and return a hook exposing `checkPermissions` method, which in its term invokes the `checkPermissions` function from the previous article.

To ensure everything works as expected, we can now add some test cases, which are basically a copy of `checkPermissions` function tests, but altered so they apply to our hook. Note, that in order to test hooks we are going to need a special package, `@testing-library/react-hooks/dom`.

```typescript
// ./permissions-validation/create-check-permissions-hook/create-check-permissions-hook.function.spec.ts
import { renderHook } from "@testing-library/react-hooks/dom";

import { createCheckPermissionsHook } from "./create-check-permissions-hook.function";

describe("Tests for createCheckPermissionsHook factory and its hook", () => {
    let checkPermissions: (
        permissions?: string[] | string | null,
        checkAll?: boolean
    ) => boolean;

    beforeEach(() => {
        const { result } = renderHook(
            createCheckPermissionsHook(() => ["some-view-permission"])
        );
        checkPermissions = result.current.checkPermissions;
    });

    it("The hook should be created", () => {
        expect(checkPermissions).toBeTruthy();
    });

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

        // Act
        const hasPermissions = checkPermissions(requiredPermission);

        // Assert
        expect(hasPermissions).toBeTruthy();
    });

    it("Result should be negative if not all required permissions are present", () => {
        // Arrange
        const requiredPermission = ["some-view-permission", "some-other-permission"];

        // Act
        const hasPermissions = checkPermissions(requiredPermission);

        // Assert
        expect(hasPermissions).toBeFalsy();
    });

    it("Result should be positive if not all required permissions are present when checkAll parameter is set to false", () => {
        // Arrange
        const requiredPermission = ["some-view-permission", "some-other-permission"];

        // Act
        const hasPermissions = checkPermissions(requiredPermission, false);

        // Assert
        expect(hasPermissions).toBeTruthy();
    });
});

```

The function powering the hook, which in its turn is going to be powering the wrapper component we will be creating in the next article :)
