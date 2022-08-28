## [`json-server`](https://github.com/typicode/json-server)

We have been using a json file `src/heroes/heroes.json` in the `Heroes` component. Our app is not talking to a backend. It would be ideal to have a fake REST API instead, and `json-server` can enable that. Add the following packages to our app:

```bash
yarn add json-server json-server-reset
yarn add -D concurrently
```

Create a `dbj.json` file in the project root and copy the below content to it.

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
  ]
}
```

Add a script to `package.json` near to the `"start"` script. This will us the `db.json` file and respond with that content at `localhost:4000`, with a 1 second simulated network delay. [ `json-server-reset`](https://github.com/bahmutov/json-server-reset) is used so that every time the the start script is run, db.json is reset to its original state.

```json
{
  "scripts": {
    "start:api": "json-server --watch db.json --port 4000 --delay 1000 --middlewares ./node_modules/json-server-reset"
  }
}
```

Browsing to `http://localhost:4000/heroes` or http://localhost:4000/villains, we should see some content

![json-server](/Users/murat/cctdd/book/img/json-server.png)

Either by hard coding into `db.json`, or using any HTTP client, add a section to heroes. The below is using [VS code REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) to post to `json-server`.

```tex
@baseUrl = http://localhost:4000/heroes

POST {{baseUrl}}
Content-Type: application/json

{
  "id": "myNewHero",
  "name": "Herooo",
  "description": "commander of heroes"
}
```
