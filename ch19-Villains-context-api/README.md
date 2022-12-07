# Context API

Before starting this section,
make sure to go through the [prerequisite](./prerequisite.md) where we update the application to be more generic between Heroes and Villains.

In this chapter we will mirror heroes into villains, and apply the Context api to villains. Context api lets us pass a value deep into the component tree, without explicitly threading it through every component. This will give us a nice contrast between passing the state as prop to child components, versus using the context to share the state down the component tree.

`Heroes.tsx` passes `heroes` as a prop to `HeroList.tsx`.

`HeroDetail` gets `hero` `{id, name, description}` state from the url with `useParams` & `useSearchParams` .

This is the extent of state management in our app. We do not really need context, but we will explore how things can be different with the context api.

## Context API for villains

At the moment we have a full mirror of heroes to villains, functioning and being tested exactly the same way. We will however modify the villains group and take advantage of Context api while doing so.

`Villains.tsx` passes `villains` as a prop to `VillainList.tsx`. We will instead use the Context api so that `villains` is available in all components under `Villains.txt`.

Here are the general steps with Context api:

1.  Create the context and export it. Usually this is in a separate file, acting as an arbiter.

    ```tsx
    // src/villains/VillainsContext.tsx (the common node)
    import { Villain } from "models/Villain";
    import { createContext } from "react";

    const VillainsContext = createContext<Villain[]>([]);
    export default VillainsContext;
    ```

2.  Identify the state to be passed down to child components. Import the context there.

    ```tsx
    // src/villains/Villains.tsx
    import { VillainsContext } from "./VillainsContext";

    // ...

    const { villains, status, getError } = useGetEntity();
    ```

3.  Wrap the UI with the context’s Provider component, assign the state to be passed down to the `value` prop:

    ```tsx
    // src/villains/Villains.tsx (the sharer)

    <VillainsContext.Provider value={villains}>
      <Routes>...</Routes>
    </VillainsContext.Provider>
    ```

4.  In any component that is needing the state, consume Context API; import the `useContext` hook, and the context object:

    ```tsx
    // src/villains/VillainList.tsx
    import { useContext } from "react";
    import { VillainsContext } from "./VillainsContext";
    ```

5.  Call useContext with the shared context, assign to a var:

    ```tsx
    // src/villains/VillainList.tsx (the sharee)
    import { useContext } from "react";
    import { VillainsContext } from "./VillainsContext";

    // ..

    const villains = useContext(VillainsContext);
    ```

Following step 1, we create the context at a new file, and export it:

```tsx
// src/villains/VillainsContext.ts
import { Villain } from "models/Villain";
import { createContext } from "react";

const VillainsContext = createContext<Villain[]>([]);

export default VillainsContext;
```

In our example, from `Villains.tsx`, we are passing `villains` to `VillainDetail.tsx`. We get `villains` from the hook `useGetEntity` . We are currently using a prop to pass `villains` to `VillainDetail.tsx`, and we want to instead use the context api. So we import the context, and wrap the routes with the context provider, which has a `value` prop with `villains` assigned to it (Steps 2, 3).

```tsx
// src/villains/Villains.tsx
import { useState } from "react";
import { useNavigate, Routes, Route } from "react-router-dom";
import ListHeader from "components/ListHeader";
import ModalYesNo from "components/ModalYesNo";
import PageSpinner from "components/PageSpinner";
import ErrorComp from "components/ErrorComp";
import VillainList from "./VillainList";
import VillainDetail from "./VillainDetail";
import { useGetEntities } from "hooks/useGetEntities";
import { useDeleteEntity } from "hooks/useDeleteEntity";
import { Villain } from "models/Villain";
import VillainsContext from "./VillainsContext";

export default function Villains() {
  const [showModal, setShowModal] = useState<boolean>(false);
  const { entities: villains, status, getError } = useGetEntities("villains");
  const [villainToDelete, setVillainToDelete] = useState<Villain | null>(null);
  const { deleteEntity: deleteVillain, isDeleteError } =
    useDeleteEntity("villain");

  const navigate = useNavigate();
  const addNewVillain = () => navigate("/villains/add-villain");
  const handleRefresh = () => navigate("/villains");

  const handleCloseModal = () => {
    setVillainToDelete(null);
    setShowModal(false);
  };

  const handleDeleteVillain = (villain: Villain) => () => {
    setVillainToDelete(villain);
    setShowModal(true);
  };
  const handleDeleteFromModal = () => {
    villainToDelete ? deleteVillain(villainToDelete) : null;
    setShowModal(false);
  };

  if (status === "loading") {
    return <PageSpinner />;
  }

  if (getError || isDeleteError) {
    return <ErrorComp />;
  }

  return (
    <div data-cy="villains">
      <ListHeader
        title="Villains"
        handleAdd={addNewVillain}
        handleRefresh={handleRefresh}
      />
      <div>
        <div>
          <VillainsContext.Provider value={villains}>
            <Routes>
              <Route
                path=""
                element={
                  <VillainList handleDeleteVillain={handleDeleteVillain} />
                }
              />
              <Route path="/add-villain" element={<VillainDetail />} />
              <Route path="/edit-villain/:id" element={<VillainDetail />} />
              <Route
                path="*"
                element={
                  <VillainList handleDeleteVillain={handleDeleteVillain} />
                }
              />
            </Routes>
          </VillainsContext.Provider>
        </div>
      </div>

      {showModal && (
        <ModalYesNo
          message="Would you like to delete the villain?"
          onNo={handleCloseModal}
          onYes={handleDeleteFromModal}
        />
      )}
    </div>
  );
}
```

