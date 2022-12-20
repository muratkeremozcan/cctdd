# Update the current app to be more generic

Create new interfaces and types that will be utilized throughout the hero and villain groups of components.

```typescript
// src/models/Boy.ts
export interface Boy {
  id: string;
  name: string;
  description: string;
}

/* istanbul ignore file */
```

```typescript
// src/models/types.ts
import { Hero } from "./Hero";
import { Villain } from "./Villain";
import { Boy } from "./Boy";

export type HeroProperty = Hero["name"] | Hero["description"] | Hero["id"];
export type VillainProperty =
  | Villain["name"]
  | Villain["description"]
  | Villain["id"];

export type BoyProperty = Boy["name"] | Boy["description"] | Boy["id"];
export type EntityRoute = "heroes" | "villains" | "boys";
export type EntityType = "hero" | "villain" | "boy";

/** Returns the corresponding route for the entity;
 *
 * `hero` -> `/heroes`, `villain` -> `/villains`, `boy` -> `/boys` */
export const entityRoute = (entityType: EntityType) =>
  entityType === "hero"
    ? "heroes"
    : entityType === "villain"
    ? "villains"
    : "boys";

/* istanbul ignore file */
```

## Move `api.ts` to its own folder

From `./src/hooks/api.ts` to `./src/api/api.ts`. Your IDE should update the dependencies should automatically.

## Update the hooks

`useDeleteEntity` gets updated for `Boy`..

```typescript
// src/hooks/useDeleteEntity.ts
import { Boy } from "models/Boy";
import { Hero } from "models/Hero";
import { EntityType, entityRoute } from "models/types";
import { Villain } from "models/Villain";
import { useMutation, useQueryClient } from "react-query";
import { useNavigate } from "react-router-dom";
import { deleteItem } from "../api/api";

/**
 * Helper for DELETE to `/heroes`, `/villains` or 'boys' routes.
 * @returns {object} {deleteEntity, isDeleting, isDeleteError, deleteError}
 */
export function useDeleteEntity(entityType: EntityType) {
  const route = entityRoute(entityType);
  const navigate = useNavigate();
  const queryClient = useQueryClient();

  const mutation = useMutation(
    (item: Hero | Villain | Boy) => deleteItem(`${route}/${item.id}`),
    {
      // on success receives the original item as a second argument
      // if you recall, the first argument is the created item
      onSuccess: (_, deletedEntity: Hero | Villain | Boy) => {
        // get all the entities from the cache
        const entities: Hero[] | Villain[] | Boy[] =
          queryClient.getQueryData([`${route}`]) || [];
        // set the entities cache without the delete one
        queryClient.setQueryData(
          [`${route}`],
          entities.filter((h) => h.id !== deletedEntity.id)
        );

        navigate(`/${route}`);
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

`useGetEntities` just needs the comment to be updated

- _Helper for GET to `/heroes` or `/villains` routes_
- _Helper for GET to `/heroes` , `/villains` or `/boys` routes_.

`usePostEntity` gets updated for `Boy`.

```typescript
// src/hooks/usePostEntity.ts
import { Boy } from "models/Boy";
import { Hero } from "models/Hero";
import { EntityType, entityRoute } from "models/types";
import { Villain } from "models/Villain";
import { useMutation, useQueryClient } from "react-query";
import { useNavigate } from "react-router-dom";
import { createItem } from "../api/api";

/**
 * Helper for simple POST to `/heroes`, `/villains`, `/boys` routes
 * @returns {object} {mutate, status, error}
 */
export function usePostEntity(entityType: EntityType) {
  const route = entityRoute(entityType);
  const queryClient = useQueryClient();
  const navigate = useNavigate();
  return useMutation((item: Hero | Villain | Boy) => createItem(route, item), {
    onSuccess: (newData: Hero | Villain | Boy) => {
      //  use queryClient's setQueryData to set the cache
      // takes a key as the first arg, the 2nd arg is a cb that takes the old query cache and returns the new one
      queryClient.setQueryData(
        [route],
        (oldData: Hero[] | Villain[] | Boy[] | undefined) => [
          ...(oldData || []),
          newData,
        ]
      );

      return navigate(`/${route}`);
    },
  });
}
```

`usePutEntity` gets updated for `Boys`.

```typescript
// src/hooks/usePutEntity.ts
import { Hero } from "models/Hero";
import { Boy } from "models/Boy";
import { Villain } from "models/Villain";
import { EntityType, entityRoute } from "models/types";
import { useMutation, useQueryClient } from "react-query";
import type { QueryClient } from "react-query";
import { useNavigate } from "react-router-dom";
import { editItem } from "../api/api";

/**
 * Helper for PUT to `/heroes` route
 * @returns {object} {updateHero, isUpdating, isUpdateError, updateError}
 */
