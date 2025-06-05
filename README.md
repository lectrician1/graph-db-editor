# graph-editor

This is a WYSIWYG node property graph editor. I intend for it to be able to be connected to any graph database you like and for you to be able to edit it using the interface.

Right now you can edit a graph that is stored locally in your browser and download it and import it.

The interface to create the graph is extreamly convenient and it is very quick to create nodes and realtionships. It might be some of the fastest diagraming software out there.

It is built with Svelte and D3.JS

## Developing

Once you've created a project and installed dependencies with `npm install` (or `pnpm install` or `yarn`), start a development server:

```bash
npm run dev

# or start the server and open the app in a new browser tab
npm run dev -- --open
```

## Building

To create a production version of your app:

```bash
npm run build
```

You can preview the production build with `npm run preview`.

> To deploy your app, you may need to install an [adapter](https://svelte.dev/docs/kit/adapters) for your target environment.