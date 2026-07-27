# sv

Everything you need to build a Svelte project, powered by [`sv`](https://github.com/sveltejs/cli).

## Creating a project

If you're seeing this, you've probably already done this step. Congrats!

```sh
# create a new project
npx sv create my-app
```

To recreate this project with the same configuration:

```sh
# recreate this project
npx sv@0.16.1 create --template minimal --types jsdoc --add prettier eslint --install npm /flood-history
```

## Developing

Once you've created a project and installed dependencies with `npm install` (or `pnpm install` or `yarn`), start a development server:

```sh
npm run dev

# or start the server and open the app in a new browser tab
npm run dev -- --open
```

## Building

To create a production version of your app:

```sh
npm run build
```

You can preview the production build with `npm run preview`.

> To deploy your app, you may need to install an [adapter](https://svelte.dev/docs/kit/adapters) for your target environment.

## Deploying to GitHub Pages

This repository is configured to deploy from GitHub Actions using `.github/workflows/deploy.yml`.

- The app is built with `@sveltejs/adapter-static` into `build/`.
- Production builds use the base path `/DataVizTCO` so assets resolve correctly under the repository site URL.
- In GitHub repository settings, set **Pages** to **Build and deployment → Source: GitHub Actions**.
