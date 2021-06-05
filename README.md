# Typescript Example Project

This is an API for the v4 autograder. The main purpose of this project is to act as an in-between for the containers doing the autograding, and the users submitting assignments.

## Project Dependencies (install these)

- [Node](https://nodejs.org/en/) - A javascript interpreter to run JS outside of the browser
- [Docker](https://www.docker.com/get-started) (this is for the DB) - Containerization

For now the only reason we're including docker is to more easily control the development database. In the future we may very well dockerize the api itself for ease of development and deployment, but for now this writeup is expecting it to be running directly on your machine via node.

## Running the project locally

### Getting Everything Started

Once you've got these installed, we can build our container and run it

```
docker run \
  --name typeorm-postgres \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=typescript_api \
  -e POSTGRES_USER=typescript_user \
  -d postgres
```

Install all node dependancies. All of the database environment variables can change, and can be set as environment variables on your machine if you want to overwrite the defaults

```
npm install
```

Run the initial migrations to setup our DB schema

```
npm run typeorm -- migration:run
```

Once you've got all the dependencies installed you can run the project via

```
npm start
```

By defualt the project runs at `localhost:3001`, but you can change the port by setting an alternate port by setting the `PORT` environment variable.

If you're working in vscode, a configuration has been included to allow for debugging. Open up the VS Code Run and Debug section and click `Debug API`.

### Convenient Devtools

The project has some nice devtools built into it that will make your life easier. Leaving these here to hopefully help make the engineering process a bit smoother as well as keep someone a bit more sane.

`environment.ts`: This is a rollup file for all the environment variables that the project uses as well as their fallbacks if none are provided. What this means is you can overwrite any of there easily **without having to change code** if you ever have a need to do so. Here's a command-line tip: you can set temporary environment variables on a per process basis by adding them before the command you want to run. Example: If we wish to set some value for `LOG_DB` temporarily, we can run the api like:

```
LOG_DB=1 npm start # this is a helpful command if you wish to log all sql commands to the DB when developing
```

Use [VSCode](https://code.visualstudio.com/) as your text editor/ IDE. There's already a debugging script setup in the repository that allows you to run this project through VSCode (the vscode script is called `Debug API`). Once running through VSCode you can set debug to your hearts content (**SERIOUSLY IF YOU'RE CODING WITH** `console.log` **STOP THAT**).

[Prettier](https://prettier.io/) is a library that's already installed and deals with code formatting. It will format all of your code on commit so that you never have to worry about style matching to the project. If you wish to see your code styled in real time, VSCode has a prettier plugin that will format on save for you (you don't _have_ to do this, it'll still format on commit). Alternatively, you can manually format all of the code in the project with:

```
npm run format
```

Use [Azure Data Studio](https://docs.microsoft.com/en-us/sql/azure-data-studio/download-azure-data-studio?view=sql-server-ver15) (or some other sql client) for easier sql queries/ testing while developing. We're using [PostgresSQL](https://www.postgresql.org/) so keep that in mind while googling how to query things as the syntax and features can be slightly different across sql flavors. **[AZURE DATA STUDIO REQUIRES A POSTGRES PLUGIN TO WORK WITH POSTGRESQL](https://docs.microsoft.com/en-us/sql/azure-data-studio/extensions/postgres-extension?view=sql-server-ver15)**. So if you want to use Azure Data Studio, you'll have to install that extension.

## Working in the Project

This project is built using

- [Typescript](https://www.typescriptlang.org/) - A superset of JavaScript. The language of the applciation
- [Express](https://expressjs.com/) - A Node REST framework.
- [TypeORM](https://typeorm.io/#/) - A Typescript ORM (manages the DB and all DB calls)

For those unfamiliar with Express, I'll attempt to give a brief rundown before talking about the project structure as a whole.

### Middleware

Express is a REST framework for node. Devs new to express will likely find one of the most perplexing parts of Express to be how it handles middleware. For those unfamiliar with what middleware is, it's largely a catch all term for code that connects together pieces of code in the api. In Express's context, middleware is basically everything that runs within each router; here's an example:

```typescript
Router.get('/:id', idAsInt, UserController.detail, serializer)
```

In express, all of the functions added after the route's path are considered middleware and each route can have as many middleware as is needed. In this project we separate out `controllers` into their own directory though they are still considered middleware.

Express's middleware is always a function with three arguments

```typescript
function middlwareName(req: Reques, res: Response, next: NextFunction)
```

Here's what you need to know:

- Middlewares are called sequentially in the order of the arguments list
- You pass data between middlewares by adding whatever you want to pass to the `req` object.
  - es `req.users = userResponse`
  - Because we're in typescript you'll have to add whatever parameter you're passing to `@types/express/index.d.s`
- Calling `next()` moves you to the next piece of middleware in the chain
  - Calling `next` with _any_ arguments is considered throwing an error and will stop you from going to the next middleware, and it will bubble back the chain to the `globalErrorHandler`.

### Control Flow

Here's the basic layout of the application

![control flow of the api](/docs/controlFlow.png 'Control Flow')

Let's take this from the top

- `index.ts`: where the application is bootstrapped from, controls all global server controls and middlewares
- `routers/index.ts`: largely a rollup for all the other routers. Can be used to add router specific middleware to routes/ subroutes
- `routers/route.ts`: Individual routes for each resource, where the list of middleware can be found. _All routers call unique middleware_
- Middleware: The above diagram is a bit of a misnomer. Not every endpoint will have validators, controllers, and serializers. Some will have all of those, some may have none. Each route will have at least one middleware, and the last middleware will deal with returning the requested data
- Validators: validates the bodies of requests
- Controllers: deals with setting status codes, and directing to services
- Services: Workhorse of the application. Deals with all major application logic and database calls
- Serializers: Formats the data to be a sane, reusable response.

The database models live outside of this control flow as they don't deal with any buisness logic. However services will use them to access the database. You can largely think of the the models as a 1:1 map to database tables.

### Common TypeORM Commands (Database/ Schema Stuff)

If the schema needs to be updated, you can do so by updating the models and running

```
npm run typeorm -- migration:generate -n generatedMigrationName
```

Doing so will attempt to create an auto migration from any changes within the `src/models` directory and add it to `src/migrations`. If an auto migration is generated for you (always check your auto migrations), you can run it with the above migration command

```
npm run typeorm --migration:run
```

And revert the latest migration with

```
npm run typeorm --migration:revert
```

## Production Builds

You can build the project as a production build

```
npm run build
```

This throws the production build into `./build` and can be run with

```
node index.js
```