`VillainsList.tsx` needs to be passed down the state via context versus a prop. So we import the context, and import`useContext` from React. We invoke `useContext` with the shared context as its argument, and assign to a variable `villains` (Steps 4, 5). Now, instead of the prop, we are getting the state from the `VillainsContext`.

```tsx
// src/villains/VillainList.tsx
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
import { useContext } from "react";
import { Villain } from "models/Villain";
import VillainsContext from "./VillainsContext";

type VillainListProps = {
  handleDeleteVillain: (
    villain: Villain
  ) => (e: MouseEvent<HTMLButtonElement>) => void;
};

export default function VillainList({ handleDeleteVillain }: VillainListProps) {
  const villains = useContext(VillainsContext);

  const deferredVillains = useDeferredValue(villains);
  const isStale = deferredVillains !== villains;
  const [filteredVillains, setFilteredVillains] = useState(deferredVillains);
  const navigate = useNavigate();
  const [isPending, startTransition] = useTransition();

  // needed to refresh the list after deleting a villain
  useEffect(() => setFilteredVillains(deferredVillains), [deferredVillains]);

  const handleSelectVillain = (villainId: string) => () => {
    const villain = deferredVillains.find((h: Villain) => h.id === villainId);
    navigate(
      `/villains/edit-villain/${villain?.id}?name=${villain?.name}&description=${villain?.description}`
    );
  };

  type VillainProperty =
    | Villain["name"]
    | Villain["description"]
    | Villain["id"];

  /** returns a boolean whether the villain properties exist in the search field */
  const searchExists = (searchProperty: VillainProperty, searchField: string) =>
    String(searchProperty).toLowerCase().indexOf(searchField.toLowerCase()) !==
    -1;

  /** given the data and the search field, returns the data in which the search field exists */
  const searchProperties = (searchField: string, data: Villain[]) =>
    [...data].filter((item: Villain) =>
      Object.values(item).find((property: VillainProperty) =>
        searchExists(property, searchField)
      )
    );

  /** filters the villains data to see if the any of the properties exist in the list */
  const handleSearch =
    (data: Villain[]) => (event: ChangeEvent<HTMLInputElement>) => {
      const searchField = event.target.value;

      return startTransition(() =>
        setFilteredVillains(searchProperties(searchField, data))
      );
    };

  return (
    <div
      style={{
        opacity: isPending ? 0.5 : 1,
        color: isStale ? "dimgray" : "black",
      }}
    >
      {deferredVillains.length > 0 && (
        <div className="card-content">
          <span>Search </span>
          <input data-cy="search" onChange={handleSearch(deferredVillains)} />
        </div>
      )}
      &nbsp;
      <ul data-cy="villain-list" className="list">
        {filteredVillains.map((villain, index) => (
          <li data-cy={`villain-list-item-${index}`} key={villain.id}>
            <div className="card">
              <CardContent
                name={villain.name}
                description={villain.description}
              />
              <footer className="card-footer">
                <ButtonFooter
                  label="Delete"
                  IconClass={FaRegSave}
                  onClick={handleDeleteVillain(villain)}
                />
                <ButtonFooter
                  label="Edit"
                  IconClass={FaEdit}
                  onClick={handleSelectVillain(villain.id)}
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

Update the component test to also use the context provider when mounting.

```tsx
// src/villains/VillainList.cy.tsx
import VillainList from "./VillainList";
import "../styles.scss";
import villains from "../../cypress/fixtures/villains.json";
import VillainsContext from "./VillainsContext";

