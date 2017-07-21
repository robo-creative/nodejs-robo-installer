# Robo Installer
This is a simple module installer that helps you install all custom modules from a specific path into a `robo-container`.

_It is NOT intended for auto-installing all `node_modules` as you should avoid doing this._

_If you haven't used `robo-container` yet, you are strongly recommended to try it out first before getting started with this installer._

## Installing

```
npm install robo-installer
```

This will also automatically install `robo-container` for you if you don't have it.

## Getting Started

Under your app's root folder, create a new folder named `my-modules`, then create another folder named `a-module` under `my-modules`.

```
my-app
|- index.js
|- my-modules
|  |- a-module
```

In `a-module`, create two .js files named meta.js and payload.js. Content is as below:

meta.js

```js
module.exports = {
    // leave it as empty object
}
```

payload.js

```js
module.exports = {
    saySomething: function() {
        console.log(`Hello World!`);
    }
}
```
To install this module, add following statements into your app's `index.js`:

```js
const $ = require('robo-container');
var installer = require('robo-installer')($);
installer.install(`${__dirname}/my-modules`);
```

To ask the module to say something:

```js
$('a-module').saySomething(); // Hello World!
```

## Module Definition

A module must be a folder that contains following files:

- `meta.js` (required) a.k.a module's header, provides metadata of the module.
- `payload.js` (optional) a.k.a module's body, encapsulates module execution code.

### Module Metadata

Metadata is mandatory for all modules. It must be an object that is assigned to `module.exports` and may have following properties:

- `use` (optional): An array that specifies dependencies which will be passed to this module via Constructor or a Building Method.
- `set` (optional): An object that specifies dependencies which will be injected to module via Property Injection.
- `singleton` (optional): A flag that indicates if the module instance is singleton or not. False (transient) by default.
- `install` (optional): A method that is used for custom installation of the module. See Customizing Module Installation for more information.

For example, given your application has four custom modules: `users`, `log`, `emitter`, and `db`. All located in folder `modules`: 

```
your-app
|- index.js
|- modules
|  |- users
|  |  |- meta.js
|  |  |- payload.js
|  |- db
|  |  |- meta.js
|  |  |- payload.js
|  |- log
|  |  |- meta.js
|  |  |- payload.js
|  |- emitter
|  |  |- meta.js
|  |  |- payload.js
```

In which, `users` uses `db` to save a certain user to database, and uses `log` to log errors occurring in update operation. It also uses `emitter` to emit a `user-updated` event when the given user got updated.

```js
// users/payload.js
module.exports = (db, log) => new (function(db, log) {
    this.emitter = undefined;

    this.updateUser = function(user) {
        var emitter = this.emitter;
        db.update(user)
            .then(user => {
                emitter.emit('user-updated', user);
            })
            .catch(ex => {
                log.error(ex);
            });
    }

})(db, log);
```

So the metadata of the module `users` may look like:

```js
// users/meta.js
module.exports = {
    use: ['db', 'log'], // both db and log will be passed to module constructor.
    set: { emitter: 'emitter' } // emitter will be injected via property injection, after module is created.
    singleton: true // false if not specified.
}
```

### Module Payload

Module payload contains module body that is assigned to `module.exports`. The body can be one of following: An Object, A Method, or A Class. 

A module may or may not have payload. When a module only has metadata, it becomes Thin Module (see Thin Module for more information).

#### An Object

The module will be treated as an instance and will be installed to the container via Instance Binding.

```js
// payload.js
module.exports = {
    error: function(ex) {
        console.log(ex);
    }
}
```

#### A Method

The module will be treated as a method that returns instance of the module later on, and will be installed to the container via Method Binding.

```js
// payload.js
class Logger() {
    constructor (strategy) {
        this.strategy = strategy;
    }

    error(ex) {
        this.strategy.error(ex);
    }
}

module.exports = (active) => new Logger(active);
```

#### A Class

The module will be treated as a class and will be installed to the container via Class Binding.

```js
// payload.js
class Logger() {
    constructor (strategy) {
        this.strategy = strategy;
    }

    error(ex) {
        this.strategy.error(ex);
    }
}

module.exports = Logger;
```

## Nested Modules

A module can be stored within either another module or a folder. In this case, its name will be prepended with a namespace that is actually a chain of its ancestor modules/folders, separated by dots. For example:

When:

