# HeaderBar

In the Angular version of the app, there is a parent component above `HeaderBarBrand`. Let's create it too.

![HeaderBar-initial](../img/HeaderBar-initial.png)

Create a branch `feat/headerBar`. Create 2 files under `src/components/` folder; `HeaderBar.cy.tsx`, `HeaderBar.tsx`. As usual, start minimal with a component rendering; copy the below to the files and execute the test after opening the runner with `yarn cy:open-ct`.

```tsx
// src/components/HeaderBar.cy.tsx
import HeaderBar from "./HeaderBar";

describe("HeaderBar", () => {
  it("should", () => {
    cy.mount(<HeaderBar />);
  });
});
```

```tsx
// src/components/HeaderBar.tsx

export default function HeaderBar() {
  return <div>hello</div>;
}
```

In `HeaderBar` component, which is the child of `HeaderBar`, we added a top level `data-cy` selector with the value of `header-bar-brand`. Let's add a failing test to check the existence of the child component (Red 1).

```tsx
// src/components/HeaderBar.cy.tsx
import HeaderBar from "./HeaderBar";

describe("HeaderBar", () => {
  it("should", () => {
    cy.mount(<HeaderBar />);
    cy.getByCy("header-bar-brand");
  });
});
```

```tsx
// src/components/HeaderBar.tsx
import HeaderBarBrand from "./HeaderBarBrand";

export default function HeaderBar() {
  return (
    <div>
      <HeaderBarBrand />
    </div>
  );
}
```

We get the same `react-router` related error that we got with the `HeaderBarBrand` component. If a child component is using `react-router`, when testing the parent component, we also have to wrap the parent in `BrowserRouter` (Green 1).

```tsx
// src/components/HeaderBar.cy.tsx
import HeaderBar from "./HeaderBar";
import { BrowserRouter } from "react-router-dom";

describe("HeaderBar", () => {
  it("should", () => {
    cy.mount(
      <BrowserRouter>
        <HeaderBar />
      </BrowserRouter>
    );
    cy.getByCy("header-bar-brand");
  });
});
```

The child component renders.

![HeaderBar-Green1](../img/HeaderBar-Green1.png)

The specification shows that the child component is wrapped by a `header` and a `nav` There is also some css that gives it the dark theme. We can copy those off of the browser.

```tsx
// src/components/HeaderBar.tsx
import HeaderBarBrand from "./HeaderBarBrand";

export default function HeaderBar() {
  return (
    <header>
      <nav
        className="navbar has-background-dark is-dark"
        role="navigation"
        aria-label="main navigation"
      >
        <HeaderBarBrand />
      </nav>
    </header>
  );
}
```

The component is looking great with CSS.

![HeaderBar-css](../img/HeaderBar-css.png)

Realize that we did not necessarily go through a traditional RedGreenRefactor cycle; our Red in this case was the component not rendering in its final form and we kept improving that step by step.

What else can be tested at this point? We do not need to repeat the tests for the child component, because we covered all of it at the child. The only valuable additional check would be to check for an attribute of the `nav` tag. We can easily add that with [Cypress testing library](https://testing-library.com/docs/cypress-testing-library/intro/) (Refactor 1).

```tsx
// src/components/HeaderBar.cy.tsx
import HeaderBar from "./HeaderBar";
import { BrowserRouter } from "react-router-dom";
import "../styles.scss";

describe("HeaderBar", () => {
  it("should", () => {
    cy.mount(
      <BrowserRouter>
        <HeaderBar />
      </BrowserRouter>
    );
    cy.getByCy("header-bar-brand");
    cy.findByRole("navigation");
  });
});
```

## Summary & takeaways

The main takeaway in this section is the parent-child relationship between the components.

We can add a `data-cy` attribute to the top tag of the component to make it easier to reference.

If a child component is using `react-router`, when testing the parent component, we also have to wrap the parent in `BrowserRouter`.

When designing the component, we do not necessarily have to go through a traditional RedGreenRefactor cycle. When our test tool is also the design tool, our RGF can as well be incremental visual enhancements to the component whilst adding classes and css.