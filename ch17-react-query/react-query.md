## Caching and [`react-query`](https://react-query-v3.tanstack.com/overview)

We can drastically simplify our UI state management if we split out the server cache into something separate

State can be lumped into two buckets:

1. UI state: Modal is open, item is highlighted, etc. _(we used `useState` hook for this)_

2. Server cache: User data, tweets, contacts, etc. _(`react-query` is useful here)_

Why `react-query`? To prevent duplicated data-fetching, we want to move all the data-fetching code into a central store and access that single source from the components that need it. With React Query, we don’t need to do any of the work involved in creating such a store. It lets us keep the data-fetching code in the components that need the data, but behind the scenes it manages a data cache, passing already-fetched data to components when they ask for them.

React-query's `useQuery` hook is for fetching data (by key) and caching it, while updating cache. Think of it similar to our `GET` requests while using Cypress as the api client.

`const { data, status, error } = useQuery(key, () => fetch(url))`

The key arg is a unique identifier for the query / data in cache; string, array or object. The 2nd arg an async function that returns the data. `useQuery` can be called with a 3rd arg, a config object with initialData property

`useMutation` is the write mirror of `useQuery`. Think of it similar to our `PUT` and `POST` requests while using Cypress as the api client.

`useQuery` fetches state: UI state <- server/url , and caches it.

`useMutation` is just the opposite: UI state -> server , and still caches it.

`useMutation` yields data, status, error just like useQuery.

`const { dataToMutate, status, error } = useMutation((*url*) => fetch(*url*) {.. non-idempotent (*POST* *for* *example*) ..})`

The first arg is a function that that executes a non-idempotent request. The second arg is an object with onMutate property
