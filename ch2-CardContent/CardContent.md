# CardContent

This is what our component might look like eventually. We need a a div wrapping two other divs.

![card-content](../img/card-content.png)

Create a branch `feat/cardContent`. Create `src/components/CardContent` folder and 2 files in it; `CardContent.cy.tsx`, `CardContent.tsx`.

We start minimal with a test that renders the component (Red 1).

```tsx
// src/components/CardContent.cy.tsx
import CardContent from "./CardContent";

describe("CardContent", () => {
  it("should", () => {
    cy.mount(<CardContent />);
  });
});
```

We do the minimal to make the compiler green.

```tsx
// src/components/CardContent.tsx

export default function CardContent() {
  return <div>hello</div>;
}
```

Start the Cypress component test runner and execute the test; `yarn cy:open-ct`.

Let's test that the string renders (Refactor 1).

```tsx
// src/components/CardContent.cy.tsx
import CardContent from "./CardContent";

describe("CardContent", () => {
  it("should", () => {
    cy.mount(<CardContent />);
    cy.contains("div", "hello");
  });
});
```

![CardContent-hello](../img/CardContent-hello.png)

Let's write a test for the divs we want; we need two divs, one contains a name, the other a description (Red 2).

```tsx
// src/components/CardContent.cy.tsx
import CardContent from "./CardContent";

describe("CardContent", () => {
  it("should", () => {
    cy.mount(<CardContent />);

    cy.contains("div", "Bjorn Ironside");
    cy.contains("div", "king of 9th century Sweden");
  });
});
```

We make it green by hard-coding the values being tested for (Green 2).

```tsx
// src/components/CardContent.tsx
export default function CardContent() {
  return (
    <div>
      <div>Bjorn Ironside</div>
      <div>king of 9th century Sweden</div>
    </div>
  );
}
```

It is becoming obvious we should be passing name and description as props (Red 3).

```tsx
// src/components/CardContent.cy.tsx
import CardContent from "./CardContent";

describe("CardContent", () => {
  it("should", () => {
    const name = "Bjorn Ironside";
    const description = "king of 9th century Sweden";
    cy.mount(<CardContent name={name} description={description} />);

    cy.contains("div", name);
    cy.contains("div", description);
  });
});
```

The test still passes, but the compiler is complaining about the props that don not exist. Let's add those to the component (Green 3). We can also add the types for these props since we know they will both be strings.

```tsx
// src/components/CardContent.tsx
type ButtonFooterProps = {
  name: string;
  description: string;
};

export default function CardContent({ name, description }: ButtonFooterProps) {
  return (
    <div>
      <div>{name}</div>
      <div>{description}</div>
    </div>
  );
}
```

Now we can add some styles to our component. Create a file `src/index.css` and paste the below css in.

```css
.menu-list .active-link,
.menu-list .router-link-active {
  color: #fff;
  background-color: #00b3e6;
}
.not-found i {
  font-size: 20px;
  margin-right: 8px;
}
.not-found .title {
  letter-spacing: 0px;
  font-weight: normal;
  font-size: 24px;
  text-transform: none;
}
header {
  font-weight: bold;
  font-family: Arial;
}
header span {
  letter-spacing: 0px;
}
header span.tour {
  color: #fff;
}
header span.of {
  color: #ccc;
}
header span.heroes {
  color: #61dafb;
}
header .navbar-item.nav-home {
  border: 3px solid transparent;
  border-radius: 0%;
}
header .navbar-item.nav-home:hover {
  border-right: 3px solid #61dafb;
  border-left: 3px solid #61dafb;
}
header .fab {
  font-size: 24px;
}
header .fab.js-logo {
  color: #61dafb;
}
header .buttons i.fab {
  color: #fff;
  margin-left: 20px;
  margin-right: 10px;
}
header .buttons i.fab:hover {
  color: #61dafb;
}
.edit-detail .input[readonly] {
  background-color: #fafafa;
}
.content-title-group {
  margin-bottom: 16px;
}
.content-title-group h2 {
  border-left: 16px solid #00b3e6;
  border-bottom: 2px solid #00b3e6;
  padding-left: 8px;
  padding-right: 16px;
  display: inline-block;
  text-transform: uppercase;
  color: #555;
  letter-spacing: 0px;
}
.content-title-group h2:hover {
  color: #00b3e6;
}
.content-title-group button.button {
  border: 0;
  color: #999;
}
.content-title-group button.button:hover {
  color: #00b3e6;
}
ul.list {
  box-shadow: none;
}
div.card-content {
  background-color: #fafafa;
  background-color: #fafafa;
}
div.card-content .name {
  font-size: 28px;
  color: #000;
}
div.card-content .description {
  font-size: 20px;
  color: #999;
}
.card {
  margin-bottom: 2em;
}
label.label {
  font-weight: normal;
}
p.card-header-title {
  background-color: #00b3e6;
  text-transform: uppercase;
  letter-spacing: 4px;
  color: #fff;
  display: block;
  padding-left: 24px;
}
.card-footer button {
  font-size: 16px;
  color: #888;
}
.card-footer button i {
  margin-right: 10px;
}
.card-footer button:hover {
  color: #00b3e6;
}
.modal-card-foot button {
  display: inline-block;
  width: 80px;
}
.modal-card-head,
.modal-card-body {
  text-align: center;
}
.field {
  margin-bottom: 0.75rem;
}
.navbar-burger {
  margin-left: auto;
}
button.link {
  background: none;
  border: none;
  cursor: pointer;
}
```

Our component is still looking the same in the Cypress runner. Let's add some classes to make it look nicer (Refactor 3).

```tsx
// src/components/CardContent.tsx
type ButtonFooterProps = {
  name: string;
  description: string;
};

export default function CardContent({ name, description }: ButtonFooterProps) {
  return (
    <div className="card-content">
      <div className="name">{name}</div>
      <div className="description">{description}</div>
    </div>
  );
}
```

This finalizes our work with the component.

![CardContent-final](../img/CardContent-final.png)



## Summary

Here are the Red Green Refactor cycles we went through while creating the component.

<br />

We started with a minimal test that renders the component (Red 1)

We added the function component to make it green (Green1)

We added a test to `cy.contains("div", "hello")` verify the render (Refactor 1)

<br />

We wrote a failing test for the divs (Red 2).

We hard-coded the values into the divs that we created for the component (Green2).

<br />

We enhanced the test to render with props instead of hard-coded values (Red 3).

We added the props and their types to the component (Green 3).

We added styles and classes to the component (Refactor 3).

<br />

A PR with these changes can be found [here](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/pull/2).

// TODO: once styles are decided, consider adding the CSS to the setup / intro section.