describe("VillainList", () => {
  it("no villains should not display a list nor search bar", () => {
    cy.wrappedMount(
      <VillainList handleDeleteVillain={cy.stub().as("handleDeleteVillain")} />
    );

    cy.getByCy("villain-list").should("exist");
    cy.getByCyLike("villain-list-item").should("not.exist");
    cy.getByCy("search").should("not.exist");
  });

  context("with villains in the list", () => {
    beforeEach(() => {
      cy.wrappedMount(
        <VillainsContext.Provider value={villains}>
          <VillainList
            handleDeleteVillain={cy.stub().as("handleDeleteVillain")}
          />
        </VillainsContext.Provider>
      );
    });

    it("should render the villain layout", () => {
      cy.getByCyLike("villain-list-item").should(
        "have.length",
        villains.length
      );

      cy.getByCy("card-content");
      cy.contains(villains[0].name);
      cy.contains(villains[0].description);

      cy.get("footer")
        .first()
        .within(() => {
          cy.getByCy("delete-button");
          cy.getByCy("edit-button");
        });
    });

    it("should search and filter villain by name and description", () => {
      cy.getByCy("search").type(villains[0].name);
      cy.getByCyLike("villain-list-item")
        .should("have.length", 1)
        .contains(villains[0].name);

      cy.getByCy("search").clear().type(villains[2].description);
      cy.getByCyLike("villain-list-item")
        .should("have.length", 1)
        .contains(villains[2].description);
    });

    it("should handle delete", () => {
      cy.getByCy("delete-button").first().click();
      cy.get("@handleDeleteVillain").should("have.been.called");
    });

    it("should handle edit", () => {
      cy.getByCy("edit-button").first().click();
      cy.location("pathname").should(
        "eq",
        "/villains/edit-villain/" + villains[0].id
      );
    });
  });
});
```

Update the RTL test to also use context provider when rendering.

```tsx
// src/villains/VillainList.test.tsx
import VillainList from "./VillainList";
import { wrappedRender, screen, waitFor } from "test-utils";
import userEvent from "@testing-library/user-event";
import { villains } from "../../db.json";
import VillainsContext from "./VillainsContext";

describe("VillainList", () => {
  const handleDeleteVillain = jest.fn();

  it("no villains should not display a list nor search bar", async () => {
    wrappedRender(<VillainList handleDeleteVillain={handleDeleteVillain} />);

    expect(await screen.findByTestId("villain-list")).toBeInTheDocument();
    expect(screen.queryByTestId("villain-list-item-1")).not.toBeInTheDocument();
    expect(screen.queryByTestId("search-bar")).not.toBeInTheDocument();
  });

  describe("with villains in the list", () => {
    beforeEach(() => {
      wrappedRender(
        <VillainsContext.Provider value={villains}>
          <VillainList handleDeleteVillain={handleDeleteVillain} />
        </VillainsContext.Provider>
      );
    });

    const cardContents = async () => screen.findAllByTestId("card-content");
    const deleteButtons = async () => screen.findAllByTestId("delete-button");
    const editButtons = async () => screen.findAllByTestId("edit-button");

    it("should render the villain layout", async () => {
      expect(
        await screen.findByTestId(`villain-list-item-${villains.length - 1}`)
      ).toBeInTheDocument();

      expect(await screen.findByText(villains[0].name)).toBeInTheDocument();
      expect(
        await screen.findByText(villains[0].description)
      ).toBeInTheDocument();
      expect(await cardContents()).toHaveLength(villains.length);
      expect(await deleteButtons()).toHaveLength(villains.length);
      expect(await editButtons()).toHaveLength(villains.length);
    });

    it("should search and filter villain by name and description", async () => {
      const search = await screen.findByTestId("search");

      userEvent.type(search, villains[0].name);
      await waitFor(async () => expect(await cardContents()).toHaveLength(1));
      await screen.findByText(villains[0].name);

      userEvent.clear(search);
      await waitFor(async () =>
        expect(await cardContents()).toHaveLength(villains.length)
      );

      userEvent.type(search, villains[2].description);
      await waitFor(async () => expect(await cardContents()).toHaveLength(1));
    });

    it("should handle delete", async () => {
      userEvent.click((await deleteButtons())[0]);
      expect(handleDeleteVillain).toHaveBeenCalled();
    });

    it("should handle edit", async () => {
      userEvent.click((await editButtons())[0]);
      await waitFor(() =>
        expect(window.location.pathname).toEqual(
          "/villains/edit-villain/" + villains[0].id
        )
      );
    });
  });
});
```

### Using a custom hook for sharing context

The context, created at `VillainsContext` is acting as the arbiter. `Villains.tsx` uses the context to share `villains` state to its child `VillainList`. `VillainList` acquires the state from `VillainsContext`. Instead of `VillainsContext` we can use a hook that will reduce the imports.

Remove `src/villains/VillainsContext.ts` and instead create a hook `src/hooks/useVillainsContext.ts`. The first half of the hook is the same as `VillainsContext`. We add a setter, so that the components that get passed down the state also gain the ability to set it. Additionally we manage the state and effects related to the hook’s functionality within the hook and return only the value(s) that components need.

```typescript
// src/hooks/useVillainsContext.ts
import { createContext, useContext, SetStateAction, Dispatch } from "react";
import { Villain } from "models/Villain";

