# CCTDD: Cypress Component Test Driven Design

We will recreate Angular's Tour of Heroes, in a component test driven manner, using Cypress, React and TS. John Papa has the same app in Angular, Vue and ReactJS, alas the React version is dated, and as the React community we know how frequently the meta changes. The approach gives readers coming from Vue and Angular to make 1:1 comparisons when writing Cypress component tests, and even a possibility to redo this exercise in one of those frameworks.

## Setup

Use this [template](https://github.com/muratkeremozcan/react-cypress-ts-template) to create a new repository. The repo will have React, TS, Cypress (e2e & ct), GHA with CI architecture, Jest, ESLint, Prettier, Renovate, Husky, Lint-staged, and most of the things you wish to have before getting started with a new project. Clone your new repo and install.

For the instructions we will assume that the repo you created is https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts, where you can find the final version of the app. The examples use `yarn` , and `npm` can be used if preferred.

```bash
git clone https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts
cd tour-of-heroes-react-cypress-ts
# if you are setup on a company proxy, append the --registry modifier
# otherwise just yarn install
yarn install --registry https://registry.yarnpkg.com
# note: from here on assume that --registry https://registry.yarnpkg.com
# will need to be appended if working on a company proxy
# else you will have to host the repo in the company's private GitHub domain
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

For the requirement spec, we will use the [Angular version of the app](https://papa-heroes-angular.azurewebsites.net/heroes). In the first half the book, we will use Cypress component testing with a test driven design approach to engineer our components. We will work on the child components first and work our way up. We will use `console.log` for the click event handlers until we have to make decisions about them.

In the later chapters when we tackle state and routing, Cypress e2e testing will become more relevant for TDD, and the component test suite will serve as the reassurance that our app still works as expected.

In every chapter we will explicitly capture Red Green Refactor cycles of TDD, and will wrap with a summary & key takeaways. Every code sample is meant to be copy pastable and to give reproducible results, however the reader is encouraged to write and incrementally improve the code rather than copy paste. One final disclaimer; in the early chapters, to avoid major rehash of files and code later, we might make some decisions towards being more aligned with the final version of the app.
