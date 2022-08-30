We can also refactor the test to use less magic values (Refactor 7). We have no network yet to stub, but we have a duplicate `heroes.json` file in Cypress fixtures which we can use.

```tsx
// cypress/e2e/edit-hero.cy.ts
import heroes from "../fixtures/heroes.json";

describe("Edit hero", () => {
  beforeEach(() => cy.visit("/"));
  it("should go through the cancel flow", () => {
    cy.location("pathname").should("eq", "/heroes");

    const heroIndex = 0;

    cy.getByCy("edit-button").eq(heroIndex).click();
    cy.location("pathname").should("eq", "/heroes/edit-hero");
    cy.getByCy("hero-detail").should("be.visible");
    cy.getByCy("input-detail-id").should("be.visible");
    cy.findByDisplayValue(heroes[heroIndex].id).should("be.visible");
  });
});
```

Now, let's change that index to something other than 0, and we have our failing test. The displayed hero should now be array position 1, but it is still the hero at index 0 (Red 8).

```tsx
// cypress/e2e/edit-hero.cy.ts
import heroes from "../fixtures/heroes.json";

describe("Edit hero", () => {
  beforeEach(() => cy.visit("/"));
  it("should go through the cancel flow", () => {
    cy.location("pathname").should("eq", "/heroes");

    const heroIndex = 1;

    cy.getByCy("edit-button").eq(heroIndex).click();
    cy.location("pathname").should("eq", "/heroes/edit-hero");
    cy.getByCy("hero-detail").should("be.visible");
    cy.getByCy("input-detail-id").should("be.visible");
    cy.findByDisplayValue(heroes[heroIndex].id).should("be.visible");
  });
});
```

We need a way for the route to edit the nth hero. In `react-router` we can take advantage of path attributes.

Here is a simple example showing how path attributes work. Assume that our data is `milkshake` and the data model looks as such:

```tsx
{
  flavor: "vanilla",
  size: "medium"
}
```

If we setup our rotes like this:

```tsx
<Route path="/milkshake/:flavor/:size" element={<Milkshake />} />
```

The url path will be ` /milkshake/vanilla/medium`.

To replicate that configuration, our `edit-hero` path needs a path attribute of `id`, and we need a way to extract that path attribute from the url. React-router's `useParam` returns an object with properties corresponding to URL parameters.

```tsx
const { flavor, size } = useParams();
```