// Context api lets us pass a value deep into the component tree
// without explicitly threading it through every component (2nd tier state management)

const VillainsContext = createContext<Villain[]>([]);
// to be used as VillainsContext.Provider,
// takes a prop as `value`, which is the context/data/state to share
export default VillainsContext;

const VillainsSetContext = createContext<Dispatch<
  SetStateAction<Villain[] | null>
> | null>(null);

// Manage state and effects related to a hook’s functionality
// within the hook and return only the value(s) that components need

export function useVillainsContext() {
  const villains = useContext(VillainsContext);
  const setVillains = useContext(VillainsSetContext);

  return [villains, setVillains] as const;
}
```

At `Villains.tsx` the only change is the import location of `VillainsContext`.

```tsx
// src/villains/Villains.tsx
import { useState } from "react";
import { useNavigate, Routes, Route } from "react-router-dom";
import ListHeader from "components/ListHeader";
import ModalYesNo from "components/ModalYesNo";
import PageSpinner from "components/PageSpinner";
import ErrorComp from "components/ErrorComp";
import VillainList from "./VillainList";
import VillainDetail from "./VillainDetail";
import { useGetEntities } from "hooks/useGetEntities";
import { useDeleteEntity } from "hooks/useDeleteEntity";
import { Villain } from "models/Villain";
import VillainsContext from "hooks/useVillainsContext";

export default function Villains() {
  const [showModal, setShowModal] = useState<boolean>(false);
  const { entities: villains, status, getError } = useGetEntities("villains");
  const [villainToDelete, setVillainToDelete] = useState<Villain | null>(null);
  const { deleteEntity: deleteVillain, isDeleteError } =
    useDeleteEntity("villain");

  const navigate = useNavigate();
  const addNewVillain = () => navigate("/villains/add-villain");
  const handleRefresh = () => navigate("/villains");

  const handleCloseModal = () => {
    setVillainToDelete(null);
    setShowModal(false);
  };

  const handleDeleteVillain = (villain: Villain) => () => {
    setVillainToDelete(villain);
    setShowModal(true);
  };
  const handleDeleteFromModal = () => {
    villainToDelete ? deleteVillain(villainToDelete) : null;
    setShowModal(false);
  };

  if (status === "loading") {
    return <PageSpinner />;
  }

  if (getError || isDeleteError) {
    return <ErrorComp />;
  }

  return (
    <div data-cy="villains">
      <ListHeader
        title="Villains"
        handleAdd={addNewVillain}
        handleRefresh={handleRefresh}
      />
      <div>
        <div>
          <VillainsContext.Provider value={villains}>
            <Routes>
              <Route
                path=""
                element={
                  <VillainList handleDeleteVillain={handleDeleteVillain} />
                }
              />
              <Route path="/add-villain" element={<VillainDetail />} />
              <Route path="/edit-villain/:id" element={<VillainDetail />} />
              <Route
                path="*"
                element={
                  <VillainList handleDeleteVillain={handleDeleteVillain} />
                }
              />
            </Routes>
          </VillainsContext.Provider>
        </div>
      </div>

      {showModal && (
        <ModalYesNo
          message="Would you like to delete the villain?"
          onNo={handleCloseModal}
          onYes={handleDeleteFromModal}
        />
      )}
    </div>
  );
}
```

Similarly, at `VillainsList` component test, only the import changes. We are using the hook `useVillainsContext`. The same applies to `VillainList.test.tsx`.

```tsx
// src/villains/VillainList.cy.tsx
import VillainList from "./VillainList";
import "../styles.scss";
import villains from "../../cypress/fixtures/villains.json";
import VillainsContext from "hooks/useVillainsContext";

