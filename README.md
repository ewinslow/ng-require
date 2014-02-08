ng-require
==========
Lazy-loading for Angular based on AMD

This documentation is very out of date.

https://github.com/ewinslow/ng-elgg is your best bet for example code, which should be more accurate.


Goals
-----
* Lightweight registration format to minimize overhead.
* Minimize initial payload size and parse time
* Give Angular team meaningful feedback on async DI

Design
------

Modules are defined by telling angular to load particular services/directives/etc from a module namespace. In this case the module namespace is "ng/modules". There are more details below.

```js
// ng/modules/module.js
define(function(require) {
	var angular = require(‘angular’);
	var createModule = require(‘ngRequire/createModule’);
	var otherModule = require(‘some/other/ngModule’);

	return createModule(‘ng/modules’,
		otherModule
	], { 
    “directives”: {},
  	“factories”: {},
  	“filters”: {},
  	“services”: {},
  	“states”: {}
  });
});
```

Adding third-party dependencies
-------

### install with volo
They should be installed to vendors/assets and be positioned in the conventional AMD format.

### Add a dependency on the module to main.js
If the third-party dependency is an angular module, you need to include it in the angular.bootstrap list of dependencies.

### Example main.js file
```js
define(‘main’, function(require) {
	var angular = require(‘angular’);

	angular.bootstrap(document, [
		require(‘elgg/core/module’).name,
		require(‘some/other/module’).name
	]);
});
require([‘main’]);
```

Adding a new service
-----
In your module’s config:

```js
createModule(‘my/mod’, [], {
	“values”: {
		“someServiceValue”: theActualValue
	},

  “services”: [
    “myServiceName”,
    “otherServiceName”,
  ],
  
  “factories”: [
    “service”, 
    “service2”,
  ]
});
```

This will:
 * Synchronously register the “someServiceValue” service as theActualValue.
 * Tells ngRequire that when “myServiceName” is injected, it should try to pull the class/constructor definition asynchronously from module “my/mod/services/MyServiceName”; similarly for otherServiceName.
 * Tells ngRequire that when “service” is injected, it should try to pull the factory definition asynchronously from the “my/mod/factories/service” module; similarly for “service2”.

If you want to use an existing class, you can do the following in your my/mod/services/MyServiceName module:

```js
define(function(require) {
	return require(‘the/existing/Class’);
});
```

Similarly, if you want to just use an existing factory function, you can do:

```js
define(function(require) {
	return require(‘the/existing/factory’);
});
```

Now you can use the service by name anywhere as you normally would and it will be lazy loaded.

Caveats:
 * if you inject $injector anywhere, only already-loaded services will be available with $injector.get() because that API is synchronous. The Angular team is considering an async API for v2.
 * There is no way to asynchronously inject providers. You should just register them synchronously on your module as normal.

### Adding a new directive
You should be familiar with Angular directives.

In your module’s config parameter:

```js
createModule(“my/mod”, [], {
  “directives”: {
  	“myDirectiveName”: true
  }
});
```

The default module ID is ng/directives/myDirectiveName/factory. You can also pass any other module id explicitly.

#### Define the factory

```js
define(function(require) {
  return {
  	restrict: ‘A’, // Or ‘C’, or ‘E’
  	template: require(‘text!./template.ng’),
  	controller: require(‘./Controller’),
  	link: require(‘./link’)
  };
});
```

This is slightly different than the stock angular directive registration system:
 * Automatic lazy-loading of directive dependencies (including filters and sub-directives that are only implicitly used in the template).
 * Makes the link function dependency-injected (including $scope, $element, $attrs, and $ctrls).

Some things to note:
 * Use AMD text! dependencies instead of templateUrl. If you don’t, we won’t be able to lazy-load the dependencies of that template.
 * You must return only a simple config object instead of a factory function. The only reason Angular uses a factory function is to inject services, but we have made the link function injectable so you don’t need the outer function.
 * If you need anything more advanced than the above options you should just define your directive synchronously in your module’s config.

#### Add tests (TODO)
In the _test.js file, add a Jasmine-style test.

```js
define(function(require) {
	// Lazy-loading magic
	var ngRequire = require(‘ngRequire’);
	beforeEach(module(ngRequire.name));

	// TODO(ewinslow): How to mock dependencies?

	describe(‘myDirectiveName’, function() {
		it(‘has some behavior’);
		// ...etc
	});
});
```

### Adding a new state (i.e. new route or new page)

In your module's config:

```js
createModule("my/mod", [], {
  "states": {
    "my.state.name": {
      "parent": "some.other.state", // optional
      "url": "/url/path/to/match"
    }
  }
});
```

#### Define state configuration file
ng/states/my.state.name/my.state.name.js

If this is not defined, the state will fail to load.

my.state.name.js should look like this:

```js
define(function(require) {
	return {
		controller: require(‘./MyStateNameCtrl’),
		template: require(‘text!./my.state.name.ng’),
		resolve: {
			foo: function(bar) { return bar.getFoo(); }
		}
	};
});
```

TODO:
 * Add a linter that runs through all states and does a “static” check for the dependencies to warn people ahead of time.
 * Add a yo task that builds out these dependencies automagically


