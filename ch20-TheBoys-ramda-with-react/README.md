# Ramda with React

Before starting this section, make sure to go through the [prerequisite](./prerequisite.md) where were mirror `Heroes` to `Boys`.

## Why Functional Programming, why Ramda?

If we take a look at the recent proposals to EcmaScript, we will realize that FP is the way and that new JS features will closely resemble [RamdaJS](https://ramdajs.com/docs/) utilities. Check out [So what's new in ES2025](https://www.youtube.com/watch?v=FaQA0qU5e6o) for a premier. We believe that learning FP and Ramda today will build a future-proof JS skillset, at least for the next decade.

We have a React app with solid tests built by TDD at all levels and 100% code coverage. In the prerequisites we mirrored `Heroes` group of entities to `Boys`. Now we can try out daring refactors and see if things still work.

While the literature on FP is plenty, there is a lack of real world examples of using FP and Ramda, especially with modern React and TypeScript. We are hopeful that this section addresses that gap.

## Functional Programming JS resources

Here are the resources that inspired this work. You do not have to be fluent in FP to benefit from this section, but if you choose to back-fill your knowledge at a later time, these resources are highly recommended.

- [Functional Light JS](https://www.manning.com/books/functional-light-javascript "https://www.manning.com/books/functional-light-javascript")
- [FP in JS](https://www.amazon.com/Functional-Programming-JavaScript-functional-techniques-ebook-dp-B09781W9HY/dp/B09781W9HY/ref=mt_other?_encoding=UTF8&me=&qid= "https://www.amazon.com/Functional-Programming-JavaScript-functional-techniques-ebook-dp-B09781W9HY/dp/B09781W9HY/ref=mt_other?_encoding=UTF8&me=&qid=")
- [Prof. Frisbie's Mostly Adequate Guide to FP](https://mostly-adequate.gitbook.io/mostly-adequate-guide/ "https://mostly-adequate.gitbook.io/mostly-adequate-guide/")
- [FP patterns with Ramda](https://www.educative.io/courses/functional-programming-patterns-with-ramdajs/YQV9QG6gqz9)
- [JS Allonge](https://leanpub.com/javascriptallongesix/read "https://leanpub.com/javascriptallongesix/read")
- [Composing Software](https://leanpub.com/composingsoftware)
- [Functional JavaScript](https://www.amazon.com/gp/product/1449360726)

## Install the Ramda package

```bash
yarn add ramda ramda-adjunct
yarn add -D @types/ramda
```

In the following sections we will give practical examples of using Ramda, then apply the knowledge to 3 Boys components; `BoyDetail.tsx`, `BoyList.tsx` and `Boys.tsx`.

## [`partial`](https://ramdajs.com/docs/#partial)

Create any function with `n` arguments and wrap it with `partial` . Declare which args are pre-packaged, and the rest of the args will be waited for. In simple terms; if you have a function with five parameters, and you supply three of the arguments, you end up with a function that expects the last two.

```typescript
// standalone example, copy paste anywhere
import { partial, partialObject, partialRight } from "ramda";

//// example 1
// create any function
const multiply = (a: number, b: number) => a * b;
// wrap it with partial, declare the pre-packaged args
const double = partial(multiply, [2]);

// the function waits for execution,
// executes upon the second arg being passed
double; // [λ]
double(3); // 6

//// example 2
// create any function
const greet = (
  salutation: string,
  title: string,
  firstName: string,
  lastName: string
) => salutation + ", " + title + " " + firstName + " " + lastName + "!";

// wrap it with partial, declare the pre-packaged args
const sayHello = partial(greet, ["Hello"]);
// the function waits for 3 arguments to be passed before executing
sayHello; // [λ]
sayHello("Ms", "Jane", "Jones"); // Hello, Ms Jane Jones!

// another variety, has 2 pre-packaged args
const sayHelloMs = partial(greet, ["Hello", "Ms"]);
// waits for 2 more args before executing
sayHelloMs; // // [λ]
sayHelloMs("Jane", "Jones"); // Hello, Ms Jane Jones!
```

Where can `partial` apply in the React world? Any event handler, where an anonymous function returns a named function with an argument. The below two are the same:

```typescript
// src/boys/BoyDetail.tsx

const handleCancel = () => navigate("/boys");
const handleCancel = partial(navigate, ["/boys"]);
```

`handleCancel` will wait for the click event - ` onClick={handleCancel}` - and navigate to `/boys` route.

Similar applications of partial are in `Boys.tsx`

```typescript
// src/boys/Boys.tsx
import { partial } from "ramda";

const addNewBoy = () => navigate("/boys/add-boy");
const addNewBoy = partial(navigate, ["/boys/add-boy"]);

const handleRefresh = () => navigate("/boys");
const handleRefresh = partial(navigate, ["/boys"]);
```

## [`curry`](https://ramdajs.com/docs/#curry)

No FP premier is complete without curry. We will cover a simple example here, and recall how we used classic curry before.

```typescript
// standalone example, copy paste anywhere
import { compose, curry } from "ramda";

const add = (x: number, y: number) => x + y;

const result = add(2, 3);
result; //? 5

// What happens if you don’t give add its required parameters?
// What if you give too little?
// add(2) // NaN

// So we curry: return a new function per expected parameter
const addCurried = curry(add); // just wrap the function in curry
const addCurriedClassic = (x: number) => (y: number) => x + y;

addCurried(2); // waits to be executed
addCurried(2)(3); //? 5
addCurriedClassic(2); // waits to be executed
addCurriedClassic(2)(3); //? 5

const add3 = (x: number, y: number, z: number) => x + y + z;
const greetFirstLast = (greeting: string, first: string, last: string) =>
  `${greeting}, ${first} ${last}`;

const add3CurriedClassic = (x: number) => (y: number) => (z: number) =>
  x + y + z;
const greetCurriedClassic =
  (greeting: string) => (first: string) => (last: string) =>
    `${greeting}, ${first} ${last}`;
const add3CurriedR = curry(add3); // just wrap the function
const greetCurriedR = curry(greetFirstLast); // just wrap the function

add3CurriedClassic(2)(3); // waiting to be executed
add3CurriedClassic(2)(3)(4); //?
// add3CurriedClassic(2, 3, 4) // doesn't work

greetCurriedClassic("hello"); // waiting to be executed
greetCurriedClassic("hello")("John")("Doe"); //?
// greetCurriedClassic('hello', 'John', 'Doe') // doesn't work

add3CurriedR(2)(3); // waiting to be executed
add3CurriedR(2)(3)(4); //? 9
// the advantage of ramda is flexible arity
add3CurriedR(2, 3, 4); //?

greetCurriedR("hello", "John"); // waiting to be executed
greetCurriedR("hello")("John")("Doe"); //? hello, John Doe
// flexible arity with ramda!
greetCurriedR("hello", "John", "Doe"); //? hello, John Doe
```

How is currying used in the React world? You will recall that we used currying in `Heroes.tsx`, `HeroesList.tsx`, `VillianList.tsx` `VillainList.tsx` components. We did not use Ramda curry, because it would be too complex at that time. You can optionally change them now.

```tsx
// src/heroes/Heroes.tsx
import { curry } from "ramda";

// currying: the outer fn takes our custom arg and returns a fn that takes the event
const handleDeleteHero = (hero: Hero) => () => {
  setHeroToDelete(hero);
  setShowModal(true);
};
// we can use Ramda curry instead, we have to pass the unused event argument though
const handleDeleteHero = curry((hero: Hero, e: React.MouseEvent) => {
  setHeroToDelete(hero);
  setShowModal(true);
});
```

```tsx
// src/heroes/HeroList.tsx
import { curry } from "ramda";

// currying: the outer fn takes our custom arg and returns a fn that takes the event
const handleSelectHero = (heroId: string) => () => {
  const hero = deferredHeroes.find((h: Hero) => h.id === heroId);
  navigate(
    `/heroes/edit-hero/${hero?.id}?name=${hero?.name}&description=${hero?.description}`
  );
};
// we can use Ramda curry instead, we have to pass the unused event argument though
const handleSelectHero = curry(
  (heroId: string, e: MouseEvent<HTMLButtonElement>) => {
    const hero = deferredHeroes.find((h: Hero) => h.id === heroId);
    navigate(
      `/heroes/edit-hero/${hero?.id}?name=${hero?.name}&description=${hero?.description}`
    );
  }
);
```

Let's make sure to use Ramda curry in the `BoysList.tsx` components for sure.

```tsx
// src/boys/BoyList.tsx
import { curry } from "ramda";

// currying: the outer fn takes our custom arg and returns a fn that takes the event
const handleSelectBoy = (boyId: string) => () => {
  const boy = deferredBoys.find((b: Boy) => b.id === boyId);
  navigate(
    `/boys/edit-boy/${boy?.id}?name=${boy?.name}&description=${boy?.description}`
  );
};
// we can use Ramda curry instead, we have to pass the unused event argument though
const handleSelectBoy = curry(
  (boyId: string, e: MouseEvent<HTMLButtonElement>) => {
    const boy = deferredBoys.find((b: Boy) => b.id === boyId);
    navigate(
      `/boys/edit-boy/${boy?.id}?name=${boy?.name}&description=${boy?.description}`
    );
  }
);
```

## [`ifElse`](https://ramdajs.com/docs/#ifElse)

Takes 3 functions; predicate, truthy-result, false-result. The advantage over classic if else or ternary operator is that we can get very creative with function expressions vs conditions inside an if statement, or a ternary value. Easily explained with an example:

```typescript
// standalone example, copy paste anywhere
const hasAccess = false; // toggle this to see the difference

// classic if else
function logAccessClassic(hasAccess: boolean) {
  if (hasAccess) {
    return "Access granted";
  } else {
    return "Access denied";
  }
}
logAccessClassic(hasAccess); // Access granted | Access denied

// ternary operator
const logAccessTernary = (hasAccess: boolean) =>
  hasAccess ? "Access granted" : "Access denied";
logAccessTernary(hasAccess); // // Access granted | Access denied

// Ramda ifElse
// The advantage is that you can package the logic away into function
// we can get very creative with function expressions
// vs conditions inside an if statement, or a ternary value
const logAccessRamda = ifElse(
  (hasAccess: boolean) => hasAccess,
  () => "Access granted",
  () => "Access denied"
);

logAccessRamda(hasAccess); // Access granted | Access denied
```

Where can `ifElse` apply in the React world? Anywhere where there is complex conditional logic, which might be hard to express with classic if else statements or ternary operators. The example in `BoyDetail.tsx` is simple, but good for practicing Ramda `ifElse`.

```typescript
// src/boys/BoyDetail.tsx
import { ifElse } from "ramda";
import { isTruthy } from "ramda-adjunct";

const handleSave = () => (name ? updateBoy(boy as Boy) : createBoy(boy as Boy));

const handleSave = ifElse(
  () => isTruthy(name),
  () => updateBoy(boy as Boy),
  () => createBoy(boy as Boy)
);
```

Here is the `BoyDetail.tsx` component after the above changes. It can be compared to `HeroDetail.tsx` side by side to see the distinction between using Ramda and classic array methods.

```tsx
// src/boys/BoyDetail.tsx
import { useState, ChangeEvent } from "react";
import { useNavigate, useParams } from "react-router-dom";
import { FaUndo, FaRegSave } from "react-icons/fa";
import InputDetail from "components/InputDetail";
import ButtonFooter from "components/ButtonFooter";
import PageSpinner from "components/PageSpinner";
import ErrorComp from "components/ErrorComp";
import { useEntityParams } from "hooks/useEntityParams";
import { usePostEntity } from "hooks/usePostEntity";
import { Boy } from "models/Boy";
import { usePutEntity } from "hooks/usePutEntity";
import { partial, ifElse } from "ramda";
import { isTruthy } from "ramda-adjunct";

export default function BoyDetail() {
  const { id } = useParams();
  const { name, description } = useEntityParams();
  const [boy, setBoy] = useState({ id, name, description });
  const { mutate: createBoy, status, error: postError } = usePostEntity("boy");
  const {
    updateEntity: updateBoy,
    isUpdating,
    isUpdateError,
  } = usePutEntity("boy");

  const navigate = useNavigate();

  const handleCancel = partial(navigate, ["/boys"]);

  const handleSave = ifElse(
    () => isTruthy(name),
    () => updateBoy(boy as Boy),
    () => createBoy(boy as Boy)
  );

  const handleNameChange = (e: ChangeEvent<HTMLInputElement>) =>
    setBoy({ ...boy, name: e.target.value });

  const handleDescriptionChange = (e: ChangeEvent<HTMLInputElement>) =>
    setBoy({ ...boy, description: e.target.value });

  if (status === "loading" || isUpdating) {
    return <PageSpinner />;
  }

  if (postError || isUpdateError) {
    return <ErrorComp />;
  }

  return (
    <div data-cy="boy-detail" className="card edit-detail">
      <header className="card-header">
        <p className="card-header-title">{name}</p>
        &nbsp;
      </header>
      <div className="card-content">
        <div className="content">
          {id && (
            <InputDetail name={"id"} value={id} readOnly={true}></InputDetail>
          )}
          <InputDetail
            name={"name"}
            value={name ? name : ""}
            placeholder="e.g. Colleen"
            onChange={handleNameChange}
          ></InputDetail>
          <InputDetail
            name={"description"}
            value={description ? description : ""}
            placeholder="e.g. dance fight!"
            onChange={handleDescriptionChange}
          ></InputDetail>
        </div>
      </div>
      <footer className="card-footer">
        <ButtonFooter
          label="Cancel"
          IconClass={FaUndo}
          onClick={handleCancel}
        />
        <ButtonFooter label="Save" IconClass={FaRegSave} onClick={handleSave} />
      </footer>
    </div>
  );
}
```

## [`pipe`](https://ramdajs.com/docs/#pipe)

`pipe` is the bread and butter of Ramda; it is used to perform left-to-right function composition - the opposite of [`compose`](https://ramdajs.com/docs/#compose) which is right-to-left. Explained with a simple example:

```typescript
// standalone example, copy paste anywhere
import { compose, pipe } from "ramda";

// create a function that composes the three functions
const toUpperCase = (str: string) => str.toUpperCase();
const emphasizeFlavor = (flavor: string) => `${flavor} IS A GREAT FLAVOR`;
const appendIWantIt = (str: string) => `${str}. I want it`;

// note that with Ramda, we can pass the argument later
// and we shape the data through the composition / pipe
const classic = (flavor: string) =>
  appendIWantIt(emphasizeFlavor(toUpperCase(flavor)));
const composeRamda = compose(appendIWantIt, emphasizeFlavor, toUpperCase);
const pipeRamda = pipe(toUpperCase, emphasizeFlavor, appendIWantIt);

classic("chocolate");
composeRamda("chocolate");
pipeRamda("chocolate");
// CHOCOLATE IS A GREAT FLAVOR. I want it
```

Where can `pipe` apply in the React world? Any sequence of actions would be a fit.

The below 3 varieties of `handleCloseModal` function from `Boys.tsx` are equivalent.

```typescript
// src/boys/Boys.tsx
import { partial, pipe } from "ramda";

const handleCloseModal = () => {
  setBoyToDelete(null);
  setShowModal(false);
};
const handleCloseModal = pipe(
  () => setBoyToDelete(null),
  () => setShowModal(false)
);
const handleCloseModal = pipe(
  partial(setBoyToDelete, [null]),
  partial(setShowModal, [false])
);
```

The below 3 varieties of `handleDeleteBoy` function from `Boys.tsx` are also equivalent.

```typescript
// src/boys/Boys.tsx
import { partial, pipe } from "ramda";

const handleDeleteBoy = (boy: Boy) => () => {
  setBoyToDelete(boy);
  setShowModal(true);
};
const handleDeleteBoy = (boy: Boy) =>
  pipe(
    () => setBoyToDelete(boy),
    () => setShowModal(true)
  );
const handleDeleteBoy = (boy: Boy) =>
  pipe(partial(setBoyToDelete, [boy]), partial(setShowModal, [true]));
```

The below 3 varieties of `handleDeleteFromModal` function from `Boys.tsx` are also equivalent.

```typescript
// src/boys/Boys.tsx
const handleDeleteFromModal = () => {
  boyToDelete ? deleteBoy(boyToDelete) : null;
  setShowModal(false);
};
const handleDeleteFromModal2 = pipe(
  () => (boyToDelete ? deleteBoy(boyToDelete) : null),
  () => setShowModal(false)
);
const handleDeleteFromModal3 = pipe(
  () => (boyToDelete ? deleteBoy(boyToDelete) : null),
  partial(setShowModal, [false])
);
```

Here is the `Boys.tsx` component after the above changes. It can be compared to `Heroes.tsx` to see the distinction between using Ramda and classic array methods.

```tsx
// src/boys/Boys.tsx
import { useState } from "react";
import { useNavigate, Routes, Route } from "react-router-dom";
import ListHeader from "components/ListHeader";
import ModalYesNo from "components/ModalYesNo";
import ErrorComp from "components/ErrorComp";
import BoyList from "./BoyList";
import BoyDetail from "./BoyDetail";
import { useGetEntities } from "hooks/useGetEntities";
import { useDeleteEntity } from "hooks/useDeleteEntity";
import { Boy } from "models/Boy";
import { partial, pipe } from "ramda";

export default function Boys() {
  const [showModal, setShowModal] = useState<boolean>(false);
  const { entities: boys, getError } = useGetEntities("boys");
  const [boyToDelete, setBoyToDelete] = useState<Boy | null>(null);
  const { deleteEntity: deleteBoy, isDeleteError } = useDeleteEntity("boy");

  const navigate = useNavigate();
  const addNewBoy = partial(navigate, ["/boys/add-boy"]);
  const handleRefresh = partial(navigate, ["/boys"]);

  const handleCloseModal = pipe(
    partial(setBoyToDelete, [null]),
    partial(setShowModal, [false])
  );

  const handleDeleteBoy = (boy: Boy) =>
    pipe(partial(setBoyToDelete, [boy]), partial(setShowModal, [true]));

  const handleDeleteFromModal = pipe(
    () => (boyToDelete ? deleteBoy(boyToDelete) : null),
    partial(setShowModal, [false])
  );

  if (getError || isDeleteError) {
    return <ErrorComp />;
  }

  return (
    <div data-cy="boys">
      <ListHeader
        title="Boys"
        handleAdd={addNewBoy}
        handleRefresh={handleRefresh}
      />
      <div>
        <div>
          <Routes>
            <Route
              path=""
              element={
                <BoyList boys={boys} handleDeleteBoy={handleDeleteBoy} />
              }
            />
            <Route path="/add-boy" element={<BoyDetail />} />
            <Route path="/edit-boy/:id" element={<BoyDetail />} />
            <Route
              path="*"
              element={
                <BoyList boys={boys} handleDeleteBoy={handleDeleteBoy} />
              }
            />
          </Routes>
        </div>
      </div>

      {showModal && (
        <ModalYesNo
          message="Would you like to delete the boy?"
          onNo={handleCloseModal}
          onYes={handleDeleteFromModal}
        />
      )}
    </div>
  );
}
```

## Refactoring search-filter with array methods to Ramda

For this example, we will extract search-filter logic in `BoyList.tsx` into a standalone TS file, alongside types and data. Given the data `heroes` array, we want to filter by any property (`id`, `name`, `description`) and display 1 or more objects. To accomplish this, we use 2 lego-functions `searchExistsC` & `propertyExistsC` that builds up to `searchPropertiesC`, `C` for classic.

```typescript
// standalone example, copy paste anywhere

const heroes = [
  {
    id: "HeroAslaug",
    name: "Aslaug",
    description: "warrior queen",
  },
  {
    id: "HeroBjorn",
    name: "Bjorn Ironside",
    description: "king of 9th century Sweden",
  },
  {
    id: "HeroIvar",
    name: "Ivar the Boneless",
    description: "commander of the Great Heathen Army",
  },
  {
    id: "HeroLagertha",
    name: "Lagertha the Shieldmaiden",
    description: "aka Hlaðgerðr",
  },
  {
    id: "HeroRagnar",
    name: "Ragnar Lothbrok",
    description: "aka Ragnar Sigurdsson",
  },
  {
    id: "HeroThora",
    name: "Thora Town-hart",
    description: "daughter of Earl Herrauðr of Götaland",
  },
];
const textToSearch = "ragnar";
interface Hero {
  id: string;
  name: string;
  description: string;
}
type HeroProperty = Hero["name"] | Hero["description"] | Hero["id"];

// use the classic array methods to search for properties in a given array of entities

/** returns a boolean whether the entity property exists in the search field */
const searchExistsC = (searchField: string, searchProperty: HeroProperty) =>
  String(searchProperty).toLowerCase().indexOf(searchField.toLowerCase()) !==
  -1;

/** finds the given entity's property in the search field  */
const propertyExistsC = (searchField: string, item: Hero) =>
  Object.values(item).find((property: HeroProperty) =>
    searchExistsC(searchField, property)
  );

/** given the search field and the entity array, returns the entity 
in which the search field exists */
const searchPropertiesC = (data: Hero[], searchField: string) =>
  [...data].filter((item: Hero) => propertyExistsC(searchField, item));

searchPropertiesC(heroes, textToSearch);
/* 
[{ 
  id: 'HeroRagnar',
  name: 'Ragnar Lothbrok',
  description: 'aka Ragnar Sigurdsson' 
}]
*/
```

Although we have partitioned the functions like legos, the logic isn't very easy to follow while reading the code. The reason is having to use multiple arguments, and how the data changes from an array to an array item (an object), while also having to change the placement of the arguments. Using Ramda, we can make the code easier to read and use.

Here is a quick comparison of `.toLowerCase()` vs `toLower()` and `.indexOf()` vs `indexOf`. Instead of chaining on the data, Ramda helpers `indexOf` and `toLower` are functions that take the data as an argument:

- `someData.toLowerCase()` vs `toLower(someData)`
- `array.indexOf(arrayItem)` vs `indexOf(arrayItem, array)`

```typescript
"HELLO".toLowerCase(); // hello
toLower("HELLO"); // hello
[1, 2, 3, 4].indexOf(3); // 2
indexOf(3, [1, 2, 3, 4]); // 2
```

We can refactor `searchExistsC` to use Ramda. Here is the before and after side by side:

```typescript
import { indexOf, toLower } from "ramda";

const searchExistsC = (searchField: string, searchProperty: HeroProperty) =>
  String(searchProperty).toLowerCase().indexOf(searchField.toLowerCase()) !==
  -1;

const searchExists = (searchField: string, searchProperty: HeroProperty) =>
  indexOf(toLower(searchField), toLower(searchProperty)) !== -1;
```

Below is a quick comparison of `array.find` vs Ramda `find`, `Object.values` vs Ramda `values`. Notice with Ramda version how the data comes at the end, with this style we can save an arg for later.

```typescript
// data.find(callback)
Object.values(heroes[4]).find(
  (property: HeroProperty) => property === "Ragnar Lothbrok"
);
// find(callback)(data)
find((property: HeroProperty) => property === "Ragnar Lothbrok")(
  values(heroes[4])
);
```

Here is the refactor of `propertyExistsC` to use Ramda, before and after side by side. We can amplify the Ramda version with `pipe` to make the flow more similar to the classic version.

```typescript
const propertyExistsC = (searchField: string, item: Hero) =>
  Object.values(item).find((property: HeroProperty) =>
    searchExistsC(searchField, property)
  );

// we wrap the function in curry to allow passing the item argument independently, later
// f(a, b) to curry(f(a, b)) allows us to f(a)(b)
const propertyExists = curry((searchField: string, item: Hero) =>
  find((property: HeroProperty) => searchExists(searchField, property))(
    values(item)
  )
);

// we can take it a step further with pipe
// the data item comes at the end, we take it and pipe through values & find
// this way the flow is more similar to the original function
const propertyExistsNew = curry((searchField: string, item: Hero) =>
  pipe(
    values,
    find((property: HeroProperty) => searchExists(searchField, property))
  )(item)
);
```

Our final function `searchExistsC` uses the previous two. Here are the classic and Ramda versions side by side. We can evolve the Ramda version to take 1 argument, and get the data later whenever it is available.

```typescript
// (2 args) => data.filter(callback)
const searchPropertiesC = (data: Hero[], searchField: string) =>
  [...data].filter((item: Hero) => propertyExistsC(searchField, item));

// (2 args) => filter(callback, data)
const searchProperties = (searchField: string, data: Hero[]) =>
  filter((item: Hero) => propertyExists(searchField, item), [...data]);

// (2 args) => filter(callback)(data)
const searchPropertiesBetter = (searchField: string, data: Hero[]) =>
  filter((item: Hero) => propertyExistsNew(searchField, item))(data);

// (1 arg) => filter(fn), the data can come later and it will flow through
// notice how we eliminated another piece of data; item
const searchPropertiesBest = (searchField: string) =>
  filter(propertyExistsNew(searchField));
```

Here is the full example file that can be copy pasted anywhere

```typescript
// standalone example, copy paste anywhere

import { indexOf, filter, find, curry, toLower, pipe, values } from "ramda";

const heroes = [
  {
    id: "HeroAslaug",
    name: "Aslaug",
    description: "warrior queen",
  },
  {
    id: "HeroBjorn",
    name: "Bjorn Ironside",
    description: "king of 9th century Sweden",
  },
  {
    id: "HeroIvar",
    name: "Ivar the Boneless",
    description: "commander of the Great Heathen Army",
  },
  {
    id: "HeroLagertha",
    name: "Lagertha the Shieldmaiden",
    description: "aka Hlaðgerðr",
  },
  {
    id: "HeroRagnar",
    name: "Ragnar Lothbrok",
    description: "aka Ragnar Sigurdsson",
  },
  {
    id: "HeroThora",
    name: "Thora Town-hart",
    description: "daughter of Earl Herrauðr of Götaland",
  },
];
const textToSearch = "ragnar";
interface Hero {
  id: string;
  name: string;
  description: string;
}

Object.values(heroes[4]).find(
  (property: HeroProperty) => property === "Ragnar Lothbrok"
);

find((property: HeroProperty) => property === "Ragnar Lothbrok")(
  values(heroes[4])
); //?

type HeroProperty = Hero["name"] | Hero["description"] | Hero["id"];

/** returns a boolean whether the hero properties exist in the search field */
const searchExistsC = (searchField: string, searchProperty: HeroProperty) =>
  String(searchProperty).toLowerCase().indexOf(searchField.toLowerCase()) !==
  -1;

const propertyExistsC = (searchField: string, item: Hero) =>
  Object.values(item).find((property: HeroProperty) =>
    searchExistsC(searchField, property)
  );

const searchPropertiesC = (data: Hero[], searchField: string) =>
  [...data].filter((item: Hero) => propertyExistsC(searchField, item));

searchPropertiesC(heroes, textToSearch); //?

// rewrite in ramda

const searchExists = (searchField: string, searchProperty: HeroProperty) =>
  indexOf(toLower(searchField), toLower(searchProperty)) !== -1;

const propertyExists = curry((searchField: string, item: Hero) =>
  find((property: HeroProperty) => searchExists(searchField, property))(
    values(item)
  )
);
// refactor propertyExists to use pipe
const propertyExistsNew = curry((searchField: string, item: Hero) =>
  pipe(
    values,
    find((property: HeroProperty) => searchExists(searchField, property))
  )(item)
);

// refactor in better ramda
const searchProperties = (searchField: string, data: Hero[]) =>
  filter((item: Hero) => propertyExists(searchField, item), [...data]);

const searchPropertiesBetter = (searchField: string, data: Hero[]) =>
  filter((item: Hero) => propertyExistsNew(searchField, item))(data);

const searchPropertiesBest = (searchField: string) =>
  filter(propertyExistsNew(searchField));

searchProperties(textToSearch, heroes); //?
searchPropertiesBetter(textToSearch, heroes); //?
searchPropertiesBest(textToSearch)(heroes); //?
```

With that, we can replace `searchProperties` and the lego-functions that lead up to it with their Ramda versions.

Here is the comparison of the key changes:

```tsx
// src/boys/BoyList.tsx (before)

/** returns a boolean whether the boy properties exist in the search field */
const searchExists = (searchProperty: BoyProperty, searchField: string) =>
  String(searchProperty).toLowerCase().indexOf(searchField.toLowerCase()) !==
  -1;

/** given the data and the search field, returns the data in which the search field exists */
const searchProperties = (searchField: string, data: Boy[]) =>
  [...data].filter((item: Boy) =>
    Object.values(item).find((property: BoyProperty) =>
      searchExists(property, searchField)
    )
  );

const handleSearch =
  (data: Boy[]) => (event: ChangeEvent<HTMLInputElement>) => {
    const searchField = event.target.value;

    return startTransition(() =>
      setFilteredBoys(searchProperties(searchField, data))
    );
  };
```

```tsx
// src/boys/BoyList.tsx (after)

/** returns a boolean whether the boy properties exist in the search field */
const searchExists = (searchField: string, searchProperty: BoyProperty) =>
  indexOf(toLower(searchField), toLower(searchProperty)) !== -1;

/** finds the given boy's property in the search field  */
const propertyExists = curry((searchField: string, item: Boy) =>
  pipe(
    values,
    find((property: BoyProperty) => searchExists(searchField, property))
  )(item)
);
/** given the search field and the boy array, returns the boy in which the search field exists */
const searchProperties = (
  searchField: string
): (<P extends Boy, C extends readonly P[] | Dictionary<P>>(
  collection: C
) => C) => filter(propertyExists(searchField));

/** filters the boys data to see if the any of the properties exist in the list */
const handleSearch =
  (data: Boy[]) => (event: ChangeEvent<HTMLInputElement>) => {
    const searchField = event.target.value;
    const searchedBoy = searchProperties(searchField)(data);

    return startTransition(() =>
      setFilteredBoys(searchedBoy as React.SetStateAction<Boy[]>)
    );
  };
```

Here is the final form of the component:

```tsx
// src/boys/BoyList.tsx
import { useNavigate } from "react-router-dom";
import CardContent from "components/CardContent";
import ButtonFooter from "components/ButtonFooter";
import { FaEdit, FaRegSave } from "react-icons/fa";
import {
  ChangeEvent,
  MouseEvent,
  useTransition,
  useEffect,
  useState,
  useDeferredValue,
} from "react";
import { Boy } from "models/Boy";
import { BoyProperty } from "models/types";
import {
  indexOf,
  find,
  curry,
  toLower,
  pipe,
  values,
  filter,
  Dictionary,
} from "ramda";

type BoyListProps = {
  boys: Boy[];
  handleDeleteBoy: (boy: Boy) => (e: MouseEvent<HTMLButtonElement>) => void;
};

export default function BoyList({ boys, handleDeleteBoy }: BoyListProps) {
  const deferredBoys = useDeferredValue(boys);
  const isStale = deferredBoys !== boys;
  const [filteredBoys, setFilteredBoys] = useState(deferredBoys);
  const navigate = useNavigate();
  const [isPending, startTransition] = useTransition();

  // needed to refresh the list after deleting a boy
  useEffect(() => setFilteredBoys(deferredBoys), [deferredBoys]);

  const handleSelectBoy = curry(
    (boyId: string, e: MouseEvent<HTMLButtonElement>) => {
      const boy = deferredBoys.find((b: Boy) => b.id === boyId);
      navigate(
        `/boys/edit-boy/${boy?.id}?name=${boy?.name}&description=${boy?.description}`
      );
    }
  );

  /** returns a boolean whether the boy properties exist in the search field */
  const searchExists = (searchField: string, searchProperty: BoyProperty) =>
    indexOf(toLower(searchField), toLower(searchProperty)) !== -1;

  /** finds the given boy's property in the search field  */
  const propertyExists = curry((searchField: string, item: Boy) =>
    pipe(
      values,
      find((property: BoyProperty) => searchExists(searchField, property))
    )(item)
  );
  /** given the search field and the boy array, returns the boy in which the search field exists */
  const searchProperties = (
    searchField: string
  ): (<P extends Boy, C extends readonly P[] | Dictionary<P>>(
    collection: C
  ) => C) => filter(propertyExists(searchField));

  /** filters the boys data to see if the any of the properties exist in the list */
  const handleSearch =
    (data: Boy[]) => (event: ChangeEvent<HTMLInputElement>) => {
      const searchField = event.target.value;
      const searchedBoy = searchProperties(searchField)(data);

      return startTransition(() =>
        setFilteredBoys(searchedBoy as React.SetStateAction<Boy[]>)
      );
    };

  return (
    <div
      style={{
        opacity: isPending ? 0.5 : 1,
        color: isStale ? "dimgray" : "black",
      }}
    >
      {deferredBoys.length > 0 && (
        <div className="card-content">
          <span>Search </span>
          <input data-cy="search" onChange={handleSearch(deferredBoys)} />
        </div>
      )}
      &nbsp;
      <ul data-cy="boy-list" className="list">
        {filteredBoys.map((boy, index) => (
          <li data-cy={`boy-list-item-${index}`} key={boy.id}>
            <div className="card">
              <CardContent name={boy.name} description={boy.description} />
              <footer className="card-footer">
                <ButtonFooter
                  label="Delete"
                  IconClass={FaRegSave}
                  onClick={handleDeleteBoy(boy)}
                />
                <ButtonFooter
                  label="Edit"
                  IconClass={FaEdit}
                  onClick={handleSelectBoy(boy.id)}
                />
              </footer>
            </div>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

Our tooling and tests are still intact after the changes, and that is the luxury of having very high coverage. We were able to apply daring, experimental refactors to our code, and are still confident that nothing regressed while the code became easier to read and work with using FP and Ramda. 

To take a look at the before and after, you can find the PR for this section at https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/pull/110. You can also compare the Boys group of files to Heroes or Villains.