describe("VillainList", () => {
  it("no villains should not display a list nor search bar", () => {
    cy.wrappedMount(
      <VillainList handleDeleteVillain={cy.stub().as("handleDeleteVillain")} />
    );

    cy.getByCy("villain-list").should("exist");
    cy.getByCyLike("villain-list-item").should("not.exist");
    cy.getByCy("search").should("not.exist");
  });

  context("with villains in the list", () => {
    beforeEach(() => {
      cy.wrappedMount(
        <VillainsContext.Provider value={villains}>
          <VillainList
            handleDeleteVillain={cy.stub().as("handleDeleteVillain")}
          />
        </VillainsContext.Provider>
      );
    });

    it("should render the villain layout", () => {
      cy.getByCyLike("villain-list-item").should(
        "have.length",
        villains.length
      );

      cy.getByCy("card-content");
      cy.contains(villains[0].name);
      cy.contains(villains[0].description);

      cy.get("footer")
        .first()
        .within(() => {
          cy.getByCy("delete-button");
          cy.getByCy("edit-button");
        });
    });

    it("should search and filter villain by name and description", () => {
      cy.getByCy("search").type(villains[0].name);
      cy.getByCyLike("villain-list-item")
        .should("have.length", 1)
        .contains(villains[0].name);

      cy.getByCy("search").clear().type(villains[2].description);
      cy.getByCyLike("villain-list-item")
        .should("have.length", 1)
        .contains(villains[2].description);
    });

    it("should handle delete", () => {
      cy.getByCy("delete-button").first().click();
      cy.get("@handleDeleteVillain").should("have.been.called");
    });

    it("should handle edit", () => {
      cy.getByCy("edit-button").first().click();
      cy.location("pathname").should(
        "eq",
        "/villains/edit-villain/" + villains[0].id
      );
    });
  });
});
```

```tsx
// src/villains/VillainList.test.tsx
import VillainList from "./VillainList";
import { wrappedRender, screen, waitFor } from "test-utils";
import userEvent from "@testing-library/user-event";
import { villains } from "../../db.json";
import VillainsContext from "hooks/useVillainsContext";

describe("VillainList", () => {
  const handleDeleteVillain = jest.fn();

  it("no villains should not display a list nor search bar", async () => {
    wrappedRender(<VillainList handleDeleteVillain={handleDeleteVillain} />);

    expect(await screen.findByTestId("villain-list")).toBeInTheDocument();
    expect(screen.queryByTestId("villain-list-item-1")).not.toBeInTheDocument();
    expect(screen.queryByTestId("search-bar")).not.toBeInTheDocument();
  });

  describe("with villains in the list", () => {
    beforeEach(() => {
      wrappedRender(
        <VillainsContext.Provider value={villains}>
          <VillainList handleDeleteVillain={handleDeleteVillain} />
        </VillainsContext.Provider>
      );
    });

    const cardContents = async () => screen.findAllByTestId("card-content");
    const deleteButtons = async () => screen.findAllByTestId("delete-button");
    const editButtons = async () => screen.findAllByTestId("edit-button");

    it("should render the villain layout", async () => {
      expect(
        await screen.findByTestId(`villain-list-item-${villains.length - 1}`)
      ).toBeInTheDocument();

      expect(await screen.findByText(villains[0].name)).toBeInTheDocument();
      expect(
        await screen.findByText(villains[0].description)
      ).toBeInTheDocument();
      expect(await cardContents()).toHaveLength(villains.length);
      expect(await deleteButtons()).toHaveLength(villains.length);
      expect(await editButtons()).toHaveLength(villains.length);
    });

    it("should search and filter villain by name and description", async () => {
      const search = await screen.findByTestId("search");

      userEvent.type(search, villains[0].name);
      await waitFor(async () => expect(await cardContents()).toHaveLength(1));
      await screen.findByText(villains[0].name);

      userEvent.clear(search);
      await waitFor(async () =>
        expect(await cardContents()).toHaveLength(villains.length)
      );

      userEvent.type(search, villains[2].description);
      await waitFor(async () => expect(await cardContents()).toHaveLength(1));
    });

    it("should handle delete", async () => {
      userEvent.click((await deleteButtons())[0]);
      expect(handleDeleteVillain).toHaveBeenCalled();
    });

    it("should handle edit", async () => {
      userEvent.click((await editButtons())[0]);
      await waitFor(() =>
        expect(window.location.pathname).toEqual(
          "/villains/edit-villain/" + villains[0].id
        )
      );
    });
  });
});
```

At `VillainList.tsx`, as in `Villains.tsx` the import changes. Additionally, we do not need to import `useContext`. We now de-structure villains; `const [villains] = useVillainsContext()`. If we needed to we could also get the setter out of the hook to set the context.

```tsx
// src/villains/VillainList.tsx
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
import { useVillainsContext } from "hooks/useVillainsContext";
import { Villain } from "models/Villain";

