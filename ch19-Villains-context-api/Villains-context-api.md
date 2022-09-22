# Context API

In this chapter we will mirror heroes into villains, and apply the Context api to villains. Context api lets us pass a value deep into the component tree, without explicitly threading it through every component. This will give us a nice contrast between passing the state as prop to child components, versus using the context to share the state down the component tree.

`Heroes.tsx` passes `heroes` as a prop to `HeroList.tsx`.

`HeroDetail` gets `hero` `{id, name, description}` state from the url with `useParams` & `useSearchParams` .

This is the extent of state management in our app. We do not really need context, but we will explore how things can be different with the context api.

## Update the current app to be more generic

Create new interfaces and types that will be utilized throughout the hero and villain groups of components.

```ts
// src/models/Villain.ts
export interface Villain {
  id: string;
  name: string;
  description: string;
}
```

```ts
// src/models/types.ts
import { Hero } from "./Hero";
import { Villain } from "./Villain";

export type HeroProperty = Hero["name"] | Hero["description"] | Hero["id"];
export type VillainProperty =
  | Villain["name"]
  | Villain["description"]
  | Villain["id"];

export type EntityRoute = "heroes" | "villains";

export type EntityType = "hero" | "villain";
```

### Update the hooks

We want to make our hooks more generic so that they can seamlessly be used in the villains group of components. In short, we will replace `useCRUDhero` hooks with `useCRUDentity`.

`useDeleteEntity` replaces `useDeleteHero`.

```ts
// src/hooks/useDeleteEntity.ts
import { Hero } from "models/Hero";
import { EntityType } from "models/types";
import { Villain } from "models/Villain";
import { useMutation, useQueryClient } from "react-query";
import { useNavigate } from "react-router-dom";
import { deleteItem } from "./api";

/**
 * Helper for DELETE to `/heroes` or `/villains` routes.
 * @returns {object} {deleteEntity, isDeleting, isDeleteError, deleteError}
 */
export function useDeleteEntity(entityType: EntityType) {
  const entityRoute = entityType === "hero" ? "heroes" : "villains";
  const navigate = useNavigate();
  const queryClient = useQueryClient();

  const mutation = useMutation(
    (item: Hero | Villain) => deleteItem(`${entityRoute}/${item.id}`),
    {
      // on success receives the original item as a second argument
      // if you recall, the first argument is the created item
      onSuccess: (_, deletedEntity: Hero | Villain) => {
        // get all the entities from the cache
        const entities: Hero[] | Villain[] =
          queryClient.getQueryData([`${entityRoute}`]) || [];
        // set the entities cache without the delete one
        queryClient.setQueryData(
          [`${entityRoute}`],
          entities.filter((h) => h.id !== deletedEntity.id)
        );

        navigate(`/${entityRoute}`);
      },
    }
  );

  return {
    deleteEntity: mutation.mutate,
    isDeleting: mutation.isLoading,
    isDeleteError: mutation.isError,
    deleteError: mutation.error,
  };
}
```

`useEntityParams` replaces `useHeroParams`.

```tsx
// src/hooks/useEntityParams.ts
import { useSearchParams } from "react-router-dom";

export function useEntityParams() {
  const [searchParams] = useSearchParams();
  const name = searchParams.get("name");
  const description = searchParams.get("description");

  return { name, description };
}
```

`useGetEntities` replaces `useGetHeroes`.

```ts
// src/hooks/useGetEntities.ts
import { EntityRoute } from "models/types";
import { useQuery } from "react-query";
import { getItem } from "./api";

/**
 * Helper for GET to `/heroes` or `/villains` routes
 * @returns {object} {entities, status, getError}
 */
export const useGetEntities = (entityRoute: EntityRoute) => {
  const query = useQuery(entityRoute, () => getItem(entityRoute), {
    suspense: true,
  });

  return {
    entities: query.data,
    status: query.status,
    getError: query.error,
  };
};
```

`usePostEntity` replaces `usePostHero`.

```ts
// src/hooks/usePostEntity.ts
import { Hero } from "models/Hero";
import { EntityType } from "models/types";
import { useMutation, useQueryClient } from "react-query";
import { useNavigate } from "react-router-dom";
import { createItem } from "./api";

/**
 * Helper for simple POST to `/heroes` route
 * @returns {object} {mutate, status, error}
 */
export function usePostEntity(entityType: EntityType) {
  const entityRoute = entityType === "hero" ? "heroes" : "villains";
  const queryClient = useQueryClient();
  const navigate = useNavigate();
  return useMutation((item: Hero) => createItem(entityRoute, item), {
    onSuccess: (newData: Hero) => {
      //  use queryClient's setQueryData to set the cache
      // takes a key as the first arg, the 2nd arg is a cb that takes the old query cache and returns the new one
      queryClient.setQueryData([entityRoute], (oldData: Hero[] | undefined) => [
        ...(oldData || []),
        newData,
      ]);

      return navigate(`/${entityRoute}`);
    },
  });
}
```

`usePutEntity` replaces `usePutHero`.

```ts
// src/hooks/usePutEntity.ts
import { Hero } from "models/Hero";
import { useMutation, useQueryClient } from "react-query";
import type { QueryClient } from "react-query";
import { useNavigate } from "react-router-dom";
import { editItem } from "./api";
import { Villain } from "models/Villain";
import { EntityType } from "models/types";

/**
 * Helper for PUT to `/heroes` route
 * @returns {object} {updateHero, isUpdating, isUpdateError, updateError}
 */
export function usePutEntity(entityType: EntityType) {
  const entityRoute = entityType === "hero" ? "heroes" : "villains";
  const queryClient = useQueryClient();
  const navigate = useNavigate();
  const mutation = useMutation(
    (item: Hero) => editItem(`${entityRoute}/${item.id}`, item),
    {
      onSuccess: (updatedEntity: Hero) => {
        updateEntityCache(entityType, updatedEntity, queryClient);
        navigate(`/${entityRoute}`);
      },
    }
  );

  return {
    updateEntity: mutation.mutate,
    isUpdating: mutation.isLoading,
    isUpdateError: mutation.isError,
    updateError: mutation.error,
  };
}

/** Replace a hero in the cache with the updated version. */
function updateEntityCache(
  entityType: EntityType,
  updatedEntity: Hero | Villain,
  queryClient: QueryClient
) {
  const entityRoute = entityType === "hero" ? "heroes" : "villains";
  // get all the heroes from the cache
  let entityCache: Hero[] | Villain[] =
    queryClient.getQueryData(entityRoute) || [];

  // find the index in the cache of the hero that's been edited
  const entityIndex = entityCache.findIndex((h) => h.id === updatedEntity.id);

  if (entityIndex !== -1) {
    // if the hero is found, replace the pre-edited hero with the updated one
    // this is just replacing an array item in place,
    // while not mutating the original array
    entityCache = entityCache.map((preEditedEntity) =>
      preEditedEntity.id === updatedEntity.id ? updatedEntity : preEditedEntity
    );
    console.log("entityCache is", entityCache);
    // use queryClient's setQueryData to set the cache
    // takes a key as the first arg, the 2nd arg is the new cache
    return queryClient.setQueryData([entityRoute], entityCache);
  } else return null;
}
```

### Update the hero components

Having changed the hooks, the hero group of components need slight modifications.

