# `@conduitvc/node`

> Neutrino preset that supports building Node.js projects according to Conduit's
process and settings.

## Features

- Zero upfront configuration necessary to start developing and building a Node.js project
- Modern Babel compilation supporting ES modules, async functions, and dynamic imports
- Optionally supports Flow and TypeScript syntax
- By default targets the version of Node.js used to run webpack
- Supports automatically-wired sourcemaps
- Tree-shaking to create smaller bundles
- Hot Module Replacement with source-watching during development
- Chunking of external dependencies apart from application code
- Easily extensible to customize your project as needed

## Requirements

- Node.js ^8.10 or 10+
- Yarn v1.2.1+, or npm v5.4+
- Neutrino 9
- webpack 4
- webpack-cli 3

## Installation

`@conduitvc/node` can be installed via the Yarn client. Inside your project,
make sure that the dependencies below are installed as development dependencies.

#### Yarn

```bash
❯ yarn add --dev neutrino@next @conduitvc/node webpack webpack-cli
```

If you want to have automatically wired sourcemaps added to your project, add
`source-map-support`:

#### Yarn

```bash
❯ yarn add source-map-support
```

## Project Layout

`@conduitvc/node` follows the standard
[project layout](https://neutrinojs.org/project-layout/) specified by Neutrino.
This means that by default all project source code should live in a directory
named `src` in the root of the project. This includes JavaScript files that
would be available to your compiled project.

After installing Neutrino and the Node.js preset, add a new directory named
`src` in the root of the project, with a single JS file named `index.js` in it.

```bash
❯ mkdir src && touch src/index.js
```

Edit your `src/index.js` file with the following:

```js
import { createServer } from 'http';

const delay = ms => new Promise(resolve => setTimeout(resolve, ms));
const port = process.env.PORT || 3000;

createServer(async (req, res) => {
  await delay(500);
  console.log('Request!');
  res.end('hi!');
})
.listen(port, () => console.log(`Server running on port ${port}`));
```

Now edit your project's `package.json` to add commands for starting and building
the application:

```json
{
  "scripts": {
    "start": "webpack --watch --mode development",
    "build": "webpack --mode production"
  }
}
```

Then create a `.neutrinorc.js` file alongside `package.json`, which contains
your Neutrino configuration:

```js
module.exports = {
  use: [
    '@conduitvc/node',
  ],
};
```

And create a `webpack.config.js` file, that uses the Neutrino API to access the
generated webpack config:

```js
const neutrino = require('neutrino');

module.exports = neutrino().webpack();
```

Start the app, then either open a browser to the address in the console, or use
curl from another terminal window:

```bash
❯ yarn start
Server running on port 3000
```

```bash
❯ curl http://localhost:3000
hi!
```

## Building

`@conduitvc/node` builds assets to the `build` directory by default when running
`yarn build`. You can either serve or deploy the contents of this `build`
directory as a Node.js module, server, or tool. For Node.js this usually means
adding a `main` property to package.json pointing to the primary main built
entry point. Also when publishing your project to npm, consider excluding your
`src` directory by using the `files` property to whitelist `build`, or via
`.npmignore` to blacklist `src`.

```json
{
  "main": "build/index.js",
  "files": [
    "build"
  ]
}
```

_Note: While this preset works well for many types of Node.js applications, it's
important to make the distinction between applications and libraries. This
preset will not work optimally out of the box for creating distributable
libraries, and will take a little extra customization to make them suitable for
that purpose._

## Hot Module Replacement

While `@conduitvc/node` supports Hot Module Replacement for your app, it does
require some application-specific changes in order to operate. Your application
should define split points for which to accept modules to reload using
`module.hot`:

For example:

```js
import { createServer } from 'http';
import app from './app';

if (module.hot) {
  module.hot.accept('./app');
}

createServer((req, res) => {
  res.end(app('example'));  
}).listen(/* */);
```

Or for all paths:

```js
import { createServer } from 'http';
import app from './app';

if (module.hot) {
  module.hot.accept();
}

createServer((req, res) => {
  res.end(app('example'));  
}).listen(/* */);
```

Using dynamic imports with `import()` will automatically create split points and
hot replace those modules upon modification during development.

## Debugging

You can start the Node.js server in `inspect` mode to debug the process by
setting `neutrino.options.debug` to `true`.

## Preset options

You can provide custom options and have them merged with this preset's default
options to easily affect how this preset builds. You can modify Node.js preset
settings from `.neutrinorc.js` by overriding with an options object. Use
an array pair instead of a string to supply these options in `.neutrinorc.js`.

The following shows how you can pass an options object to the Node.js preset and
override its options, showing the defaults:

```js
module.exports = {
  use: [
    ['@conduitvc/node', {
      // Enables Hot Module Replacement. Set to false to disable
      hot: true,

      // Enables Flow syntax support. Set to true to enable.
      flow: false,

      // Enables TypeScript syntax support. Set to true to enable.
      typescript: false,

      // Targets the version of Node.js used to run webpack.
      targets: {
        node: 'current',
      },

      // Remove the contents of the output directory prior to building.
      // Set to false to disable cleaning this directory
      clean: {
        paths: [neutrino.options.output]
      },

      // Add additional Babel plugins, presets, or env options
      babel: {
        // Override options for @babel/preset-env, showing defaults:
        presets: [
          ['@babel/preset-env', {
            // Targets the version of Node.js used to run webpack.
            targets: { node: 'current' },
            useBuiltIns: 'entry',
          }],
        ],
      },
    }],
  ],
};
```

_Example: Override the Node.js Babel compilation target to Node.js v6:_

```js
module.exports = {
  use: [
    ['@conduitvc/node', {
      // Target specific versions via @babel/preset-env
      targets: {
        node: '6.0',
      },
    }],
  ],
};
```

## Customizing

To override the build configuration, start with the Neutrino documentation on
[customization](https://neutrinojs.org/customization/).
`@conduitvc/node` creates some conventions to make overriding the configuration
easier once you are ready to make changes.

By default Neutrino, and therefore this preset, creates a single **main**
`index` entry point to your application, and this maps to the `index.*` file in
the `src` directory. This means that this preset is optimized toward a single
main entry to your application. Code not imported in the hierarchy of the
`index` entry will not be output to the bundle. To overcome this you must either
define more mains via
[`options.mains`](https://neutrinojs.org/customization/#optionsmains), import
the code path somewhere along the `index` hierarchy, or define multiple
configurations in your `.neutrinorc.js`.

If the need arises, you can also compile `node_modules` by referring to the
relevant
[`compile-loader` documentation](https://neutrinojs.org/packages/compile-loader/#compiling-node_modules).

### Vendoring

This preset automatically vendors all external dependencies into a separate
chunk based on their inclusion in your package.json. No extra work is required
to make this work.

### Source minification

By default script sources are minified in production only, using
[terser-webpack-plugin](https://github.com/webpack-contrib/terser-webpack-plugin)
(which replaces `uglifyjs-webpack-plugin`). To customize the options passed to
`TerserPlugin` or even use a different minifier, override
`optimization.minimizer`.

_Example: Adjust the `terser` minification settings:_

```js
module.exports = {
  use: [
    '@conduitvc/node',
    (neutrino) => {
      // The `terser` plugin only exists in the configuration in production.
      if (process.env.NODE_ENV === 'production') {
        neutrino.config.optimization
          .minimizer('terser')
          .tap(([defaultOptions]) => [{
            ...defaultOptions,
            // https://github.com/webpack-contrib/terser-webpack-plugin#terseroptions
            // https://github.com/terser-js/terser#minify-options
            terserOptions: {
              mangle: false,
            },
          }]);
      }
    },
  ],
};
```

### Rules

The following is a list of rules and their identifiers which can be overridden:

| Name | Description | NODE_ENV |
| --- | --- | --- |
| `compile` | Compiles JS files from the `src` directory using Babel. Contains a single loader named `babel` | all |

### Plugins

The following is a list of plugins and their identifiers which can be
overridden:

_Note: Some plugins are only available in certain environments. To override
them, they should be modified conditionally._

| Name | Description | NODE_ENV |
| --- | --- | --- |
| `banner` | Injects source-map-support into the mains (entry points) of your application if detected in `dependencies` or `devDependencies` of your package.json. | Only when `source-map-support` is installed |
| `clean` | Clears the contents of `build` prior to creating a production bundle. | `'production'` |
| `start-server` | Start a Node.js for the first configured main entry point. | `'development'` |
| `hot` | Enables Hot Module Replacement. | `'development'` |

### Override configuration

By following the [customization guide](https://neutrinojs.org/customization/)
and knowing the rule, loader, and plugin IDs above, you can override and augment
the build by by providing a function to your `.neutrinorc.js` use array. You can
also make these changes from the Neutrino API in custom middleware.

_Example: Allow importing modules with a `.esm` extension._

```js
module.exports = {
  use: [
    '@conduitvc/node',
    (neutrino) => neutrino.config.resolve.extensions.add('.esm'),
  ],
};
```