export function usePutEntity(entityType: EntityType) {
  const route = entityRoute(entityType);
  const queryClient = useQueryClient();
  const navigate = useNavigate();
  const mutation = useMutation(
    (item: Hero | Villain | Boy) => editItem(`${route}/${item.id}`, item),
    {
      onSuccess: (updatedEntity: Hero | Villain | Boy) => {
        updateEntityCache(entityType, updatedEntity, queryClient);
        navigate(`/${route}`);
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
  updatedEntity: Hero | Villain | Boy,
  queryClient: QueryClient
) {
  const route = entityRoute(entityType);
  // get all the heroes from the cache
  let entityCache: Hero[] | Villain[] | Boy[] =
    queryClient.getQueryData(route) || [];

  // find the index in the cache of the hero that's been edited
  const entityIndex = entityCache.findIndex((h) => h.id === updatedEntity.id);

  if (entityIndex !== -1) {
    // if the entity is found, replace the pre-edited entity with the updated one
    // this is just replacing an array item in place,
    // while not mutating the original array
    entityCache = entityCache.map((preEditedEntity) =>
      preEditedEntity.id === updatedEntity.id ? updatedEntity : preEditedEntity
    );
    console.log("entityCache is", entityCache);
    // use queryClient's setQueryData to set the cache
    // takes a key as the first arg, the 2nd arg is the new cache
    return queryClient.setQueryData([route], entityCache);
  }
}
```

## Modify the e2e commands

Between the 3 files under `./cypress/support`, we can compartmentalize the imports in favor of relevance

- `commands.ts`: all plugin imports, and commands common to e2e & CT (ex: `cy.getByCY`).
- `component.ts`: the commands specific to component tests (ex: `cy.mount`, `cy.wrappedMount`).
- `e2e.ts`: the commands specific to e2e (ex: `cy.crud`).

```typescript
// cypress/support/commands.ts
import "@testing-library/cypress/add-commands";
import "@bahmutov/cypress-code-coverage/support";

Cypress.Commands.add("getByCy", (selector, ...args) =>
  cy.get(`[data-cy="${selector}"]`, ...args)
);

Cypress.Commands.add("getByCyLike", (selector, ...args) =>
  cy.get(`[data-cy*=${selector}]`, ...args)
);

Cypress.Commands.add("getByClassLike", (selector, ...args) =>
  cy.get(`[class*=${selector}]`, ...args)
);
```

`component.tsx` and `e2e.ts` will import `commands.ts` so that CT and e2e files have access to common commands and plugins.

```tsx
// cypress/support/component.tsx
import "./commands";
import { mount } from "cypress/react18";
import { BrowserRouter } from "react-router-dom";
import { QueryClient, QueryClientProvider } from "react-query";
import { ErrorBoundary } from "react-error-boundary";
import ErrorComp from "../../src/components/ErrorComp";
import PageSpinner from "../../src/components/PageSpinner";
import { Suspense } from "react";

Cypress.Commands.add("mount", mount);

Cypress.Commands.add(
  "wrappedMount",
  (WrappedComponent: React.ReactNode, options = {}) => {
    const wrapped = (
      <QueryClientProvider client={new QueryClient()}>
        <ErrorBoundary fallback={<ErrorComp />}>
          <Suspense fallback={<PageSpinner />}>
            <BrowserRouter>{WrappedComponent}</BrowserRouter>
          </Suspense>
        </ErrorBoundary>
      </QueryClientProvider>
    );
    return cy.mount(wrapped, options);
  }
);
```

```typescript
// cypress/support/e2e.ts
import "./commands";
import { Villain } from "./../../src/models/Villain";
import { Hero } from "../../src/models/Hero";
import { Boy } from "../../src/models/Boy";
import {
  entityRoute,
  EntityRoute,
  EntityType,
  HeroProperty,
  VillainProperty,
  BoyProperty,
} from "../../src/models/types";
import data from "../fixtures/db.json";

Cypress.Commands.add(
  "crud",
  (
    method: "GET" | "POST" | "PUT" | "DELETE",
    route: string,
    {
      body,
      allowedToFail = false,
    }: { body?: Hero | Villain | Boy | object; allowedToFail?: boolean } = {}
  ) =>
    cy.request<(Hero[] & Hero) | (Villain[] & Villain) | (Boy[] & Boy)>({
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
  (property: HeroProperty | VillainProperty | BoyProperty) =>
  (entity: Hero | Villain | Boy) =>
    entity.name === property ||
    entity.description === property ||
    entity.id === property;

const getEntities = (entityRoute: EntityRoute) =>
  cy.crud("GET", entityRoute).its("body");

Cypress.Commands.add(
  "getEntityByProperty",
  (
    entityType: EntityType,
    property: HeroProperty | VillainProperty | BoyProperty
  ) =>
    getEntities(entityRoute(entityType)).then((entities) =>
      _.find(entities, propExists(property))
    )
);

Cypress.Commands.add(
  "findEntityIndex",
  (
    entityType: EntityType,
    property: HeroProperty | VillainProperty | BoyProperty
  ) =>
    getEntities(entityRoute(entityType)).then(
      (body: Hero[] | Villain[] | Boy[]) => ({
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

Update the type definitions in the repo root.

````typescript
// cypress.d.ts
/* eslint-disable @typescript-eslint/no-explicit-any */
import { MountOptions, MountReturn } from "cypress/react";
import { HeroProperty, VillainProperty, EntityType } from "models/types";
import type { Hero } from "./models/types/Hero";
import type { Villain } from "./models/types/Villain";
import type { Boy } from "./models/types/Boy";

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
      ): Cypress.Chainable<Hero | Villain | Boy>;

      /**
       * Given a hero property (name, description or id),
       * returns the index of the hero, and the entire collection, as an object.
       */
      findEntityIndex(
        entityType: EntityType,
        property: HeroProperty
      ): Cypress.Chainable<{
        entityIndex: number;
        entityArray: Hero[] | Villain[] | Boy[];
      }>;

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
      resetData(): Cypress.Chainable<
        Response<(Hero[] & Hero) | (Villain[] & Villain) | (Boy[] & Boy)>
      >;
    }
  }
}
````

## Create a boys mirror of the heroes

### Update `./db.json`

Add `boys` group.

```json
{
  "heroes": [
    {
      "id": "HeroAslaug",
      "name": "Aslaug",
      "description": "warrior queen"
    },
    {
      "id": "HeroBjorn",
      "name": "Bjorn Ironside",
      "description": "king of 9th century Sweden"
    },
    {
      "id": "HeroIvar",
      "name": "Ivar the Boneless",
      "description": "commander of the Great Heathen Army"
    },
    {
      "id": "HeroLagertha",
      "name": "Lagertha the Shieldmaiden",
      "description": "aka Hlaðgerðr"
    },
    {
      "id": "HeroRagnar",
      "name": "Ragnar Lothbrok",
      "description": "aka Ragnar Sigurdsson"
    },
    {
      "id": "HeroThora",
      "name": "Thora Town-hart",
      "description": "daughter of Earl Herrauðr of Götaland"
    }
  ],
  "villains": [
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
  ],
  "boys": [
    {
      "id": "BoyHomelander",
      "name": "Homelander",
      "description": "Like Superman, but a jerk."
    },
    {
      "id": "BoyAnnieJanuary",
      "name": "Annie January",
      "description": "The Defender of Des Moines."
    },
    {
      "id": "BoyBillyButcher",
      "name": "Billy Butcher",
      "description": "A former member of the British special forces turned vigilante."
    },
    {
      "id": "BoyBlackNoir",
      "name": "Black Noir",
      "description": "Master Martial Artist, expert hand-to-hand combatant highly trained in various forms of martial arts."
    }
  ]
}
```

Mirror it to `./cypress/fixtures/db.json` so that we can reset the db state correctly.

```json
{
  "heroes": [
    {
      "id": "HeroAslaug",
      "name": "Aslaug",
      "description": "warrior queen"
    },
    {
      "id": "HeroBjorn",
      "name": "Bjorn Ironside",
      "description": "king of 9th century Sweden"
    },
    {
      "id": "HeroIvar",
      "name": "Ivar the Boneless",
      "description": "commander of the Great Heathen Army"
    },
    {
      "id": "HeroLagertha",
      "name": "Lagertha the Shieldmaiden",
      "description": "aka Hlaðgerðr"
    },
    {
      "id": "HeroRagnar",
      "name": "Ragnar Lothbrok",
      "description": "aka Ragnar Sigurdsson"
    },
    {
      "id": "HeroThora",
      "name": "Thora Town-hart",
      "description": "daughter of Earl Herrauðr of Götaland"
    }
  ],
  "villains": [
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
  ],
  "boys": [
    {
      "id": "BoyHomelander",
      "name": "Homelander",
      "description": "Like Superman, but a jerk."
    },
    {
      "id": "BoyAnnieJanuary",
      "name": "Annie January",
      "description": "The Defender of Des Moines."
    },
    {
      "id": "BoyBillyButcher",
      "name": "Billy Butcher",
      "description": "A former member of the British special forces turned vigilante."
    },
    {
      "id": "BoyBlackNoir",
      "name": "Black Noir",
      "description": "Master Martial Artist, expert hand-to-hand combatant highly trained in various forms of martial arts."
    }
  ]
}
```

### Add fixture `./cypress/fixtures/boys.json`

```json
[
  {
    "id": "BoyHomelander",
    "name": "Homelander",
    "description": "Like Superman, but a jerk."
  },
  {
    "id": "BoyAnnieJanuary",
    "name": "Annie January",
    "description": "The Defender of Des Moines."
  },
  {
    "id": "BoyBillyButcher",
    "name": "Billy Butcher",
    "description": "A former member of the British special forces turned vigilante."
  },
  {
    "id": "BoyBlackNoir",
    "name": "Black Noir",
    "description": "Master Martial Artist, expert hand-to-hand combatant highly trained in various forms of martial arts."
  }
]
```

### Mirror the components

We are creating 3 components for boys, mirroring heroes group as they are. We also have to update some of the base components.

For `ListHeader` only the type changes for `title`.

```tsx
// src/components/ListHeader.tsx
import { MouseEvent } from "react";
import { NavLink } from "react-router-dom";
import { FiRefreshCcw } from "react-icons/fi";
import { GrAdd } from "react-icons/gr";

type ListHeaderProps = {
  title: "Heroes" | "Villains" | "Boys" | "About";
  handleAdd: (e: MouseEvent<HTMLButtonElement>) => void;
  handleRefresh: (e: MouseEvent<HTMLButtonElement>) => void;
};

export default function ListHeader({
  title,
  handleAdd,
  handleRefresh,
}: ListHeaderProps) {
  return (
    <div data-cy="list-header" className="content-title-group">
      <NavLink data-cy="title" to={title}>
        <h2>{title}</h2>
      </NavLink>
      <button data-cy="add-button" onClick={handleAdd} aria-label="add">
        <GrAdd />
      </button>
      <button
        data-cy="refresh-button"
        onClick={handleRefresh}
        aria-label="refresh"
      >
        <FiRefreshCcw />
      </button>
    </div>
  );
}
```

The rest are mirrors. Create a folder `./src/boys/` and the 3 mirror files under it.

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
  const handleCancel = () => navigate("/boys");
  const handleSave = () =>
    name ? updateBoy(boy as Boy) : createBoy(boy as Boy);
  const handleNameChange = (e: ChangeEvent<HTMLInputElement>) => {
    setBoy({ ...boy, name: e.target.value });
  };
  const handleDescriptionChange = (e: ChangeEvent<HTMLInputElement>) => {
    setBoy({ ...boy, description: e.target.value });
  };

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

  // currying: the outer fn takes our custom arg and returns a fn that takes the event
  const handleSelectBoy = (boyId: string) => () => {
    const boy = deferredBoys.find((b: Boy) => b.id === boyId);
    navigate(
      `/boys/edit-boy/${boy?.id}?name=${boy?.name}&description=${boy?.description}`
    );
  };

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

  /** filters the boys data to see if the any of the properties exist in the list */
  const handleSearch =
    (data: Boy[]) => (event: ChangeEvent<HTMLInputElement>) => {
      const searchField = event.target.value;

      return startTransition(() =>
        setFilteredBoys(searchProperties(searchField, data))
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

export default function Boys() {
  const [showModal, setShowModal] = useState<boolean>(false);
  const { entities: boys, getError } = useGetEntities("boys");
  const [boyToDelete, setBoyToDelete] = useState<Boy | null>(null);
  const { deleteEntity: deleteBoy, isDeleteError } = useDeleteEntity("boy");

  const navigate = useNavigate();
  const addNewBoy = () => navigate("/boys/add-boy");
  const handleRefresh = () => navigate("/boys");

  const handleCloseModal = () => {
    setBoyToDelete(null);
    setShowModal(false);
  };
  // currying: the outer fn takes our custom arg and returns a fn that takes the event
  const handleDeleteBoy = (boy: Boy) => () => {
    setBoyToDelete(boy);
    setShowModal(true);
  };
  const handleDeleteFromModal = () => {
    boyToDelete ? deleteBoy(boyToDelete) : null;
    setShowModal(false);
  };

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

Add Boys route to `App.tsx`

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
import Boys from "boys/Boys";
import "./styles.scss";
const Heroes = lazy(() => import("heroes/Heroes"));
const NotFound = lazy(() => import("components/NotFound"));
const About = lazy(() => import("About"));

const queryClient = new QueryClient();

export default function App() {
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
                  <Route path="/boys/*" element={<Boys />} />
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
```

Add `Boys` to `NavBar` component.

```tsx
// src/components/NavBar.tsx
import { NavLink } from "react-router-dom";

export default function NavBar() {
  const linkIsActive = (link: { isActive: boolean }) =>
    link.isActive ? "active-link" : "";

  return (
    <nav data-cy="nav-bar" className="column is-2 menu">
      <p className="menu-label">Menu</p>
      <ul data-cy="menu-list" className="menu-list">
        <NavLink to="/heroes" className={linkIsActive}>
          Heroes
        </NavLink>
        <NavLink to="/villains" className={linkIsActive}>
          Villains
        </NavLink>
        <NavLink to="/boys" className={linkIsActive}>
          Boys
        </NavLink>
        <NavLink to="/about" className={linkIsActive}>
          About
        </NavLink>
      </ul>
    </nav>
  );
}
```

### Mirror the e2e tests

We create 3 new e2e tests, which are boy mirrors of the hero versions.

```typescript
// cypress/e2e/create-boy.cy.ts
import { faker } from "@faker-js/faker";
describe("Create boy", () => {
  before(cy.resetData);

  const navToAddBoy = () => {
    cy.location("pathname").should("eq", "/boys");
    cy.getByCy("add-button").click();
    cy.location("pathname").should("eq", "/boys/add-boy");
    cy.getByCy("boy-detail").should("be.visible");
    cy.getByCy("input-detail-id").should("not.exist");
  };

  it("should go through the refresh flow (ui-integration)", () => {
    cy.visitStubbedEntities("boys");
    navToAddBoy();

    cy.getByCy("refresh-button").click();
    cy.location("pathname").should("eq", "/boys");
    cy.getByCy("boy-list").should("be.visible");
  });

  it("should go through the cancel flow and perform direct navigation (ui-integration)", () => {
    cy.intercept("GET", `${Cypress.env("API_URL")}/boys`, {
      fixture: "boys",
    }).as("stubbedGetBoys");
    cy.visit("/boys/add-boy");
    cy.wait("@stubbedGetBoys");

    cy.getByCy("cancel-button").click();
    cy.location("pathname").should("eq", "/boys");
    cy.getByCy("boy-list").should("be.visible");
  });

  it("should go through the add boy flow (ui-e2e)", () => {
    cy.visitEntities("boys");
    navToAddBoy();

    const newBoy = {
      name: faker.internet.userName(),
      description: `description ${faker.internet.userName()}`,
    };
    cy.getByCy("input-detail-name").type(newBoy.name);
    cy.getByCy("input-detail-description").type(newBoy.description);
    cy.getByCy("save-button").click();

    cy.location("pathname").should("eq", "/boys");

    cy.getByCy("boys").should("be.visible");
    cy.getByCyLike("boy-list-item").should("have.length.gt", 0);
    cy.getByCy("boy-list")
      .should("contain", newBoy.name)
      .and("contain", newBoy.description);

    cy.getEntityByProperty("boy", newBoy.name).then((myBoy) =>
      cy.crud("DELETE", `boys/${myBoy.id}`)
    );
  });
});
```

```typescript
// cypress/e2e/delete-boy.cy.ts
import { faker } from "@faker-js/faker";
import { Boy } from "../../src/models/Boy";
describe("Delete boy", () => {
  before(cy.resetData);

  const yesOnModal = () =>
    cy.getByCy("modal-yes-no").within(() => cy.getByCy("button-yes").click());

  it("should go through the cancel flow (ui-integration)", () => {
    cy.visitStubbedEntities("boys");

    cy.getByCy("delete-button").first().click();
    cy.getByCy("modal-yes-no").within(() => cy.getByCy("button-no").click());
    cy.getByCy("boys").should("be.visible");
    cy.get("modal-yes-no").should("not.exist");
  });

  it("should go through the edit flow (ui-e2e)", () => {
    const boy: Boy = {
      id: faker.datatype.uuid(),
      name: faker.internet.userName(),
      description: `description ${faker.internet.userName()}`,
    };

    cy.crud("POST", "boys", { body: boy });

    cy.visitEntities("boys");

    cy.findEntityIndex("boy", boy.id).then(
      ({ entityIndex: boyIndex, entityArray: boyArray }) => {
        cy.getByCy("delete-button").eq(boyIndex).click();

        yesOnModal();

        cy.getByCy("boy-list")
          .should("be.visible")
          .should("not.contain", boyArray[boyIndex].name)
          .and("not.contain", boyArray[boyIndex].description);
      }
    );
  });
});
```

```typescript
// cypress/e2e/edit-boy.cy.ts
import { faker } from "@faker-js/faker";
import { Boy } from "../../src/models/Boy";
describe("Edit boy", () => {
  before(cy.resetData);

  /** Verifies boy info on Edit page */
  const verifyBoy = (boys: Boy[], heroIndex: number) => {
    cy.location("pathname").should("include", "/boys/edit-boy/");
    cy.getByCy("boy-detail").should("be.visible");
    cy.getByCy("input-detail-id").should("be.visible");
    cy.findByDisplayValue(boys[heroIndex].id);
    cy.findByDisplayValue(boys[heroIndex].name);
    cy.findByDisplayValue(boys[heroIndex].description);
  };

  const randomBoyIndex = (boys: Boy[]) => Cypress._.random(0, boys.length - 1);

  it("should go through the cancel flow for a random boy (ui-integration)", () => {
    cy.visitStubbedEntities("boys");

    cy.fixture("boys").then((boys) => {
      const heroIndex = randomBoyIndex(boys);
      cy.getByCy("edit-button").eq(heroIndex).click();
      verifyBoy(boys, heroIndex);
    });

    cy.getByCy("cancel-button").click();
    cy.location("pathname").should("eq", "/boys");
    cy.getByCy("boy-list").should("be.visible");
  });

  it("should go through the PUT error flow (ui-integration)", () => {
    cy.visitStubbedEntities("boys");

    cy.fixture("boys").then((boys) => {
      const heroIndex = randomBoyIndex(boys);
      cy.getByCy("edit-button").eq(heroIndex).click();
      verifyBoy(boys, heroIndex);
    });

    cy.intercept("PUT", `${Cypress.env("API_URL")}/boys/*`, {
      statusCode: 500,
      delay: 100,
    }).as("isUpdateError");

    cy.getByCy("save-button").click();
    cy.getByCy("spinner");
    cy.wait("@isUpdateError");
    cy.getByCy("error");
  });

  it("should navigate to add from an existing boy (ui-integration)", () => {
    cy.visitStubbedEntities("boys");

    cy.fixture("boys").then((boys) => {
      const heroIndex = randomBoyIndex(boys);
      cy.getByCy("edit-button").eq(heroIndex).click();
      verifyBoy(boys, heroIndex);

      cy.getByCy("add-button").click();
      cy.getByCy("input-detail-id").should("not.exist");
      cy.findByDisplayValue(boys[heroIndex].name).should("not.exist");
      cy.findByDisplayValue(boys[heroIndex].description).should("not.exist");
    });
  });

  it("should go through the edit flow (ui-e2e)", () => {
    const newBoy: Boy = {
      id: faker.datatype.uuid(),
      name: faker.internet.userName(),
      description: `description ${faker.internet.userName()}`,
    };

    cy.crud("POST", "boys", { body: newBoy });

    cy.visit(`boys/edit-boy/${newBoy.id}`, {
      qs: { name: newBoy.name, description: newBoy.description },
    });

    const editedBoy = {
      name: faker.internet.userName(),
      description: `description ${faker.internet.userName()}`,
    };

    cy.getByCy("input-detail-name")
      .find(".input")
      .clear()
      .type(`${editedBoy.name}`);
    cy.getByCy("input-detail-description")
      .find(".input")
      .clear()
      .type(`${editedBoy.description}`);
    cy.getByCy("save-button").click();

    cy.getByCy("boy-list")
      .should("be.visible")
      .should("contain", editedBoy.name)
      .and("contain", editedBoy.description);

    cy.getEntityByProperty("boy", newBoy.id).then((myBoy: Boy) =>
      cy.crud("DELETE", `boys/${myBoy.id}`)
    );
  });
});
```

The backend test needs a new block to cover villains.

```typescript
// cypress/e2e/backend/crud.cy.ts
import { faker } from "@faker-js/faker";
import { Hero } from "../../../src/models/Hero";
import { Villain } from "../../../src/models/Villain";
import { Boy } from "../../../src/models/Boy";

describe("Backend e2e", () => {
  const assertProperties = (entity: Hero | Villain | Boy) => {
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

  it("should CRUD a new boy entity", () => {
    const newVillain = {
      id: faker.datatype.uuid(),
      name: faker.internet.userName(),
      description: `description ${faker.internet.userName()}`,
    };

    cy.crud("POST", "boys", { body: newVillain })
      .its("status")
      .should("eq", 201);

    cy.crud("GET", "boys")
      .its("body")
      .then((body) => {
        expect(body.at(-1)).to.deep.eq(newVillain);
      });

    const editedVillain = { ...newVillain, name: "Murat" };
    cy.crud("PUT", `boys/${editedVillain.id}`, { body: editedVillain })
      .its("status")
      .should("eq", 200);
    cy.crud("GET", `boys/${editedVillain.id}`)
      .its("body")
      .should("deep.eq", editedVillain);

    cy.crud("DELETE", `boys/${editedVillain.id}`)
      .its("status")
      .should("eq", 200);
    cy.crud("GET", `boys/${editedVillain.id}`, { allowedToFail: true })
      .its("status")
      .should("eq", 404);
  });
});
```

The routes-nav needs a new test to cover villains route.

```typescript
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

    cy.location("pathname").should("eq", "/heroes");
    cy.getByCy("heroes").should("be.visible");
  });

  it("should direct-navigate to /heroes", () => {
    const route = "/heroes";
    cy.visit(route);
    cy.location("pathname").should("eq", route);
    cy.getByCy("heroes").should("be.visible");
  });

  it("should direct-navigate to /villains", () => {
    const route = "/villains";
    cy.visit(route);
    cy.location("pathname").should("eq", route);
    cy.getByCy("villains").should("be.visible");
  });

  it("should land on not found when visiting an non-existing route", () => {
    const route = "/route48";
    cy.visit(route);
    cy.location("pathname").should("eq", route);
    cy.getByCy("not-found").should("be.visible");
  });

  it("should direct-navigate to about", () => {
    const route = "/about";
    cy.visit(route);
    cy.location("pathname").should("eq", route);
    cy.getByCy("about").contains("CCTDD");
  });

  it("should cover route history with browser back and forward", () => {
    cy.visit("/about");
    const routes = ["villains", "heroes", "about"];
    cy.wrap(routes).each((route: string) =>
      cy.get(`[href="/${route}"]`).click()
    );

    const lastIndex = routes.length - 1;
    cy.location("pathname").should("include", routes[lastIndex]);
    cy.go("back");
    cy.location("pathname").should("include", routes[lastIndex - 1]);
    cy.go("back");
    cy.location("pathname").should("include", routes[lastIndex - 2]);
    cy.go("forward").go("forward");
    cy.location("pathname").should("include", routes[lastIndex]);
  });
});
```

### Mirror the Cypress component tests

First, update `Navbar` and `App` tests. For `Navbar.cy` we just need to add `boys` to the `routes` array

```tsx
// src/components/NavBar.cy.tsx
import NavBar from "./NavBar";
import { BrowserRouter } from "react-router-dom";
import "../styles.scss";

describe("NavBar", () => {
  it("should navigate to the correct routes", () => {
    cy.mount(
      <BrowserRouter>
        <NavBar />
      </BrowserRouter>
    );

    cy.contains("p", "Menu");
    cy.getByCy("menu-list").children().should("have.length", routes.length);

    routes.forEach((route: string) => {
      cy.get(`[href="/${route}"]`)
        .contains(route, { matchCase: false })
        .click()
        .should("have.class", "active-link")
        .siblings()
        .should("not.have.class", "active-link");

      cy.url().should("contain", route);
    });
  });
});
```

For `App.cy` we need to intercept the `boys` route and add a check for `boys`.

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

    cy.intercept("GET", `${Cypress.env("API_URL")}/boys`, {
      fixture: "boys.json",
    });

    cy.mount(<App />);
    cy.getByCy("not-found").should("be.visible");

    cy.contains("Heroes").click();
    cy.getByCy("heroes").should("be.visible");

    cy.contains("Villains").click();
    cy.getByCy("villains").should("be.visible");

    cy.contains("Boys").click();
    cy.getByCy("boys").should("be.visible");

    cy.contains("About").click();
    cy.getByCy("about").should("be.visible");
  });
});
```

We need 3 new component tests for the 3 components under `boys`.

```tsx
// src/boys/BoyDetail.cy.tsx
import BoyDetail from "./BoyDetail";
import "../styles.scss";
import React from "react";
import * as postHook from "hooks/usePostEntity";

describe("BoyDetail", () => {
  beforeEach(() => {
    cy.wrappedMount(<BoyDetail />);
  });

  it("should handle Save", () => {
    // example of testing implementation details
    cy.spy(React, "useState").as("useState");
    cy.spy(postHook, "usePostEntity").as("usePostEntity");
    // instead prefer to test at a higher level
    cy.intercept("POST", "*", { statusCode: 200 }).as("postBoy");
    cy.getByCy("save-button").click();

    // test at a higher level
    cy.wait("@postBoy");
    // test implementation details (what not to do)
    cy.get("@useState").should("have.been.called");
    cy.get("@usePostEntity").should("have.been.called");
  });

  it("should handle non-200 Save", () => {
    cy.intercept("POST", "*", { statusCode: 400, delay: 100 }).as("postBoy");
    cy.getByCy("save-button").click();
    cy.getByCy("spinner");
    cy.wait("@postBoy");
    cy.getByCy("error");
  });

  it("should handle Cancel", () => {
    cy.getByCy("cancel-button").click();
    cy.location("pathname").should("eq", "/boys");
  });

  it("should handle name change", () => {
    const newBoyName = "abc";
    cy.getByCy("input-detail-name").type(newBoyName);

    cy.findByDisplayValue(newBoyName).should("be.visible");
  });

  it("should handle description change", () => {
    const newBoyDescription = "123";
    cy.getByCy("input-detail-description").type(newBoyDescription);

    cy.findByDisplayValue(newBoyDescription).should("be.visible");
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
// src/boys/BoyList.cy.tsx
import BoyList from "./BoyList";
import "../styles.scss";
import boys from "../../cypress/fixtures/boys.json";

describe("BoyList", () => {
  it("no boys should not display a list nor search bar", () => {
    cy.wrappedMount(
      <BoyList boys={[]} handleDeleteBoy={cy.stub().as("handleDeleteBoy")} />
    );

    cy.getByCy("boy-list").should("exist");
    cy.getByCyLike("boy-list-item").should("not.exist");
    cy.getByCy("search").should("not.exist");
  });

  context("with boys in the list", () => {
    beforeEach(() => {
      cy.wrappedMount(
        <BoyList
          boys={boys}
          handleDeleteBoy={cy.stub().as("handleDeleteBoy")}
        />
      );
    });

    it("should render the boy layout", () => {
      cy.getByCyLike("boy-list-item").should("have.length", boys.length);

      cy.getByCy("card-content");
      cy.contains(boys[0].name);
      cy.contains(boys[0].description);

      cy.get("footer")
        .first()
        .within(() => {
          cy.getByCy("delete-button");
          cy.getByCy("edit-button");
        });
    });

    it("should search and filter boy by name and description", () => {
      cy.getByCy("search").type(boys[0].name);
      cy.getByCyLike("boy-list-item")
        .should("have.length", 1)
        .contains(boys[0].name);

      cy.getByCy("search").clear().type(boys[2].description);
      cy.getByCyLike("boy-list-item")
        .should("have.length", 1)
        .contains(boys[2].description);
    });

    it("should handle delete", () => {
      cy.getByCy("delete-button").first().click();
      cy.get("@handleDeleteBoy").should("have.been.called");
    });

    it("should handle edit", () => {
      cy.getByCy("edit-button").first().click();
      cy.location("pathname").should("eq", "/boys/edit-boy/" + boys[0].id);
    });
  });
});
```

```tsx
// src/boys/Boys.cy.tsx
import Boys from "./Boys";
import "../styles.scss";

describe("Boys", () => {
  it("should see error on initial load with GET", () => {
    Cypress.on("uncaught:exception", () => false);
    cy.clock();
    cy.intercept("GET", `${Cypress.env("API_URL")}/boys`, {
      statusCode: 400,
      delay: 100,
    }).as("notFound");

    cy.wrappedMount(<Boys />);

    cy.getByCy("page-spinner").should("be.visible");
    Cypress._.times(3, () => {
      cy.tick(5000);
      cy.wait("@notFound");
    });
    cy.tick(5000);

    cy.getByCy("error");
  });

  context("200 flows", () => {
    beforeEach(() => {
      cy.intercept("GET", `${Cypress.env("API_URL")}/boys`, {
        fixture: "boys.json",
      }).as("getBoys");

      cy.wrappedMount(<Boys />);
    });

    it("should display the boy list on render, and go through boy add & refresh flow", () => {
      cy.wait("@getBoys");

      cy.getByCy("list-header").should("be.visible");
      cy.getByCy("boy-list").should("be.visible");

      cy.getByCy("add-button").click();
      cy.location("pathname").should("eq", "/boys/add-boy");

      cy.getByCy("refresh-button").click();
      cy.location("pathname").should("eq", "/boys");
    });

    const invokeBoyDelete = () => {
      cy.getByCy("delete-button").first().click();
      cy.getByCy("modal-yes-no").should("be.visible");
    };
    it("should go through the modal flow, and cover error on DELETE", () => {
      cy.getByCy("modal-yes-no").should("not.exist");

      cy.log("do not delete flow");
      invokeBoyDelete();
      cy.getByCy("button-no").click();
      cy.getByCy("modal-yes-no").should("not.exist");

      cy.log("delete flow");
      invokeBoyDelete();
      cy.intercept("DELETE", "*", { statusCode: 500 }).as("deleteBoy");

      cy.getByCy("button-yes").click();
      cy.wait("@deleteBoy");
      cy.getByCy("modal-yes-no").should("not.exist");
      cy.getByCy("error").should("be.visible");
    });
  });
});
```

### Mirror the RTL tests

We just need to replicate what was done with Cy CT to RTL.

```tsx
// src/boys/BoyDetail.test.tsx
import BoyDetail from "./BoyDetail";
import "@testing-library/jest-dom";
import { wrappedRender, act, screen, waitFor } from "test-utils";
import userEvent from "@testing-library/user-event";

describe("BoyDetail", () => {
  beforeEach(() => {
    wrappedRender(<BoyDetail />);
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

    expect(window.location.pathname).toBe("/boys");
  });

  it("should handle name change", async () => {
    const newBoyName = "abc";
    const inputDetailName = await screen.findByPlaceholderText("e.g. Colleen");
    userEvent.type(inputDetailName, newBoyName);

    await waitFor(async () =>
      expect(inputDetailName).toHaveDisplayValue(newBoyName)
    );
  });

  const inputDetailDescription = async () =>
    screen.findByPlaceholderText("e.g. dance fight!");

  it("should handle description change", async () => {
    const newBoyDescription = "123";

    userEvent.type(await inputDetailDescription(), newBoyDescription);
    await waitFor(async () =>
      expect(await inputDetailDescription()).toHaveDisplayValue(
        newBoyDescription
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
// src/boys/BoyList.test.tsx
import BoyList from "./BoyList";
import { wrappedRender, screen, waitFor } from "test-utils";
import userEvent from "@testing-library/user-event";
import { boys } from "../../db.json";

describe("BoyList", () => {
  const handleDeleteBoy = jest.fn();

  it("no boys should not display a list nor search bar", async () => {
    wrappedRender(<BoyList boys={[]} handleDeleteBoy={handleDeleteBoy} />);

    expect(await screen.findByTestId("boy-list")).toBeInTheDocument();
    expect(screen.queryByTestId("boy-list-item-1")).not.toBeInTheDocument();
    expect(screen.queryByTestId("search-bar")).not.toBeInTheDocument();
  });

  describe("with boys in the list", () => {
    beforeEach(() => {
      wrappedRender(<BoyList boys={boys} handleDeleteBoy={handleDeleteBoy} />);
    });

    const cardContents = async () => screen.findAllByTestId("card-content");
    const deleteButtons = async () => screen.findAllByTestId("delete-button");
    const editButtons = async () => screen.findAllByTestId("edit-button");

    it("should render the boy layout", async () => {
      expect(
        await screen.findByTestId(`boy-list-item-${boys.length - 1}`)
      ).toBeInTheDocument();

      expect(await screen.findByText(boys[0].name)).toBeInTheDocument();
      expect(await screen.findByText(boys[0].description)).toBeInTheDocument();
      expect(await cardContents()).toHaveLength(boys.length);
      expect(await deleteButtons()).toHaveLength(boys.length);
      expect(await editButtons()).toHaveLength(boys.length);
    });

    it("should search and filter boy by name and description", async () => {
      const search = await screen.findByTestId("search");

      userEvent.type(search, boys[0].name);
      await waitFor(async () => expect(await cardContents()).toHaveLength(1));
      await screen.findByText(boys[0].name);

      userEvent.clear(search);
      await waitFor(async () =>
        expect(await cardContents()).toHaveLength(boys.length)
      );

      userEvent.type(search, boys[2].description);
      await waitFor(async () => expect(await cardContents()).toHaveLength(1));
    });

    it("should handle delete", async () => {
      userEvent.click((await deleteButtons())[0]);
      expect(handleDeleteBoy).toHaveBeenCalled();
    });

    it("should handle edit", async () => {
      userEvent.click((await editButtons())[0]);
      await waitFor(() =>
        expect(window.location.pathname).toEqual("/boys/edit-boy/" + boys[0].id)
      );
    });
  });
});
```

```tsx
// src/boys/Boys.test.tsx
import Boys from "./Boys";
import { wrappedRender, screen, waitForElementToBeRemoved } from "test-utils";
import userEvent from "@testing-library/user-event";
import { rest } from "msw";
import { setupServer } from "msw/node";
import { boys } from "../../db.json";

describe("Boys", () => {
  // mute the expected console.error message, because we are mocking non-200 responses
  // eslint-disable-next-line @typescript-eslint/no-empty-function
  jest.spyOn(console, "error").mockImplementation(() => {});

  beforeEach(() => wrappedRender(<Boys />));

  it("should see error on initial load with GET", async () => {
    const handlers = [
      rest.get(
        `${process.env.REACT_APP_API_URL}/boys`,
        async (_req, res, ctx) => res(ctx.status(400))
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
        `${process.env.REACT_APP_API_URL}/boys`,
        async (_req, res, ctx) => res(ctx.status(200), ctx.json(boys))
      ),
      rest.delete(
        `${process.env.REACT_APP_API_URL}/boys/${boys[0].id}`, // use /.*/ for all requests
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

    it("should display the boy list on render, and go through boy add & refresh flow", async () => {
      expect(await screen.findByTestId("list-header")).toBeVisible();
      expect(await screen.findByTestId("boy-list")).toBeVisible();

      await userEvent.click(await screen.findByTestId("add-button"));
      expect(window.location.pathname).toBe("/boys/add-boy");

      await userEvent.click(await screen.findByTestId("refresh-button"));
      expect(window.location.pathname).toBe("/boys");
    });

    const deleteButtons = async () => screen.findAllByTestId("delete-button");
    const modalYesNo = async () => screen.findByTestId("modal-yes-no");
    const maybeModalYesNo = () => screen.queryByTestId("modal-yes-no");
    const invokeBoyDelete = async () => {
      userEvent.click((await deleteButtons())[0]);
      expect(await modalYesNo()).toBeVisible();
    };

    it("should go through the modal flow, and cover error on DELETE", async () => {
      expect(screen.queryByTestId("modal-dialog")).not.toBeInTheDocument();

      await invokeBoyDelete();
      await userEvent.click(await screen.findByTestId("button-no"));
      expect(maybeModalYesNo()).not.toBeInTheDocument();

      await invokeBoyDelete();
      await userEvent.click(await screen.findByTestId("button-yes"));

      expect(maybeModalYesNo()).not.toBeInTheDocument();
      expect(await screen.findByTestId("error")).toBeVisible();
      expect(screen.queryByTestId("modal-dialog")).not.toBeInTheDocument();
    });
  });
});
```

`App.test` needs a `msw` handler and a new route check.

```tsx
// src/App.test.tsx
iimport {act, render, screen} from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import App from './App'
import {heroes, villains, boys} from '../db.json'

import {rest} from 'msw'
import {setupServer} from 'msw/node'

describe('200 flow', () => {
  const handlers = [
    rest.get(
      `${process.env.REACT_APP_API_URL}/heroes`,
      async (_req, res, ctx) => res(ctx.status(200), ctx.json(heroes)),
    ),
    rest.get(
      `${process.env.REACT_APP_API_URL}/villains`,
      async (_req, res, ctx) => res(ctx.status(200), ctx.json(villains)),
    ),
    rest.get(`${process.env.REACT_APP_API_URL}/boys`, async (_req, res, ctx) =>
      res(ctx.status(200), ctx.json(boys)),
    ),
  ]
  const server = setupServer(...handlers)
  beforeAll(() => {
    server.listen({
      onUnhandledRequest: 'warn',
    })
  })
  afterEach(server.resetHandlers)
  afterAll(server.close)

  test('renders tour of heroes', async () => {
    render(<App />)
    await act(() => new Promise(r => setTimeout(r, 0))) // spinner

    await userEvent.click(screen.getByText('About'))
    expect(await screen.findByTestId('about')).toBeVisible()

    await userEvent.click(screen.getByText('Heroes'))
    expect(await screen.findByTestId('heroes')).toBeVisible()

    await userEvent.click(screen.getByText('Villains'))
    expect(await screen.findByTestId('villains')).toBeVisible()

    await userEvent.click(screen.getByText('Boys'))
    expect(await screen.findByTestId('boys')).toBeVisible()
  })
})
```

`Navbar.test` needs the same `routes` variable update to include `Boys`.

```tsx
// src/components/NavBar.test.tsx
import NavBar from "./NavBar";
import { render, screen, within, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { BrowserRouter } from "react-router-dom";
import "@testing-library/jest-dom";

const routes = ["Heroes", "Villains", "Boys", "About"];

describe("NavBar", () => {
  beforeEach(() => {
    render(
      <BrowserRouter>
        <NavBar />
      </BrowserRouter>
    );
  });

  it("should verify route layout", async () => {
    expect(await screen.findByText("Menu")).toBeVisible();

    const menuList = await screen.findByTestId("menu-list");
    expect(within(menuList).queryAllByRole("link").length).toBe(routes.length);

    routes.forEach((route) => within(menuList).getByText(route));
  });

  it.each(routes)("should navigate to route %s", async (route: string) => {
    const link = async (name: string) => screen.findByRole("link", { name });
    const activeRouteLink = await link(route);
    userEvent.click(activeRouteLink);
    await waitFor(() => expect(activeRouteLink).toHaveClass("active-link"));
    expect(window.location.pathname).toEqual(`/${route.toLowerCase()}`);

    const remainingRoutes = routes.filter((r) => r !== route);
    remainingRoutes.forEach(async (inActiveRoute) => {
      expect(await link(inActiveRoute)).not.toHaveClass("active-link");
    });
  });
});
```