```tsx
// src/heroes/HeroDetail.tsx
import { useState, ChangeEvent } from "react";
import { useNavigate, useParams } from "react-router-dom";
import { FaUndo, FaRegSave } from "react-icons/fa";
import InputDetail from "components/InputDetail";
import ButtonFooter from "components/ButtonFooter";
import PageSpinner from "components/PageSpinner";
import ErrorComp from "components/ErrorComp";
import { useEntityParams } from "hooks/useEntityParams";
import { usePostEntity } from "hooks/usePostEntity";
import { Hero } from "models/Hero";
import { usePutEntity } from "hooks/usePutEntity";

export default function HeroDetail() {
  const { id } = useParams();
  const { name, description } = useEntityParams();
  const [hero, setHero] = useState({ id, name, description });
  const {
    mutate: createHero,
    status,
    error: postError,
  } = usePostEntity("hero");
  const {
    updateEntity: updateHero,
    isUpdating,
    isUpdateError,
  } = usePutEntity("hero");

  const navigate = useNavigate();
  const handleCancel = () => navigate("/heroes");
  const handleSave = () =>
    name ? updateHero(hero as Hero) : createHero(hero as Hero);
  const handleNameChange = (e: ChangeEvent<HTMLInputElement>) => {
    setHero({ ...hero, name: e.target.value });
  };
  const handleDescriptionChange = (e: ChangeEvent<HTMLInputElement>) => {
    setHero({ ...hero, description: e.target.value });
  };

  if (status === "loading" || isUpdating) {
    return <PageSpinner />;
  }

  if (postError || isUpdateError) {
    return <ErrorComp />;
  }

  return (
    <div data-cy="hero-detail" className="card edit-detail">
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

```tsx
// src/heroes/Heroes.tsx
import { useState } from "react";
import { useNavigate, Routes, Route } from "react-router-dom";
import ListHeader from "components/ListHeader";
import ModalYesNo from "components/ModalYesNo";
import PageSpinner from "components/PageSpinner";
import ErrorComp from "components/ErrorComp";
import HeroList from "./HeroList";
import HeroDetail from "./HeroDetail";
import { useGetEntities } from "hooks/useGetEntities";
import { useDeleteEntity } from "hooks/useDeleteEntity";
import { Hero } from "models/Hero";

export default function Heroes() {
  const [showModal, setShowModal] = useState<boolean>(false);
  const { entities: heroes, status, getError } = useGetEntities("heroes");
  const [heroToDelete, setHeroToDelete] = useState<Hero | null>(null);
  const { deleteEntity: deleteHero, isDeleteError } = useDeleteEntity("hero");

  const navigate = useNavigate();
  const addNewHero = () => navigate("/heroes/add-hero");
  const handleRefresh = () => navigate("/heroes");

  const handleCloseModal = () => {
    setHeroToDelete(null);
    setShowModal(false);
  };

  const handleDeleteHero = (hero: Hero) => () => {
    setHeroToDelete(hero);
    setShowModal(true);
  };
  const handleDeleteFromModal = () => {
    heroToDelete ? deleteHero(heroToDelete) : null;
    setShowModal(false);
  };

  if (status === "loading") {
    return <PageSpinner />;
  }

  if (getError || isDeleteError) {
    return <ErrorComp />;
  }

  return (
    <div data-cy="heroes">
      <ListHeader
        title="Heroes"
        handleAdd={addNewHero}
        handleRefresh={handleRefresh}
      />
      <div>
        <div>
          <Routes>
            <Route
              path=""
              element={
                <HeroList heroes={heroes} handleDeleteHero={handleDeleteHero} />
              }
            />
            <Route path="/add-hero" element={<HeroDetail />} />
            <Route path="/edit-hero/:id" element={<HeroDetail />} />
            <Route
              path="*"
              element={
                <HeroList heroes={heroes} handleDeleteHero={handleDeleteHero} />
              }
            />
          </Routes>
        </div>
      </div>

      {showModal && (
        <ModalYesNo
          message="Would you like to delete the hero?"
          onNo={handleCloseModal}
          onYes={handleDeleteFromModal}
        />
      )}
    </div>
  );
}
```

```tsx
// src/heroes/HeroList.tsx
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
import { Hero } from "models/Hero";
import { HeroProperty } from "models/types";

type HeroListProps = {
  heroes: Hero[];
  handleDeleteHero: (hero: Hero) => (e: MouseEvent<HTMLButtonElement>) => void;
};