```
your-app
|- index.js
|- modules
|  |- db
|  |  |- mysql
|  |  |  |- meta.js
|  |  |  |- index.js
|  |  |- mongodb
|  |  |  |- meta.js
|  |  |  |- index.js
|  |- log
|  |  |- meta.js
|  |  |- payload.js
|  |  |- file-based
|  |  |  |- meta.js
|  |  |  |- index.js
```

Then:

```js
var db = $('db'); // undefined
var mysql = $('db.mysql'); // mysql module returned.
var mongodb = $('db.mongodb'); // mongodb module returned.

var log = $('log'); // log module returned.
var fileBasedLog = $('log.file-based'); // file-based log module returned.
```

This mechanism also brings you an ability to switch between modules. For example, given you have two different configuration sets, one for production and one for development. You can define `config` as a module with two sub-modules: `production` and `development`, and then set the active config to your prefered module.

```
your-app
|- index.js
|- modules
|  |- config
|  |  |- meta.js
|  |  |- index.js
|  |  |- development
|  |  |  |- meta.js
|  |  |  |- index.js
|  |  |- production
|  |  |  |- meta.js
|  |  |  |- index.js
```

Config:

```js
// meta.js
modules.exports = {
    use: ['config.development'], // sets active config to development.
    singleton: true
}

// payload.js
module.exports = (active) => active; // returns active config
```

Development:

```js
// meta.js
module.exports = {
    // leave as empty.
}

// payload.js
module.exports = {
    DB_HOST: 'localhost',
    DB_NAME: 'mydb'
    // and other settings...
}
```

Usage:

```js
var config = $('config');

var dbHost = config.DB_HOST; // 'localhost'
var dbName = config.DB_NAME; // 'mydb'
```

## Customizing Module Installation

Besides auto-installing modules with dependencies and life cycle given in `meta.js`, Robo Installer also gives you ability to customize installation of any particular module. To manually install a module, specify function `install` in its `meta.js`.

```js
// meta.js
module.exports = {
    install: function(container, name, path) {
        // your custom installation logic here.
    }
}
```

The function `install` accepts following parameters: 

- <strong>container</strong>: The instance of Robo-Container used to bind module to.
- <strong>name</strong>: Name of the module recognized by the Module Installer, e.g 'config', 'config.development'.
- <strong>path</strong>: Path to the folder that contains the module.

_Once specified, only function `install` will take effects and all other settings including `use`, `set` and `singleton` will be skipped._

Let's say your `pricing` module has more than one payload files: `gateway.js` and `connector.js` and you want to install them all at once:

```
your-app
|- index.js
|- modules
|  |- pricing
|  |  |- meta.js
|  |  |- gateway.js
|  |  |- connector.js
```

Your installation will look like:

```js
// meta.js
module.exports = {
    install: function($, name, path) {
        $.bind(`${name}.connector`).to(`${path}/connector.js`); // $('pricing.connector') will return a connector.
        $.bind(name).to(`${path}/gateway.js`).use(`${name}.connector`); // $('pricing') will return a gateway.        
    }
}
```

You can also use custom installation to install any `node_modules` to the container. For example:

```
your-app
|- index.js
|- modules
|  |- core
|  |  |- meta.js
```

Installation statements:

```js
// meta.js
module.exports = {
    install: function ($, name, path) {
        $.bind('express').to(() => require('express')).asSingleton();
        $.bind('app').to((express) => {
            var app = express();
            var bodyParser = require('body-parser');
            app.use(bodyParser.json());
            app.use(bodyParser.urlencoded({ extended: true }));
            return app;
        }).use('express').asSingleton();
        $.bind('http').to((app) => require('http').Server(app)).use('app').asSingleton();        
    }
}
```

_`$('core')` will throw a `ComponentNotFoundError` in this case._

## Thin Module

A Thin Module is a module that only has metadata, no payloads.

As Thin Module does not have any payloads, its `meta.js` can only accept function `install`. Any other option like `use`, `set` or `singleton` will not be applicable. The function `install` will be used to initialize the module inself and register it to the container.

Thin Module is useful in case of you want to have a light-weight module with small amount of code, or you want to use the module to install some `node_modules` as mentioned in Customizing Module Installation.

Below example uses a thin `config.development` module:

```js
// meta.js
module.exports = {
    install: function ($, name, path) {
        $.bind(name).to({
            DB_HOST: 'localhost',
            DB_NAME: 'mydb'
            // and more...
        });
    }
}
```