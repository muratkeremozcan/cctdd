# CCTDD: Cypress Component Test Driven Design

We will recreate Angular's Tour of Heroes, in a component test driven manner, using Cypress, React and Redux.

## Setup

Use this [template](https://github.com/muratkeremozcan/react-cypress-ts-template) to create a repository, it has React, TS, Cypress (e2e & ct), GHA with CI architecture, Jest, ESLint, Prettier, Renovate, Husky, Lint-staged, and most of the things you need to get started with a new project. Clone the repo and install.

For the instructions we will assume that the repo is https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts, and we are using yarn.

```bash
git clone https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts
cd tour-of-heroes-react-cypress-ts
# if you are setup on company proxy, append the --registry modifier
# otherwise just yarn install
yarn install --registry https://registry.yarnpkg.com
# note: from here on assume that --registry https://registry.yarnpkg.com
# will need to be appended if working on a company proxy
```

For a sanity check, we will execute the commands one at a time

```bash
# parallel unit, typecheck, lint, format, build
yarn validate

# Execute the e2e, ct & unit tests

# check the Cypress component test runner
yarn cy:open-ct

# check the Cypress e2e test runner (starts the app)
yarn cy:open-e2e

# check Jest (local execution is in watch mode)
yarn test
```

We will be following Angular's well known [Tour of Heroes tutorial](https://angular.io/tutorial), which serves us a requirement specification.