export default function HeroList({ heroes, handleDeleteHero }: HeroListProps) {
  const deferredHeroes = useDeferredValue(heroes);
  const isStale = deferredHeroes !== heroes;
  const [filteredHeroes, setFilteredHeroes] = useState(deferredHeroes);
  const navigate = useNavigate();
  const [isPending, startTransition] = useTransition();

  // needed to refresh the list after deleting a hero
  useEffect(() => setFilteredHeroes(deferredHeroes), [deferredHeroes]);

  const handleSelectHero = (heroId: string) => () => {
    const hero = deferredHeroes.find((h: Hero) => h.id === heroId);
    navigate(
      `/heroes/edit-hero/${hero?.id}?name=${hero?.name}&description=${hero?.description}`
    );
  };

  /** returns a boolean whether the hero properties exist in the search field */
  const searchExists = (searchProperty: HeroProperty, searchField: string) =>
    String(searchProperty).toLowerCase().indexOf(searchField.toLowerCase()) !==
    -1;

  /** given the data and the search field, returns the data in which the search field exists */
  const searchProperties = (searchField: string, data: Hero[]) =>
    [...data].filter((item: Hero) =>
      Object.values(item).find((property: HeroProperty) =>
        searchExists(property, searchField)
      )
    );

  /** filters the heroes data to see if the any of the properties exist in the list */
  const handleSearch =
    (data: Hero[]) => (event: ChangeEvent<HTMLInputElement>) => {
      const searchField = event.target.value;

      return startTransition(() =>
        setFilteredHeroes(searchProperties(searchField, data))
      );
    };

  return (
    <div
      style={{
        opacity: isPending ? 0.5 : 1,
        color: isStale ? "dimgray" : "black",
      }}
    >
      {deferredHeroes.length > 0 && (
        <div className="card-content">
          <span>Search </span>
          <input data-cy="search" onChange={handleSearch(deferredHeroes)} />
        </div>
      )}
      &nbsp;
      <ul data-cy="hero-list" className="list">
        {filteredHeroes.map((hero, index) => (
          <li data-cy={`hero-list-item-${index}`} key={hero.id}>
            <div className="card">
              <CardContent name={hero.name} description={hero.description} />
              <footer className="card-footer">
                <ButtonFooter
                  label="Delete"
                  IconClass={FaRegSave}
                  onClick={handleDeleteHero(hero)}
                />
                <ButtonFooter
                  label="Edit"
                  IconClass={FaEdit}
                  onClick={handleSelectHero(hero.id)}
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

### Note about testing implementation details

There are no modifications needed to Cypress e2e, CT, or RTL tests with MSW because we did not test implementation details in any of them. They will all work as they are at the moment.

With `cy.intercept` and MSW we checked that a network request goes out versus checking that the operation caused a hook to be called. Consequently changing the hooks had no impact on the tests, or the functionality. This is why we want to test at a slightly higher level of abstraction, and why we want to verify the consequences of the implementation vs the implementation details themselves.

### Modify the e2e commands to be more generic

The e2e tests for villains will look exactly the same, however our commands are a specific to heroes. They can also be more generic so that mirroring the hero group of tests into villains is easier.

In the commands file we will change most references to `hero` to `entity`, and for types we will include the `villain` varieties next to `hero`. The change will require small updates to type definitions and e2e tests.

```tsx
// cypress/support/commands.ts
import { Villain } from "./../../src/models/Villain";
import { Hero } from "../../src/models/Hero";
import {
  EntityRoute,
  EntityType,
  HeroProperty,
  VillainProperty,
} from "../../src/models/types";
import data from "../fixtures/db.json";

Cypress.Commands.add("getByCy", (selector, ...args) =>
  cy.get(`[data-cy="${selector}"]`, ...args)
);

Cypress.Commands.add("getByCyLike", (selector, ...args) =>
  cy.get(`[data-cy*=${selector}]`, ...args)
);

Cypress.Commands.add("getByClassLike", (selector, ...args) =>
  cy.get(`[class*=${selector}]`, ...args)
);

Cypress.Commands.add(
  "crud",
  (
    method: "GET" | "POST" | "PUT" | "DELETE",
    route: string,
    {
      body,
      allowedToFail = false,
    }: { body?: Hero | object; allowedToFail?: boolean } = {}
  ) =>
    cy.request<Hero[] & Hero>({
      method: method,
      url: `${Cypress.env("API_URL")}/${route}`,
      body: method === "POST" || method === "PUT" ? body : undefined,
      retryOnStatusCodeFailure: !allowedToFail,
      failOnStatusCode: !allowedToFail,
    })
);

Cypress.Commands.add("resetData", () =>
  cy.crud("POST", "reset", { body: data })
);

const { _ } = Cypress;

const propExists =
  (property: HeroProperty | VillainProperty) => (entity: Hero | Villain) =>
    entity.name === property ||
    entity.description === property ||
    entity.id === property;

const getEntities = (entityRoute: EntityRoute) =>
  cy.crud("GET", entityRoute).its("body");

Cypress.Commands.add(
  "getEntityByProperty",
  (entityType: EntityType, property: HeroProperty | VillainProperty) =>
    getEntities(entityType === "hero" ? "heroes" : "villains").then(
      (entities) => _.find(entities, propExists(property))
    )
);

Cypress.Commands.add(
  "findEntityIndex",
  (entityType: EntityType, property: HeroProperty | VillainProperty) =>
    getEntities(entityType === "hero" ? "heroes" : "villains").then(
      (body: Hero[]) => ({
        entityIndex: _.findIndex(body, propExists(property)),
        entityArray: body,
      })
    )
);

Cypress.Commands.add("visitStubbedEntities", (entityRoute: EntityRoute) => {
  cy.intercept("GET", `${Cypress.env("API_URL")}/${entityRoute}`, {
    fixture: `${entityRoute}.json`,
  }).as(`stubbed${_.startCase(entityRoute)}`);
  cy.visit(`/${entityRoute}`);
  cy.wait(`@stubbed${_.startCase(entityRoute)}`);
  return cy.location("pathname").should("eq", `/${entityRoute}`);
});

Cypress.Commands.add("visitEntities", (entityRoute: EntityRoute) => {
  cy.intercept("GET", `${Cypress.env("API_URL")}/${entityRoute}`).as(
    `get${_.startCase(entityRoute)}`
  );
  cy.visit(`/${entityRoute}`);
  cy.wait(`@get${_.startCase(entityRoute)}`);
  return cy.location("pathname").should("eq", `/${entityRoute}`);
});
```

````tsx
// cypress.d.ts
/* eslint-disable @typescript-eslint/no-explicit-any */
import { MountOptions, MountReturn } from "cypress/react";
import { HeroProperty, VillainProperty, EntityType } from "models/types";
import type { Hero } from "./cypress/support/commands";

export {};
declare global {
  namespace Cypress {
    interface Chainable {
      /** Yields elements with a data-cy attribute that matches a specified selector.
       * ```
       * cy.getByCy('search-toggle') // where the selector is [data-cy="search-toggle"]
       * ```
       */
      getByCy(qaSelector: string, args?: any): Chainable<JQuery<HTMLElement>>;

      /** Yields elements with data-cy attribute that partially matches a specified selector.
       * ```
       * cy.getByCyLike('chat-button') // where the selector is [data-cy="chat-button-start-a-new-claim"]
       * ```
       */
      getByCyLike(
        qaSelector: string,
        args?: any
      ): Chainable<JQuery<HTMLElement>>;

      /** Yields the element that partially matches the css class
       * ```
       * cy.getByClassLike('StyledIconBase') // where the class is class="StyledIconBase-ea9ulj-0 lbJwfL"
       * ```
       */
      getByClassLike(
        qaSelector: string,
        args?: any
      ): Chainable<JQuery<HTMLElement>>;

      /** Mounts a React node
       * @param component React Node to mount
       * @param options Additional options to pass into mount
       */
      mount(
        component: React.ReactNode,
        options?: MountOptions
      ): Cypress.Chainable<MountReturn>;

      /** Mounts the component wrapped by all the providers:
       * QueryClientProvider, ErrorBoundary, Suspense, BrowserRouter
       * @param component React Node to mount
       * @param options Additional options to pass into mount
       */
      wrappedMount(
        component: React.ReactNode,
        options?: MountOptions
      ): Cypress.Chainable<MountReturn>;

      /** Visits heroes or villains routes, uses real network, verifies path */
      visitEntities(entityRoute: EntityRoute): Cypress.Chainable<string>;

      /** Visits heroes or villains routes, uses stubbed network, verifies path */
      visitStubbedEntities(entityRoute: EntityRoute): Cypress.Chainable<string>;

      /**
       * Gets an entity by name.
       * ```js
       * cy.getEntityByName(newHero.name).then(myHero => ...)
       * ```
       * @param name: Hero['name']
       */
      getEntityByProperty(
        entityType: EntityType,
        property: HeroProperty | VillainProperty
      ): Cypress.Chainable<Hero | Villain>;

      /**
       * Given a hero property (name, description or id),
       * returns the index of the hero, and the entire collection, as an object.
       */
      findEntityIndex(
        entityType: EntityType,
        property: HeroProperty
      ): Cypress.Chainable<{ entityIndex: number; entityArray: Hero[] }>;

      /**
       * Performs crud operations GET, POST, PUT and DELETE.
       *
       * `body` and `allowedToFail are optional.
       *
       * If they are not passed in, body is empty but `allowedToFail` still is `false`.
       *
       * If the body is passed in and the method is `POST` or `PUT`, the payload will be taken,
       * otherwise undefined for `GET` and `DELETE`.
       * @param method
       * @param route
       * @param options: {body?: Hero | object; allowedToFail?: boolean}
       */
      crud(
        method: "GET" | "POST" | "PUT" | "DELETE",
        route: string,
        {
          body,
          allowedToFail = false,
        }: { body?: Hero | object; allowedToFail?: boolean } = {}
      ): Cypress.Chainable<Response<Hero[] & Hero>>;

      /**
       * Resets the data in the database to the initial data.
       */
      resetData(): Cypress.Chainable<Response<Hero[] & Hero>>;
    }
  }
}
````

```ts
// cypress/e2e/create-hero.cy.ts
import { faker } from "@faker-js/faker";
describe("Create hero", () => {
  before(cy.resetData);

  const navToAddHero = () => {
    cy.location("pathname").should("eq", "/heroes");
    cy.getByCy("add-button").click();
    cy.location("pathname").should("eq", "/heroes/add-hero");
    cy.getByCy("hero-detail").should("be.visible");
    cy.getByCy("input-detail-id").should("not.exist");
  };

  it("should go through the refresh flow (ui-integration)", () => {
    cy.visitStubbedEntities("heroes");
    navToAddHero();

    cy.getByCy("refresh-button").click();
    cy.location("pathname").should("eq", "/heroes");
    cy.getByCy("hero-list").should("be.visible");
  });

  it("should go through the cancel flow and perform direct navigation (ui-integration)", () => {
    cy.intercept("GET", `${Cypress.env("API_URL")}/heroes`, {
      fixture: "heroes",
    }).as("stubbedGetHeroes");
    cy.visit("/heroes/add-hero");
    cy.wait("@stubbedGetHeroes");

    cy.getByCy("cancel-button").click();
    cy.location("pathname").should("eq", "/heroes");
    cy.getByCy("hero-list").should("be.visible");
  });

  it("should go through the add hero flow (ui-e2e)", () => {
    cy.visitEntities("heroes");
    navToAddHero();

    const newHero = {
      name: faker.internet.userName(),
      description: `description ${faker.internet.userName()}`,
    };
    cy.getByCy("input-detail-name").type(newHero.name);
    cy.getByCy("input-detail-description").type(newHero.description);
    cy.getByCy("save-button").click();

    cy.location("pathname").should("eq", "/heroes");

    cy.getByCy("heroes").should("be.visible");
    cy.getByCyLike("hero-list-item").should("have.length.gt", 0);
    cy.getByCy("hero-list")
      .should("contain", newHero.name)
      .and("contain", newHero.description);

    cy.getEntityByProperty("hero", newHero.name).then((myHero) =>
      cy.crud("DELETE", `heroes/${myHero.id}`)
    );
  });
});
```

```ts
// cypress/e2e/delete-hero.cy.ts
import { faker } from "@faker-js/faker";
import { Hero } from "../../src/models/Hero";
describe("Delete hero", () => {
  before(cy.resetData);

  const yesOnModal = () =>
    cy.getByCy("modal-yes-no").within(() => cy.getByCy("button-yes").click());

  it("should go through the cancel flow (ui-integration)", () => {
    cy.visitStubbedEntities("heroes");

    cy.getByCy("delete-button").first().click();
    cy.getByCy("modal-yes-no").within(() => cy.getByCy("button-no").click());
    cy.getByCy("heroes").should("be.visible");
    cy.get("modal-yes-no").should("not.exist");
  });

  it("should go through the edit flow (ui-e2e)", () => {
    const hero: Hero = {
      id: faker.datatype.uuid(),
      name: faker.internet.userName(),
      description: `description ${faker.internet.userName()}`,
    };

    cy.crud("POST", "heroes", { body: hero });

    cy.visitEntities("heroes");

    cy.findEntityIndex("hero", hero.id).then(
      ({ entityIndex: heroIndex, entityArray: heroArray }) => {
        cy.getByCy("delete-button").eq(heroIndex).click();

        yesOnModal();

        cy.getByCy("hero-list")
          .should("be.visible")
          .should("not.contain", heroArray[heroIndex].name)
          .and("not.contain", heroArray[heroIndex].description);
      }
    );
  });
});
```

```ts
// cypress/e2e/edit-hero.cy.ts
import { faker } from "@faker-js/faker";
import { Hero } from "../../src/models/Hero";
describe("Edit hero", () => {
  before(cy.resetData);

  /** Verifies hero info on Edit page */
  const verifyHero = (heroes: Hero[], heroIndex: number) => {
    cy.location("pathname").should("include", "/heroes/edit-hero/");
    cy.getByCy("hero-detail").should("be.visible");
    cy.getByCy("input-detail-id").should("be.visible");
    cy.findByDisplayValue(heroes[heroIndex].id);
    cy.findByDisplayValue(heroes[heroIndex].name);
    cy.findByDisplayValue(heroes[heroIndex].description);
  };

  const randomHeroIndex = (heroes: Hero[]) =>
    Cypress._.random(0, heroes.length - 1);

  it("should go through the cancel flow for a random hero (ui-integration)", () => {
    cy.visitStubbedEntities("heroes");

    cy.fixture("heroes").then((heroes) => {
      const heroIndex = randomHeroIndex(heroes);
      cy.getByCy("edit-button").eq(heroIndex).click();
      verifyHero(heroes, heroIndex);
    });

    cy.getByCy("cancel-button").click();
    cy.location("pathname").should("eq", "/heroes");
    cy.getByCy("hero-list").should("be.visible");
  });

  it("should go through the PUT error flow (ui-integration)", () => {
    cy.visitStubbedEntities("heroes");

    cy.fixture("heroes").then((heroes) => {
      const heroIndex = randomHeroIndex(heroes);
      cy.getByCy("edit-button").eq(heroIndex).click();
      verifyHero(heroes, heroIndex);
    });

    cy.intercept("PUT", `${Cypress.env("API_URL")}/heroes/*`, {
      statusCode: 500,
      delay: 100,
    }).as("isUpdateError");

    cy.getByCy("save-button").click();
    cy.getByCy("spinner");
    cy.wait("@isUpdateError");
    cy.getByCy("error");
  });

  it("should navigate to add from an existing hero (ui-integration)", () => {
    cy.visitStubbedEntities("heroes");

    cy.fixture("heroes").then((heroes) => {
      const heroIndex = randomHeroIndex(heroes);
      cy.getByCy("edit-button").eq(heroIndex).click();
      verifyHero(heroes, heroIndex);

      cy.getByCy("add-button").click();
      cy.getByCy("input-detail-id").should("not.exist");
      cy.findByDisplayValue(heroes[heroIndex].name).should("not.exist");
      cy.findByDisplayValue(heroes[heroIndex].description).should("not.exist");
    });
  });

  it("should go through the edit flow (ui-e2e)", () => {
    const newHero: Hero = {
      id: faker.datatype.uuid(),
      name: faker.internet.userName(),
      description: `description ${faker.internet.userName()}`,
    };

    cy.crud("POST", "heroes", { body: newHero });

    cy.visit(`heroes/edit-hero/${newHero.id}`, {
      qs: { name: newHero.name, description: newHero.description },
    });

    const editedHero = {
      name: faker.internet.userName(),
      description: `description ${faker.internet.userName()}`,
    };

    cy.getByCy("input-detail-name")
      .find(".input")
      .clear()
      .type(`${editedHero.name}`);
    cy.getByCy("input-detail-description")
      .find(".input")
      .clear()
      .type(`${editedHero.description}`);
    cy.getByCy("save-button").click();

    cy.getByCy("hero-list")
      .should("be.visible")
      .should("contain", editedHero.name)
      .and("contain", editedHero.description);

    cy.getEntityByProperty("hero", newHero.id).then((myHero: Hero) =>
      cy.crud("DELETE", `heroes/${myHero.id}`)
    );
  });
});
```

## Create a villains mirror of the heroes

Aligned with the theme of the book, we will first create villain related tests.

> If your local copy of heroes versions of the test are slightly different, you can optionally tweak them.

### Mirror the e2e tests

We create 3 new e2e tests, which are villain mirrors of the hero versions. We also enhance the remaining tests to check for villain related features.

```ts
// cypress/e2e/create-villain.cy.ts
import { faker } from "@faker-js/faker";
describe("Create villain", () => {
  before(cy.resetData);

  const navToAddVillain = () => {
    cy.location("pathname").should("eq", "/villains");
    cy.getByCy("add-button").click();
    cy.location("pathname").should("eq", "/villains/add-villain");
    cy.getByCy("villain-detail").should("be.visible");
    cy.getByCy("input-detail-id").should("not.exist");
  };

  it("should go through the refresh flow (ui-integration)", () => {
    cy.visitStubbedEntities("villains");
    navToAddVillain();

    cy.getByCy("refresh-button").click();
    cy.location("pathname").should("eq", "/villains");
    cy.getByCy("villain-list").should("be.visible");
  });

  it("should go through the cancel flow and perform direct navigation (ui-integration)", () => {
    cy.intercept("GET", `${Cypress.env("API_URL")}/villains`, {
      fixture: "villains",
    }).as("stubbedGetVillains");
    cy.visit("/villains/add-villain");
    cy.wait("@stubbedGetVillains");

    cy.getByCy("cancel-button").click();
    cy.location("pathname").should("eq", "/villains");
    cy.getByCy("villain-list").should("be.visible");
  });

  it("should go through the add villain flow (ui-e2e)", () => {
    cy.visitEntities("villains");
    navToAddVillain();

    const newVillain = {
      name: faker.internet.userName(),
      description: `description ${faker.internet.userName()}`,
    };
    cy.getByCy("input-detail-name").type(newVillain.name);
    cy.getByCy("input-detail-description").type(newVillain.description);
    cy.getByCy("save-button").click();

    cy.location("pathname").should("eq", "/villains");

    cy.getByCy("villains").should("be.visible");
    cy.getByCyLike("villain-list-item").should("have.length.gt", 0);
    cy.getByCy("villain-list")
      .should("contain", newVillain.name)
      .and("contain", newVillain.description);

    cy.getEntityByProperty("villain", newVillain.name).then((myVillain) =>
      cy.crud("DELETE", `villains/${myVillain.id}`)
    );
  });
});
```

```ts
// cypress/e2e/delete-villain.cy.ts
import { faker } from "@faker-js/faker";
import { Villain } from "../../src/models/Villain";
describe("Delete villain", () => {
  before(cy.resetData);

  const yesOnModal = () =>
    cy.getByCy("modal-yes-no").within(() => cy.getByCy("button-yes").click());

  it("should go through the cancel flow (ui-integration)", () => {
    cy.visitStubbedEntities("villains");

    cy.getByCy("delete-button").first().click();
    cy.getByCy("modal-yes-no").within(() => cy.getByCy("button-no").click());
    cy.getByCy("villains").should("be.visible");
    cy.get("modal-yes-no").should("not.exist");
  });

  it("should go through the edit flow (ui-e2e)", () => {
    const villain: Villain = {
      id: faker.datatype.uuid(),
      name: faker.internet.userName(),
      description: `description ${faker.internet.userName()}`,
    };

    cy.crud("POST", "villains", { body: villain });

    cy.visitEntities("villains");

    cy.findEntityIndex("villain", villain.id).then(
      ({ entityIndex: villainIndex, entityArray: villainArray }) => {
        cy.getByCy("delete-button").eq(villainIndex).click();

        yesOnModal();

        cy.getByCy("villain-list")
          .should("be.visible")
          .should("not.contain", villainArray[villainIndex].name)
          .and("not.contain", villainArray[villainIndex].description);
      }
    );
  });
});
```

```ts
// cypress/e2e/edit-villain.cy.ts
import { faker } from "@faker-js/faker";
import { Villain } from "../../src/models/Villain";
describe("Edit villain", () => {
  before(cy.resetData);

  /** Verifies villain info on Edit page */
  const verifyVillain = (villains: Villain[], villainIndex: number) => {
    cy.location("pathname").should("include", "/villains/edit-villain/");
    cy.getByCy("villain-detail").should("be.visible");
    cy.getByCy("input-detail-id").should("be.visible");
    cy.findByDisplayValue(villains[villainIndex].id);
    cy.findByDisplayValue(villains[villainIndex].name);
    cy.findByDisplayValue(villains[villainIndex].description);
  };

  const randomVillainIndex = (villains: Villain[]) =>
    Cypress._.random(0, villains.length - 1);

  it("should go through the cancel flow for a random villain (ui-integration)", () => {
    cy.visitStubbedEntities("villains");

    cy.fixture("villains").then((villains) => {
      const villainIndex = randomVillainIndex(villains);
      cy.getByCy("edit-button").eq(villainIndex).click();
      verifyVillain(villains, villainIndex);
    });

    cy.getByCy("cancel-button").click();
    cy.location("pathname").should("eq", "/villains");
    cy.getByCy("villain-list").should("be.visible");
  });

  it("should go through the PUT error flow (ui-integration)", () => {
    cy.visitStubbedEntities("villains");

    cy.fixture("villains").then((villains) => {
      const villainIndex = randomVillainIndex(villains);
      cy.getByCy("edit-button").eq(villainIndex).click();
      verifyVillain(villains, villainIndex);
    });

    cy.intercept("PUT", `${Cypress.env("API_URL")}/villains/*`, {
      statusCode: 500,
      delay: 100,
    }).as("isUpdateError");

    cy.getByCy("save-button").click();
    cy.getByCy("spinner");
    cy.wait("@isUpdateError");
    cy.getByCy("error");
  });

  it("should navigate to add from an existing villain (ui-integration)", () => {
    cy.visitStubbedEntities("villains");

    cy.fixture("villains").then((villains) => {
      const villainIndex = randomVillainIndex(villains);
      cy.getByCy("edit-button").eq(villainIndex).click();
      verifyVillain(villains, villainIndex);

      cy.getByCy("add-button").click();
      cy.getByCy("input-detail-id").should("not.exist");
      cy.findByDisplayValue(villains[villainIndex].name).should("not.exist");
      cy.findByDisplayValue(villains[villainIndex].description).should(
        "not.exist"
      );
    });
  });

  it("should go through the edit flow (ui-e2e)", () => {
    const newVillain: Villain = {
      id: faker.datatype.uuid(),
      name: faker.internet.userName(),
      description: `description ${faker.internet.userName()}`,
    };

    cy.crud("POST", "villains", { body: newVillain });

    cy.visit(`villains/edit-villain/${newVillain.id}`, {
      qs: { name: newVillain.name, description: newVillain.description },
    });

    const editedVillain = {
      name: faker.internet.userName(),
      description: `description ${faker.internet.userName()}`,
    };

    cy.getByCy("input-detail-name")
      .find(".input")
      .clear()
      .type(`${editedVillain.name}`);
    cy.getByCy("input-detail-description")
      .find(".input")
      .clear()
      .type(`${editedVillain.description}`);
    cy.getByCy("save-button").click();

    cy.getByCy("villain-list")
      .should("be.visible")
      .should("contain", editedVillain.name)
      .and("contain", editedVillain.description);

    cy.getEntityByProperty("villain", newVillain.id).then(
      (myVillain: Villain) => cy.crud("DELETE", `villains/${myVillain.id}`)
    );
  });
});
```

The backend test needs a new block to cover villains.

```ts
// cypress/e2e/backend/crud.cy.ts
import { faker } from "@faker-js/faker";
import { Hero } from "../../../src/models/Hero";

describe("Backend e2e", () => {
  const assertProperties = (entity: Hero) => {
    expect(entity.id).to.be.a("string");
    expect(entity.name).to.be.a("string");
    expect(entity.description).to.be.a("string");
  };

  before(() => cy.resetData());

  it("should GET heroes and villains ", () => {
    cy.crud("GET", "heroes")
      .its("body")
      .should("have.length.gt", 0)
      .each(assertProperties);

    cy.crud("GET", "villains")
      .its("body")
      .should("have.length.gt", 0)
      .each(assertProperties);
  });

  it("should CRUD a new hero entity", () => {
    const newHero = {
      id: faker.datatype.uuid(),
      name: faker.internet.userName(),
      description: `description ${faker.internet.userName()}`,
    };

    cy.crud("POST", "heroes", { body: newHero })
      .its("status")
      .should("eq", 201);

    cy.crud("GET", "heroes")
      .its("body")
      .then((body) => {
        expect(body.at(-1)).to.deep.eq(newHero);
      });

    const editedHero = { ...newHero, name: "Murat" };
    cy.crud("PUT", `heroes/${editedHero.id}`, { body: editedHero })
      .its("status")
      .should("eq", 200);
    cy.crud("GET", `heroes/${editedHero.id}`)
      .its("body")
      .should("deep.eq", editedHero);

    cy.crud("DELETE", `heroes/${editedHero.id}`)
      .its("status")
      .should("eq", 200);
    cy.crud("GET", `heroes/${editedHero.id}`, { allowedToFail: true })
      .its("status")
      .should("eq", 404);
  });

  it("should CRUD a new villain entity", () => {
    const newVillain = {
      id: faker.datatype.uuid(),
      name: faker.internet.userName(),
      description: `description ${faker.internet.userName()}`,
    };

    cy.crud("POST", "villains", { body: newVillain })
      .its("status")
      .should("eq", 201);

    cy.crud("GET", "villains")
      .its("body")
      .then((body) => {
        expect(body.at(-1)).to.deep.eq(newVillain);
      });

    const editedVillain = { ...newVillain, name: "Murat" };
    cy.crud("PUT", `villains/${editedVillain.id}`, { body: editedVillain })
      .its("status")
      .should("eq", 200);
    cy.crud("GET", `villains/${editedVillain.id}`)
      .its("body")
      .should("deep.eq", editedVillain);

    cy.crud("DELETE", `villains/${editedVillain.id}`)
      .its("status")
      .should("eq", 200);
    cy.crud("GET", `villains/${editedVillain.id}`, { allowedToFail: true })
      .its("status")
      .should("eq", 404);
  });
});
```

The routes-nav needs a new test to cover villains route.

```ts
// cypress/e2e/routes-nav.cy.ts
describe("routes navigation (ui-integration)", () => {
  beforeEach(() => {
    cy.intercept("GET", `${Cypress.env("API_URL")}/heroes`, {
      fixture: "heroes",
    }).as("stubbedGetHeroes");
  });
  it("should land on baseUrl, redirect to /heroes", () => {
    cy.visit("/");
    cy.getByCy("header-bar").should("be.visible");
    cy.getByCy("nav-bar").should("be.visible");

    cy.location('pathname').should('eq' "/heroes");
    cy.getByCy("heroes").should("be.visible");
  });

  it("should direct-navigate to /heroes", () => {
    const route = "/heroes";
    cy.visit(route);
    cy.location('pathname').should('eq' route);
    cy.getByCy("heroes").should("be.visible");
  });

  it("should direct-navigate to /villains", () => {
    const route = "/villains";
    cy.visit(route);
    cy.location('pathname').should('eq' route);
    cy.getByCy("villains").should("be.visible");
  });

  it("should land on not found when visiting an non-existing route", () => {
    const route = "/route48";
    cy.visit(route);
    cy.location('pathname').should('eq' route);
    cy.getByCy("not-found").should("be.visible");
  });

  it("should direct-navigate to about", () => {
    const route = "/about";
    cy.visit(route);
    cy.location('pathname').should('eq' route);
    cy.getByCy("about").contains("CCTDD");
  });

  it("should cover route history with browser back and forward", () => {
    cy.visit("/about");
    const routes = ["villains", "heroes", "about"];
    cy.wrap(routes).each((route: string) =>
      cy.get(`[href="/${route}"]`).click()
    );

    const lastIndex = routes.length - 1;
    cy.location('pathname').should('eq' routes[lastIndex]);
    cy.go("back");
    cy.location('pathname').should('eq' routes[lastIndex - 1]);
    cy.go("back");
    cy.location('pathname').should('eq' routes[lastIndex - 2]);
    cy.go("forward").go("forward");
    cy.location('pathname').should('eq' routes[lastIndex]);
  });
});
```

### Mirror the Cypress component tests

```tsx
// src/villains/VillainDetail.cy.tsx
import VillainDetail from "./VillainDetail";
import "../styles.scss";

describe("VillainDetail", () => {
  beforeEach(() => {
    cy.wrappedMount(<VillainDetail />);
  });

  it("should handle Save", () => {
    cy.intercept("POST", "*", { statusCode: 200 }).as("postVillain");
    cy.getByCy("save-button").click();
    cy.wait("@postVillain");
  });

  it("should handle non-200 Save", () => {
    cy.intercept("POST", "*", { statusCode: 400, delay: 100 }).as(
      "postVillain"
    );
    cy.getByCy("save-button").click();
    cy.getByCy("spinner");
    cy.wait("@postVillain");
    cy.getByCy("error");
  });

  it("should handle Cancel", () => {
    cy.getByCy("cancel-button").click();
    cy.location("pathname").should("eq", "/villains");
  });

  it("should handle name change", () => {
    const newVillainName = "abc";
    cy.getByCy("input-detail-name").type(newVillainName);

    cy.findByDisplayValue(newVillainName).should("be.visible");
  });

  it("should handle description change", () => {
    const newVillainDescription = "123";
    cy.getByCy("input-detail-description").type(newVillainDescription);

    cy.findByDisplayValue(newVillainDescription).should("be.visible");
  });

  it("id: false, name: false - should verify the minimal state of the component", () => {
    cy.get("p").then(($el) => cy.wrap($el.text()).should("equal", ""));
    cy.getByCyLike("input-detail").should("have.length", 2);
    cy.getByCy("input-detail-id").should("not.exist");

    cy.findByPlaceholderText("e.g. Colleen").should("be.visible");
    cy.findByPlaceholderText("e.g. dance fight!").should("be.visible");

    cy.getByCy("save-button").should("be.visible");
    cy.getByCy("cancel-button").should("be.visible");
  });
});
```

```tsx
// src/villains/VillainList.cy.tsx
import VillainList from "./VillainList";
import "../styles.scss";
import villains from "../../cypress/fixtures/villains.json";

describe("VillainList", () => {
  it("no villains should not display a list nor search bar", () => {
    cy.wrappedMount(
      <VillainList
        villains={[]}
        handleDeleteVillain={cy.stub().as("handleDeleteVillain")}
      />
    );

    cy.getByCy("villain-list").should("exist");
    cy.getByCyLike("villain-list-item").should("not.exist");
    cy.getByCy("search").should("not.exist");
  });

  context("with villains in the list", () => {
    beforeEach(() => {
      cy.wrappedMount(
        <VillainList
          villains={villains}
          handleDeleteVillain={cy.stub().as("handleDeleteVillain")}
        />
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

      cy.get("footer").within(() => {
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
// src/villains/Villains.cy.tsx
import Villains from "./Villains";
import "../styles.scss";

describe("Villains", () => {
  it("should see error on initial load with GET", () => {
    Cypress.on("uncaught:exception", () => false);
    cy.clock();
    cy.intercept("GET", `${Cypress.env("API_URL")}/villains`, {
      statusCode: 400,
      delay: 100,
    }).as("notFound");

    cy.wrappedMount(<Villains />);

    cy.getByCy("page-spinner").should("be.visible");
    Cypress._.times(4, () => {
      cy.tick(5000);
      cy.wait("@notFound");
    });

    cy.getByCy("error");
  });

  context("200 flows", () => {
    beforeEach(() => {
      cy.intercept("GET", `${Cypress.env("API_URL")}/villains`, {
        fixture: "villains.json",
      }).as("getVillains");

      cy.wrappedMount(<Villains />);
    });

    it("should display the villain list on render, and go through villain add & refresh flow", () => {
      cy.wait("@getVillains");

      cy.getByCy("list-header").should("be.visible");
      cy.getByCy("villain-list").should("be.visible");

      cy.getByCy("add-button").click();
      cy.location("pathname").should("eq", "/villains/add-villain");

      cy.getByCy("refresh-button").click();
      cy.location("pathname").should("eq", "/villains");
    });

    const invokeVillainDelete = () => {
      cy.getByCy("delete-button").first().click();
      cy.getByCy("modal-yes-no").should("be.visible");
    };
    it("should go through the modal flow, and cover error on DELETE", () => {
      cy.getByCy("modal-yes-no").should("not.exist");

      cy.log("do not delete flow");
      invokeVillainDelete();
      cy.getByCy("button-no").click();
      cy.getByCy("modal-yes-no").should("not.exist");

      cy.log("delete flow");
      invokeVillainDelete();
      cy.intercept("DELETE", "*", { statusCode: 500 }).as("deleteVillain");

      cy.getByCy("button-yes").click();
      cy.wait("@deleteVillain");
      cy.getByCy("modal-yes-no").should("not.exist");
      cy.getByCy("error").should("be.visible");
    });
  });
});
```

Create a new fixture for villains at `cypress/fixtures/villains.json`.

```json
[
  {
    "id": "VillainMadelyn",
    "name": "Madelyn",
    "description": "the cat whisperer"
  },
  {
    "id": "VillainHaley",
    "name": "Haley",
    "description": "pen wielder"
  },
  {
    "id": "VillainElla",
    "name": "Ella",
    "description": "fashionista"
  },
  {
    "id": "VillainLandon",
    "name": "Landon",
    "description": "Mandalorian mauler"
  }
]
```

### Mirror the RTL tests

```tsx
// src/villains/VillainDetail.test.tsx
import VillainDetail from "./VillainDetail";
import "@testing-library/jest-dom";
import { wrappedRender, act, screen, waitFor } from "test-utils";
import userEvent from "@testing-library/user-event";

describe("VillainDetail", () => {
  beforeEach(() => {
    wrappedRender(<VillainDetail />);
  });

  // with msw, it is not recommended to use verify XHR calls going out of the app
  // instead, the advice is the verify the changes in the UI
  // alas, sometimes there are no changes in the component itself
  // therefore we cannot test everything 1:1 versus Cypress component test
  // should handle Save and should handle non-200 Save have no RTL mirrors

  it("should handle Cancel", async () => {
    // code that causes React state updates (ex: BrowserRouter)
    // should be wrapped into act(...):
    // userEvent.click(await screen.findByTestId('cancel-button')) // won't work
    act(() => screen.getByTestId("cancel-button").click());

    expect(window.location.pathname).toBe("/villains");
  });

  it("should handle name change", async () => {
    const newVillainName = "abc";
    const inputDetailName = await screen.findByPlaceholderText("e.g. Colleen");
    userEvent.type(inputDetailName, newVillainName);

    await waitFor(async () =>
      expect(inputDetailName).toHaveDisplayValue(newVillainName)
    );
  });

  const inputDetailDescription = async () =>
    screen.findByPlaceholderText("e.g. dance fight!");

  it("should handle description change", async () => {
    const newVillainDescription = "123";

    userEvent.type(await inputDetailDescription(), newVillainDescription);
    await waitFor(async () =>
      expect(await inputDetailDescription()).toHaveDisplayValue(
        newVillainDescription
      )
    );
  });

  it("id: false, name: false - should verify the minimal state of the component", async () => {
    expect(await screen.findByTestId("input-detail-name")).toBeVisible();
    expect(await screen.findByTestId("input-detail-description")).toBeVisible();
    expect(screen.queryByTestId("input-detail-id")).not.toBeInTheDocument();

    expect(await inputDetailDescription()).toBeVisible();

    expect(await screen.findByTestId("save-button")).toBeVisible();
    expect(await screen.findByTestId("cancel-button")).toBeVisible();
  });
});
```

```tsx
// src/villains/VillainList.test.tsx
import VillainList from "./VillainList";
import { wrappedRender, screen, waitFor } from "test-utils";
import userEvent from "@testing-library/user-event";
import { villains } from "../../db.json";

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
      wrappedRender(<VillainList handleDeleteVillain={handleDeleteVillain} />);
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

```tsx
// src/villains/Villains.test.tsx
import Villains from "./Villains";
import { wrappedRender, screen, waitForElementToBeRemoved } from "test-utils";
import userEvent from "@testing-library/user-event";
import { rest } from "msw";
import { setupServer } from "msw/node";
import { villains } from "../../db.json";

describe("Villains", () => {
  // mute the expected console.error message, because we are mocking non-200 responses
  // eslint-disable-next-line @typescript-eslint/no-empty-function
  jest.spyOn(console, "error").mockImplementation(() => {});

  beforeEach(() => wrappedRender(<Villains />));

  it("should see error on initial load with GET", async () => {
    const handlers = [
      rest.get(
        `${process.env.REACT_APP_API_URL}/villains`,
        async (_req, res, ctx) => res(ctx.status(500))
      ),
    ];
    const server = setupServer(...handlers);
    server.listen({
      onUnhandledRequest: "warn",
    });
    jest.useFakeTimers();

    expect(await screen.findByTestId("page-spinner")).toBeVisible();

    jest.advanceTimersByTime(25000);
    await waitForElementToBeRemoved(
      () => screen.queryByTestId("page-spinner"),
      {
        timeout: 25000,
      }
    );

    expect(await screen.findByTestId("error")).toBeVisible();
    jest.useRealTimers();
    server.resetHandlers();
    server.close();
  });

  describe("200 flows", () => {
    const handlers = [
      rest.get(
        `${process.env.REACT_APP_API_URL}/villains`,
        async (_req, res, ctx) => res(ctx.status(200), ctx.json(villains))
      ),
      rest.delete(
        `${process.env.REACT_APP_API_URL}/villains/${villains[0].id}`, // use /.*/ for all requests
        async (_req, res, ctx) =>
          res(ctx.status(400), ctx.json("expected error"))
      ),
    ];
    const server = setupServer(...handlers);
    beforeAll(() => {
      server.listen({
        onUnhandledRequest: "warn",
      });
    });
    afterEach(server.resetHandlers);
    afterAll(server.close);

    it("should display the villain list on render, and go through villain add & refresh flow", async () => {
      expect(await screen.findByTestId("list-header")).toBeVisible();
      expect(await screen.findByTestId("villain-list")).toBeVisible();

      await userEvent.click(await screen.findByTestId("add-button"));
      expect(window.location.pathname).toBe("/villains/add-villain");

      await userEvent.click(await screen.findByTestId("refresh-button"));
      expect(window.location.pathname).toBe("/villains");
    });

    const deleteButtons = async () => screen.findAllByTestId("delete-button");
    const modalYesNo = async () => screen.findByTestId("modal-yes-no");
    const maybeModalYesNo = () => screen.queryByTestId("modal-yes-no");
    const invokeVillainDelete = async () => {
      userEvent.click((await deleteButtons())[0]);
      expect(await modalYesNo()).toBeVisible();
    };

    it("should go through the modal flow, and cover error on DELETE", async () => {
      expect(screen.queryByTestId("modal-dialog")).not.toBeInTheDocument();

      await invokeVillainDelete();
      await userEvent.click(await screen.findByTestId("button-no"));
      expect(maybeModalYesNo()).not.toBeInTheDocument();

      await invokeVillainDelete();
      await userEvent.click(await screen.findByTestId("button-yes"));

      expect(maybeModalYesNo()).not.toBeInTheDocument();
      expect(await screen.findByTestId("error")).toBeVisible();
      expect(screen.queryByTestId("modal-dialog")).not.toBeInTheDocument();
    });
  });
});
```

### Enhance `App.cy.tsx` and `App.test.tsx`

```tsx
// src/App.cy.tsx
import App from "./App";

describe("ct sanity", () => {
  it("should render the App", () => {
    cy.intercept("GET", `${Cypress.env("API_URL")}/heroes`, {
      fixture: "heroes.json",
    }).as("getHeroes");

    cy.intercept("GET", `${Cypress.env("API_URL")}/villains`, {
      fixture: "villains.json",
    }).as("getVillains");

    cy.mount(<App />);
    cy.getByCy("not-found").should("be.visible");

    cy.contains("Heroes").click();
    cy.getByCy("heroes").should("be.visible");

    cy.contains("Villains").click();
    cy.getByCy("villains").should("be.visible");

    cy.contains("About").click();
    cy.getByCy("about").should("be.visible");
  });
});
```

```tsx
// src/App.test.tsx
import { act, render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import App from "./App";
import { heroes, villains } from "../db.json";

import { rest } from "msw";
import { setupServer } from "msw/node";

describe("200 flow", () => {
  const handlers = [
    rest.get(
      `${process.env.REACT_APP_API_URL}/heroes`,
      async (_req, res, ctx) => res(ctx.status(200), ctx.json(heroes))
    ),
    rest.get(
      `${process.env.REACT_APP_API_URL}/villains`,
      async (_req, res, ctx) => res(ctx.status(200), ctx.json(villains))
    ),
  ];
  const server = setupServer(...handlers);
  beforeAll(() => {
    server.listen({
      onUnhandledRequest: "warn",
    });
  });
  afterEach(server.resetHandlers);
  afterAll(server.close);

  test("renders tour of heroes", async () => {
    render(<App />);
    await act(() => new Promise((r) => setTimeout(r, 0))); // spinner

    await userEvent.click(screen.getByText("About"));
    expect(await screen.findByTestId("about")).toBeVisible();

    await userEvent.click(screen.getByText("Heroes"));
    expect(await screen.findByTestId("heroes")).toBeVisible();

    await userEvent.click(screen.getByText("Villains"));
    expect(await screen.findByTestId("villains")).toBeVisible();
  });
});
```

### Mirror the 3 hero components to villain components

We are creating 3 components for villains, mirroring heroes group as they are.

```tsx
// src/villains/VillainDetail.tsx
import { useState, ChangeEvent } from "react";
import { useNavigate, useParams } from "react-router-dom";
import { FaUndo, FaRegSave } from "react-icons/fa";
import InputDetail from "components/InputDetail";
import ButtonFooter from "components/ButtonFooter";
import PageSpinner from "components/PageSpinner";
import ErrorComp from "components/ErrorComp";
import { useEntityParams } from "hooks/useEntityParams";
import { usePostEntity } from "hooks/usePostEntity";
import { Villain } from "models/Villain";
import { usePutEntity } from "hooks/usePutEntity";

export default function VillainDetail() {
  const { id } = useParams();
  const { name, description } = useEntityParams();
  const [villain, setVillain] = useState({ id, name, description });
  const {
    mutate: createVillain,
    status,
    error: postError,
  } = usePostEntity("villain");
  const {
    updateEntity: updateVillain,
    isUpdating,
    isUpdateError,
  } = usePutEntity("villain");

  const navigate = useNavigate();
  const handleCancel = () => navigate("/villains");
  const handleSave = () =>
    name
      ? updateVillain(villain as Villain)
      : createVillain(villain as Villain);
  const handleNameChange = (e: ChangeEvent<HTMLInputElement>) => {
    setVillain({ ...villain, name: e.target.value });
  };
  const handleDescriptionChange = (e: ChangeEvent<HTMLInputElement>) => {
    setVillain({ ...villain, description: e.target.value });
  };

  if (status === "loading" || isUpdating) {
    return <PageSpinner />;
  }

  if (postError || isUpdateError) {
    return <ErrorComp />;
  }

  return (
    <div data-cy="villain-detail" className="card edit-detail">
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

```tsx
// src/villains/VillainList.tsx
import VillainList from "./VillainList";
import { wrappedRender, screen, waitFor } from "test-utils";
import userEvent from "@testing-library/user-event";
import { villains } from "../../db.json";

describe("VillainList", () => {
  const handleDeleteVillain = jest.fn();

  it("no villains should not display a list nor search bar", async () => {
    wrappedRender(
      <VillainList villains={[]} handleDeleteVillain={handleDeleteVillain} />
    );

    expect(await screen.findByTestId("villain-list")).toBeInTheDocument();
    expect(screen.queryByTestId("villain-list-item-1")).not.toBeInTheDocument();
    expect(screen.queryByTestId("search-bar")).not.toBeInTheDocument();
  });

  describe("with villains in the list", () => {
    beforeEach(() => {
      wrappedRender(
        <VillainList
          villains={villains}
          handleDeleteVillain={handleDeleteVillain}
        />
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
          <Routes>
            <Route
              path=""
              element={
                <VillainList
                  villains={villains}
                  handleDeleteVillain={handleDeleteVillain}
                />
              }
            />
            <Route path="/add-villain" element={<VillainDetail />} />
            <Route path="/edit-villain/:id" element={<VillainDetail />} />
            <Route
              path="*"
              element={
                <VillainList
                  villains={villains}
                  handleDeleteVillain={handleDeleteVillain}
                />
              }
            />
          </Routes>
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

Add `/villains` route to `App.tsx`.

```tsx
// src/App.tsx
import { lazy, Suspense } from "react";
import { BrowserRouter, Routes, Route, Navigate } from "react-router-dom";
import { QueryClient, QueryClientProvider } from "react-query";
import { ErrorBoundary } from "react-error-boundary";
import HeaderBar from "components/HeaderBar";
import NavBar from "components/NavBar";
import PageSpinner from "components/PageSpinner";
import ErrorComp from "components/ErrorComp";
import Villains from "villains/Villains";
import "./styles.scss";
const Heroes = lazy(() => import("heroes/Heroes"));
const NotFound = lazy(() => import("components/NotFound"));
const About = lazy(() => import("About"));

const queryClient = new QueryClient();

function App() {
  return (
    <BrowserRouter>
      <HeaderBar />
      <div className="section columns">
        <NavBar />
        <main className="column">
          <QueryClientProvider client={queryClient}>
            <ErrorBoundary fallback={<ErrorComp />}>
              <Suspense fallback={<PageSpinner />}>
                <Routes>
                  <Route path="/" element={<Navigate replace to="/heroes" />} />
                  <Route path="/heroes/*" element={<Heroes />} />
                  <Route path="/villains/*" element={<Villains />} />
                  <Route path="/about" element={<About />} />
                  <Route path="*" element={<NotFound />} />
                </Routes>
              </Suspense>
            </ErrorBoundary>
          </QueryClientProvider>
        </main>
      </div>
    </BrowserRouter>
  );
}

export default App;
```

## Context API for villains

At the moment we have a full mirror of heroes to villains, functioning and being tested exactly the same way. We will however modify the villains group and take advantage of Context api while doing so.

`Villains.tsx` passes `villains` as a prop to `VillainList.tsx`. We will instead use the Context api so that `villains` is available in all components under `Villains.txt`.

Here are the general steps with Context api:

1. Create the context and export it. Usually this is in a separate file, acting as an arbiter.

   ```tsx
   // src/villains/VillainsContext.tsx (the common node)
   import { Villain } from "models/Villain";
   import { createContext } from "react";

   const VillainsContext = createContext<Villain[]>([]);
   export default VillainsContext;
   ```

2. Identify the state to be passed down to child components. Import the context there.

   ```tsx
   // src/villains/Villains.tsx
   import { VillainsContext } from "./VillainsContext";

   // ...

   const { villains, status, getError } = useGetEntity();
   ```

3. Wrap the UI with the context’s Provider component, assign the state to be passed down to the `value` prop:

   ```tsx
   // src/villains/Villains.tsx (the sharer)

   <VillainsContext.Provider value={villains}>
     <Routes>...</Routes>
   </VillainsContext.Provider>
   ```

4. In any component that is needing the state, consume Context API; import the `useContext` hook, and the context object:

   ```tsx
   // src/villains/VillainList.tsx
   import { useContext } from "react";
   import { VillainsContext } from "./VillainsContext";
   ```

5. Call useContext with the shared context, assign to a var:

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

      cy.get("footer").within(() => {
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

```ts
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

      cy.get("footer").within(() => {
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