type VillainListProps = {
  handleDeleteVillain: (
    villain: Villain
  ) => (e: MouseEvent<HTMLButtonElement>) => void;
};

export default function VillainList({ handleDeleteVillain }: VillainListProps) {
  const [villains] = useVillainsContext();

  const deferredVillains = useDeferredValue(villains);
  const isStale = deferredVillains !== villains;
  const [filteredVillains, setFilteredVillains] = useState(deferredVillains);
  const navigate = useNavigate();
  const [isPending, startTransition] = useTransition();

  // needed to refresh the list after deleting a villain
  useEffect(() => setFilteredVillains(deferredVillains), [deferredVillains]);

  // currying: the outer fn takes our custom arg and returns a fn that takes the event
  const handleSelectVillain = (villainId: string) => () => {
    const villain = deferredVillains.find((h: Villain) => h.id === villainId);
    navigate(
      `/villains/edit-villain/${villain?.id}?name=${villain?.name}&description=${villain?.description}`
    );
  };

  type VillainProperty =
    | Villain["name"]
    | Villain["description"]
    | Villain["id"];

  /** returns a boolean whether the villain properties exist in the search field */
  const searchExists = (searchProperty: VillainProperty, searchField: string) =>
    String(searchProperty).toLowerCase().indexOf(searchField.toLowerCase()) !==
    -1;

  /** given the data and the search field, returns the data in which the search field exists */
  const searchProperties = (searchField: string, data: Villain[]) =>
    [...data].filter((item: Villain) =>
      Object.values(item).find((property: VillainProperty) =>
        searchExists(property, searchField)
      )
    );

  /** filters the villains data to see if the any of the properties exist in the list */
  const handleSearch =
    (data: Villain[]) => (event: ChangeEvent<HTMLInputElement>) => {
      const searchField = event.target.value;

      return startTransition(() =>
        setFilteredVillains(searchProperties(searchField, data))
      );
    };

  return (
    <div
      style={{
        opacity: isPending ? 0.5 : 1,
        color: isStale ? "dimgray" : "black",
      }}
    >
      {deferredVillains.length > 0 && (
        <div className="card-content">
          <span>Search </span>
          <input data-cy="search" onChange={handleSearch(deferredVillains)} />
        </div>
      )}
      &nbsp;
      <ul data-cy="villain-list" className="list">
        {filteredVillains.map((villain, index) => (
          <li data-cy={`villain-list-item-${index}`} key={villain.id}>
            <div className="card">
              <CardContent
                name={villain.name}
                description={villain.description}
              />
              <footer className="card-footer">
                <ButtonFooter
                  label="Delete"
                  IconClass={FaRegSave}
                  onClick={handleDeleteVillain(villain)}
                />
                <ButtonFooter
                  label="Edit"
                  IconClass={FaEdit}
                  onClick={handleSelectVillain(villain.id)}
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

## Summary & Takeaways

- Context api lets us pass a value deep into the component tree, without explicitly threading it through every component.
- Use a custom hook, manage state and effects within the hook, and only return the values that the components need which may be a value and a setValue.
- With `cy.intercept` and MSW we checked that a network request goes out versus checking that the operation caused a hook to be called. Consequently changing the hooks had no impact on the tests, or the functionality. This is why we want to test at a slightly higher level of abstraction, and why we want to verify the consequences of the implementation vs the implementation details themselves.
