```ic-metadata
{
  "name": "Implementing React Check Permissions. Part 3: with Permissions Component",
  "series": {
    "name": "Implementing React Check Permissions",
    "part": 3
  },
  "date": "2021-11-04",
  "lastModifiedDate": "2024-07-09",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "tutorial", "react"],
  "canonicalLink": "https://dev.to/bwca/implementing-react-check-permissions-the-component-3gjm"
}
```

# Implementing React Check Permissions. Part 3: with Permissions Component

You can follow the code in this article in the [commit](https://github.com/Bwca/implementing-react-check-permissions/commit/8db94681ed7aae6740fddb383de0a3b05fe43a17) in the repo I made for the series.

This is the last article in the series and in this final article we will be taking a look how to create a wrapper component for conditional rendering of components based on user permissions.

Why do we need it? Because it is more convenient to wrap a component in a wrapper than to call a hook in every case we need conditional rendering.

Let's create an interface to represent the props of our future wrapper component in our models folder:

```typescript
// ./permissions-validation/models/with-permissions-props.ts
import { ReactElement } from 'react';

export interface WithPermissionsProps {
  checkAll?: boolean;
  children: ReactElement<string, string>;
  permissions?: string | string[];
  placeholder?: JSX.Element;
}
```

So we still have the usual `permissions` and `checkAll`, along with the `children` for conditional rendering and the `placeholder` which we are going to show in cases when the user has no permissions to view the children.

The wrapper itself therefore is going to be a function receiving these props and returning a JXS element:

```typescript
// ./permissions-validation/models/with-permissions.ts
import { WithPermissionsProps } from './with-permissions-props';

export type WithPermissions = (props: WithPermissionsProps) => JSX.Element;
```

Looking at these models you might understand that we are going to power the permission check in our wrapper with the hook from the previous article, but where is it and how are we going to create it for the wrapper?

Obviously the wrapper should not be trusted with hook creation, it is not its purpose. So we are going to create yet another factory to produce the wrapper component and provide it with the hook to perform necessary permission checks.

```typescript
// ./permissions-validation/create-with-permissions-wrapper/create-with-permissions-wrapper.tsx
import { UseCheckPermissions, WithPermissions, WithPermissionsProps } from '../models';

export function createWithPermissionsWrapper(useCheckPermissions: UseCheckPermissions): WithPermissions {
  return ({ checkAll = true, children, permissions, placeholder = <div /> }: WithPermissionsProps) => {
    const { checkPermissions } = useCheckPermissions();
    return checkPermissions(permissions, checkAll) ? children : placeholder;
  };
}
```

Note how the factory expects a hook. It does not create it, it only ensures this dependency is injected into its produced component.

Of course, we're going to throw some more tests to ensure the wrapper component actually works. Good that we've already installed dependencies for testing hooks:

```typescript
import { render } from '@testing-library/react';

import { createCheckPermissionsHook } from '../create-check-permissions-hook';
import { WithPermissionsProps } from '../models';
import { createWithPermissionsWrapper } from './create-with-permissions-wrapper';

const THE_ONLY_AVAILABLE_PERMISSION = 'some-view-permission';
const WithPermissions = createWithPermissionsWrapper(createCheckPermissionsHook(() => [THE_ONLY_AVAILABLE_PERMISSION]));

describe('Tests for WithPermissions without placeholder', () => {
  it('Should render anything really', () => {
    // Arrange
    const renderProps: WithPermissionsProps = {
      children: <div data-test-id="invisible" />,
      permissions: 'some-view-permission',
    };

    // Act
    const { baseElement } = render(renderWithPermissionsWrapper(renderProps));

    // Assert
    expect(baseElement).toBeTruthy();
  });
  it('Result should be positive if no required permissions provided', () => {
    // Arrange
    const testId = 'child-element';
    const renderProps: WithPermissionsProps = {
      children: <div data-testid={testId} />,
    };

    // Act
    const { queryByTestId } = render(renderWithPermissionsWrapper(renderProps));

    // Assert
    expect(queryByTestId(testId)).toBeTruthy();
  });
  it('Result should be positive if required permissions are present in current permissions', () => {
    // Arrange
    const testId = 'child-element';
    const renderProps: WithPermissionsProps = {
      children: <div data-testid={testId} />,
      permissions: THE_ONLY_AVAILABLE_PERMISSION,
    };

    // Act
    const { queryByTestId } = render(renderWithPermissionsWrapper(renderProps));

    // Assert
    expect(queryByTestId(testId)).toBeTruthy();
  });
  it('Result should be negative if not all required permissions are present', () => {
    // Arrange
    const testId = 'child-element';
    const renderProps: WithPermissionsProps = {
      children: <div data-testid={testId} />,
      permissions: [THE_ONLY_AVAILABLE_PERMISSION, 'some-other-permission'],
    };

    // Act
    const { queryByTestId } = render(renderWithPermissionsWrapper(renderProps));

    // Assert
    expect(queryByTestId(testId)).toBeFalsy();
  });
  it('Result should be positive if not all required permissions are present when checkAll parameter is set to false', () => {
    // Arrange
    const testId = 'child-element';
    const renderProps: WithPermissionsProps = {
      children: <div data-testid={testId} />,
      permissions: [THE_ONLY_AVAILABLE_PERMISSION, 'some-other-permission'],
      checkAll: false,
    };

    // Act
    const { queryByTestId } = render(renderWithPermissionsWrapper(renderProps));

    // Assert
    expect(queryByTestId(testId)).toBeTruthy();
  });
});

describe('Tests for WithPermissions placeholder', () => {
  const placeholderId = 'placeholder-id';
  const placeholder = <div data-testid={placeholderId} />;

  it('Placeholder is not visible if no required permissions provided', () => {
    // Arrange
    const renderProps: WithPermissionsProps = {
      children: <div  />,
      placeholder,
    };

    // Act
    const { queryByTestId } = render(renderWithPermissionsWrapper(renderProps));

    // Assert
    expect(queryByTestId(placeholderId)).toBeFalsy();
  });
  it('Placeholder is not visible if required permissions are present in current permissions', () => {
    // Arrange
    const renderProps: WithPermissionsProps = {
      children: <div />,
      permissions: THE_ONLY_AVAILABLE_PERMISSION,
      placeholder
    };

    // Act
    const { queryByTestId } = render(renderWithPermissionsWrapper(renderProps));

    // Assert
    expect(queryByTestId(placeholderId)).toBeFalsy();
  });
  it('Placeholder is visible if not all required permissions are present', () => {
    // Arrange
    const renderProps: WithPermissionsProps = {
      children: <div />,
      permissions: [THE_ONLY_AVAILABLE_PERMISSION, 'some-other-permission'],
      placeholder
    };

    // Act
    const { queryByTestId } = render(renderWithPermissionsWrapper(renderProps));

    // Assert
    expect(queryByTestId(placeholderId)).toBeTruthy();
  });
  it('Placeholder is not visible if not all required permissions are present when checkAll parameter is set to false', () => {
    // Arrange
    const renderProps: WithPermissionsProps = {
      children: <div />,
      permissions: [THE_ONLY_AVAILABLE_PERMISSION, 'some-other-permission'],
      checkAll: false,
      placeholder
    };

    // Act
    const { queryByTestId } = render(renderWithPermissionsWrapper(renderProps));

    // Assert
    expect(queryByTestId(placeholderId)).toBeFalsy();
  });
});

function renderWithPermissionsWrapper(props: WithPermissionsProps): JSX.Element {
  return <WithPermissions {...props}></WithPermissions>;
}

```

At this point our check permissions module is almost complete. We have a factory to produce a hook and a factory to produce a wrapper component. Yet, it would look a bit messy if we just exported these factories. We would rely on consumers creating these items in certain order, i.e. hook and then component.

So for convenience we can create yet another factory, which is going to be the only exported member from our check permissions module.

Overall the only this we need from our consumers is to provide us with a function to get an array of user's current permissions, from that point we're good to go. We can create a hook and wrapper and give them back.

So our final factory:

```typescript
// ./permissions-validation/create-permission-checkers/create-permission-checkers.ts
import { createCheckPermissionsHook } from '../create-check-permissions-hook';
import { createWithPermissionsWrapper } from '../create-with-permissions-wrapper';
import { GetPermissions, UseCheckPermissions, WithPermissions } from '../models';

export function createPermissionCheckers(fun: GetPermissions): PermissionCheckers {
  const useCheckPermissions = createCheckPermissionsHook(fun);
  const withPermissions = createWithPermissionsWrapper(useCheckPermissions);
  return {
    useCheckPermissions,
    WithPermissions: withPermissions,
  };
}

interface PermissionCheckers {
  useCheckPermissions: UseCheckPermissions;
  WithPermissions: WithPermissions;
}

```

Now to add the index file for the public api of our module to expose only this factory to the outer world:

```typescript
// ./permissions-validation/index.ts
export { createPermissionCheckers } from "./create-permission-checkers";
```

What's cool about this way is that we are completely oblivious of where do permissions come from. We don't care how they are stored and were.

That's it, we've implemented a [permission validation module for React](https://www.npmjs.com/package/@merry-solutions/react-check-permissions) and covered it with tests.
