# Kibana knowledge share
This is a collection of knowledge that will make Kibana code and plugin development a little bit easier.


## Future proof your plugins
**Note:** Kibana is constantly changing and getting better. This document is targeted at the 6.0 baseline. Examples and architecture will not be accurate for future versions of kibana. 

1) Use React. We are in the long, slow, and painful process of removing Angular from Kibana
2) Do not use Bootstrap CSS classes or components. Bootstrap is getting removed from Kibana
3) Write Jest tests instead of Mocha tests
4) Inside of plugins, try to limit `ui/` imports and rely on `vis.API` for dependency injection. Future versions of Kibana will try to more module and limit allowing plugins to import directly from one module to another.


## Foundational knowledge (Javascript, ECMAScript 6 and TC-39, build process)

### Functions are objects
Functions are objects and can have properties.

```javascript
const myFunc = function(a, b) {
  return a + b;
}
myFunc.prop1 = 'someValue';
myFunc.prop2 = function () {
  return 'a function property of a function, my head hurts';
}

console.log( myFunc(1,2) ) // 3
console.log( myFunc.prop1 ) // someValue
console.log( myFunc.prop2() ) // a function property of a function, my head hurts
```

### [Destructuring assignment](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment)
Concise way of unpacking values from arrays, or properties from objects, into distinct variables.

```javascript
const myObj = {
  prop1: 'someValue',
  prop2: 'anotherValue'
};
```

```javascript
// Old way
const prop1 = myObj.prop1;
const prop2 = myObj.prop2;
```

```javascript
//Destructuring assignment
{ prop1, prop2 } = myObj;
```

[Kibana Example](https://github.com/elastic/kibana/blob/master/src/core_plugins/kibana/public/home/components/home.js#L14)

### [Arrow functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions)
Shorter syntax, does not have its own `this`

```javascript
const self = this;
// function syntax
function(arg1, arg2) {
  self.sum = arg1 + arg2;
}
```

```javascript
// Arrow functions
(arg1, arg2) => {
  this.sum = arg1 + arg2;
}
```

#### Arrow function concise body
```javascript
// block body
(x, y) => { 
  return x + y; 
}; 
```

```javascript
// concise body
(x, y) => x + y; 
```

### [Array iteration methods](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array)
Provide a declaritive syntax for collection iteration

```javascript
// imperative iteration
const myArray = [1,2,3];
const mySquaredArray = [];
for (let i=0, i<) {
  mySquaredArray[i] = myArray[i] * myArray[i];
}
```

```javascript
// Declaritive iteration
const myArray = [1,2,3];
const mySquaredArray = myArray.map((arrayElement) => {
  return arrayElement * arrayElement;
});
```

[Kibana Example](https://github.com/elastic/kibana/blob/master/src/core_plugins/kibana/public/home/components/home.js#L35)


### [Async functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)
An async function returns a `Promise` when called. The `Promise` will be resolved when the async function returns a value.


### [Spread syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_operator)

```javascript
const insertAtIndex = function(origArray, index, newItem) {
  const newArray = [];
  newArray.concat(origArray.slice(0, index));
  newArray.push(newItem);
  newArray.concat(origArray.slice(index + 1))
  return newArray;
}

```

```javascript
// spread operators + arrow functon with concise body
const insertAtIndex = (origArray, index, newItem) => [
  ...origArray.slice(0, index),
  newItem,
  ...origArray.slice(index + 1)
];
```

### Build process
Understanding the build process makes it easier to find things in source code 

#### babel
JavaScript transpiler that converts new JavaScript syntax into plain old ES5 JavaScript that can run in any browser (even the old ones).

```javascript
// Actuall code
getIndexPatterns = async (search) => {
  const resp = await this.props.scope.vis.API.savedObjectsClient.find({
    type: 'index-pattern',
    fields: ['title'],
    search: `${search}*`,
    search_fields: ['title'],
    perPage: 100
  });
  return resp.savedObjects;
}
```

```javascript
// What's running in browser
_this.getIndexPatterns = function () {
  var _ref2 = _asyncToGenerator( /*#__PURE__*/regeneratorRuntime.mark(function _callee(search) {
    var resp;
    return regeneratorRuntime.wrap(function _callee$(_context) {
      while (1) {
        switch (_context.prev = _context.next) {
          case 0:
            _context.next = 2;
            return _this.props.scope.vis.API.savedObjectsClient.find({
              type: 'index-pattern',
              fields: ['title'],
              search: search + '*',
              search_fields: ['title'],
              perPage: 100
            });

          case 2:
            resp = _context.sent;
            return _context.abrupt('return', resp.savedObjects);

          case 4:
          case 'end':
            return _context.stop();
        }
      }
    }, _callee, _this2);
  }));

  return function (_x) {
    return _ref2.apply(this, arguments);
  };
}
```

#### webpack
Module bundler. Builds `KIBANA_HOME/optimize/bundles/kibana.bundle.js` from `KIBANA_HOME/src/`

[webpackShims](https://github.com/elastic/kibana/tree/6.0/webpackShims)

webpack alias are used to make import statements cleaner but can cause confusion when looking for files by path.
* [ui alias](https://github.com/elastic/kibana/blob/6.0/src/ui/ui_bundler_env.js#L31) 
* [plugin alias](https://github.com/elastic/kibana/blob/6.0/src/ui/ui_bundler_env.js#L51)

Example from [markdown_vis.js](https://github.com/elastic/kibana/blob/6.0/src/core_plugins/markdown_vis/public/markdown_vis.js)
```
// Src is located at KIBANA_HOME/src/ui/public/vis/vis_factory.js
import { VisFactoryProvider } from 'ui/vis/vis_factory';

// Src is located at KIBANA_HOME/src/core_plugins/markdown_vis/public/markdown_vis.js
import 'plugins/markdown_vis/markdown_vis_controller';
```

## Kibana deep-dive

### Dependency injection
AngularJS doesn't handle namespace collisions for services. If you have 2 different modules with identical service names and include both modules in your app, which service will be made provided?

[Private](https://github.com/elastic/kibana/blob/6.0/src/ui/public/private/private.js) is Kibana's module loader that resolves this problem by mapping angular service's to a file path.

`Private` is a function that takes a single argument, `provider`. When the `Private` function is executed, it calls the `provider` function with injected angular dependencies and returns the result. 

https://github.com/elastic/kibana/blob/6.0/src/ui/public/agg_types/buckets/date_histogram.js#L12
```
export function AggTypesBucketsDateHistogramProvider(timefilter, config, Private) {
   ...
}
```

https://github.com/elastic/kibana/blob/6.0/src/ui/public/agg_types/index.js#L63
```
import { AggTypesBucketsDateHistogramProvider } from 'ui/agg_types/buckets/date_histogram';

Private(AggTypesBucketsDateHistogramProvider),
```

### [Indexed Array](https://github.com/elastic/kibana/blob/6.0/src/ui/public/indexed_array/indexed_array.js#L24)
An array with some special methods added to make searching easy.

```javascript
// this is generally a data-structure that IndexedArray is good for managing
const users = [
  { name: 'John', id: 69, username: 'beast', group: 'admins' },
  { name: 'Anon', id: 0, username: 'shhhh', group: 'secret' },
  { name: 'Fern', id: 42, username: 'kitty', group: 'editor' },
  { name: 'Mary', id: 55, username: 'sheep', group: 'editor' }
];

const indexedArray = new IndexedArray({
  index: ['username'],
  group: ['group'],
  order: ['id'],
  initialSet: users
});

const usersJohn = indexedArray.byUsername('beast');
```

### Registries
Registries are a function with a `register` property.
* Calling `register`, adds a `moduleProvider` to an array. `moduleProvider` is a function that returns a module.
* Calling the function itself
  * Calls each moduleProvider (with injected dependencies)
  * Each module is added to an Indexed Array.
  * returns Indexed Array.

[uiRegistry](https://github.com/elastic/kibana/blob/6.0/src/ui/public/registry/_registry.js#L48)
```javascript
export function uiRegistry(spec) {
  spec = spec || {};

  const constructor = _.has(spec, 'constructor') && spec.constructor;
  const invokeProviders = _.has(spec, 'invokeProviders') && spec.invokeProviders;
  const iaOpts = _.defaults(_.pick(spec, IndexedArray.OPT_NAMES), { index: ['name'] });
  const props = _.omit(spec, notPropsOptNames);
  const providers = [];

  const registry = function (Private, $injector) {
    // call the registered providers to get their values
    iaOpts.initialSet = invokeProviders
      ? $injector.invoke(invokeProviders, undefined, { providers })
      : providers.map(Private);

    // index all of the modules
    let modules = new IndexedArray(iaOpts);

    // mixin other props
    _.assign(modules, props);

    // construct
    if (constructor) {
      modules = $injector.invoke(constructor, modules) || modules;
    }

    return modules;
  };

  registry.displayName = '[registry ' + props.name + ']';

  registry.register = function (privateModule) {
    providers.push(privateModule);
    return registry;
  };

  return registry;
}
```


[vis_types.js](https://github.com/elastic/kibana/blob/6.0/src/ui/public/registry/vis_types.js)
```javascript
import { uiRegistry } from 'ui/registry/_registry';

export const VisTypesRegistryProvider = uiRegistry({
  name: 'visTypes',
  index: ['name'],
  order: ['title']
});
```

Example of registering new module [timeline/public/vis/index.js](https://github.com/elastic/kibana/blob/6.0/src/core_plugins/timelion/public/vis/index.js#L18)
```javascript
import { VisTypesRegistryProvider } from 'ui/registry/vis_types';

VisTypesRegistryProvider.register(TimelionVisProvider);

function TimelionVisProvider(Private) {
  const VisFactory = Private(VisFactoryProvider);
  
  return VisFactory.createAngularVisualization({ ...details omitted...});
}
```

Example of using the registry in [vis.js](https://github.com/elastic/kibana/blob/6.0/src/ui/public/vis/vis.js#L13)
```javascript
import { VisTypesRegistryProvider } from 'ui/registry/vis_types';

const visTypes = Private(VisTypesRegistryProvider);

this.type = visTypes.byName[type];
```

### SearchSource and Courier
[SearchSource](https://github.com/elastic/kibana/blob/6.0/src/ui/public/courier/data_source/search_source.js) is Kibana's wrapper around Elastic Search `search`.

SearchSources can inherit from other SearchSources. When a SearchSource is serialized into JSON, the inhertience tree is flattened into a single search body. Below is how the SearchSource hierarchy looks for Dashboards.
1) Each Visualization panel has its own search source that inherts from the application search source
2) Application search source: Contains filter pills and query bar state. It inherits from root search source.
3) Root search source: Contains global time range and pinned filters

SearchSource can be executed by calling `fetch` or `onResult`. Both result in request getting added to `Courier's` queue.

[fetch](https://github.com/elastic/kibana/blob/6.0/src/ui/public/courier/data_source/_abstract.js#L180)
```
SourceAbstract.prototype.fetch = function () {
  const self = this;
  let req = _.first(self._myStartableQueued());

  if (!req) {
    req = self._createRequest();
  }

  fetchSoon.these([req]);

  return req.getCompletePromise();
};
```

[onResults](https://github.com/elastic/kibana/blob/6.0/src/ui/public/courier/data_source/_abstract.js#L133)
```
SourceAbstract.prototype.onResults = function (handler) {
  const self = this;

  return new PromiseEmitter(function (resolve, reject) {
    const defer = Promise.defer();
    defer.promise.then(resolve, reject);

    self._createRequest(defer);
  }, handler);
};
```

[Courier](https://github.com/elastic/kibana/blob/6.0/src/ui/public/courier/courier.js) is Kibana's queueing mechanim around `_msearch`. All items in the request queue are serilized into a single `_msearch` request with a seperate `header\n body\n` section per item in the queue.

### The `msearch` life-cycle: dashboard, visualizations, SearchSource, and Courier
Visualization panels are wrapped in the [visualize directive](https://github.com/elastic/kibana/blob/6.0/src/ui/public/visualize/visualize.js). The visualize directive monitors the application state and calls request/response handlers as needed.

```
const requestHandler = getHandler(requestHandlers, $scope.vis.type.requestHandler);
const responseHandler = getHandler(responseHandlers, $scope.vis.type.responseHandler);
        
$scope.fetch = _.debounce(function () {
  requestHandler($scope.vis, $scope.appState, $scope.uiState, queryFilter, $scope.savedObj.searchSource)
    .then(requestHandlerResponse => {
      return Promise.resolve(responseHandler($scope.vis, requestHandlerResponse));
    }, e => {
      // error state - omitted for readability
    })
    .then(resp => {
      $scope.visData = resp;
      $scope.$apply();
      $scope.$broadcast('render');
      return resp;
    });
}, 100);

const handleQueryUpdate = ()=> {
  $scope.fetch();
};
queryFilter.on('update', handleQueryUpdate);

if ($scope.appState) {
  const stateMonitor = stateMonitorFactory.create($scope.appState);
  stateMonitor.onChange((status, type, keys) => {
    if (keys[0] === 'vis') {
      if ($scope.appState.vis) $scope.vis.setState($scope.appState.vis);
      $scope.fetch();
    }
    if ($scope.vis.type.requiresSearch && ['query', 'filters'].includes(keys[0])) {
      $scope.fetch();
    }
  });

  $scope.$on('$destroy', () => {
    stateMonitor.destroy();
  });
}

$scope.$listen(timefilter, 'fetch', $scope.fetch);
$scope.uiState.on('change', $scope.fetch);
resizeChecker.on('resize', $scope.fetch);
```

[courier](https://github.com/elastic/kibana/blob/6.0/src/ui/public/vis/request_handlers/courier.js) is the default request handler. `Courier` converts your Visualization Search Source into an `search` request that gets put on `Courier's` queue for fetching

```
import _ from 'lodash';
import { SearchSourceProvider } from 'ui/courier/data_source/search_source';
import { VisRequestHandlersRegistryProvider } from 'ui/registry/vis_request_handlers';

const CourierRequestHandlerProvider = function (Private, courier, timefilter) {
  const SearchSource = Private(SearchSourceProvider);

  return {
    name: 'courier',
    handler: function (vis, appState, uiState, queryFilter, searchSource) {


      if (queryFilter && vis.editorMode) {
        searchSource.set('filter', queryFilter.getFilters());
        searchSource.set('query', appState.query);
      }

      return new Promise((resolve, reject) => {
        searchSource.onResults().then(resp => {
          resolve(_.cloneDeep(resp));
        }).catch(e => reject(e));
        courier.fetch();
      });
    }
  };
};

VisRequestHandlersRegistryProvider.register(CourierRequestHandlerProvider);
```

### visualizations, saved objects, and aggregations

[SavedVis](https://github.com/elastic/kibana/blob/6.0/src/core_plugins/kibana/public/visualize/saved_visualizations/_saved_vis.js#L79) inherits from SavedObject. SavedVis links the Visualization aggregation configurations to the SearchSource.  
```
self.searchSource.aggs(function () {
  return self.vis.aggs.toDsl();
});
```

`vis.aggs` is an instance of [AggConfigs](https://github.com/elastic/kibana/blob/6.0/src/ui/public/vis/agg_configs.js). AggConfigs is a wrapper around Elastic Search aggregations.

```
visState: {
  "aggs": [
    {
      "id": "1",
      "enabled": true,
      "type": "count",
      "schema": "metric",
      "params": {}
    },
    {
      "id": "2",
      "enabled": true,
      "type": "date_histogram",
      "schema": "segment",
      "params": {
        "field": "@timestamp",
        "interval": "auto",
        "customInterval": "2h",
        "min_doc_count": 1,
        "extended_bounds": {}
      }
    },
    {
      "id": "3",
      "enabled": true,
      "type": "terms",
      "schema": "group",
      "params": {
        "field": "machine.os.raw",
        "otherBucket": false,
        "otherBucketLabel": "Other",
        "missingBucket": false,
        "missingBucketLabel": "Missing",
        "size": 5,
        "order": "desc",
        "orderBy": "1"
      }
    }
  ]
}
```

[AggConfigs.prototype.toDsl](https://github.com/elastic/kibana/blob/6.0/src/ui/public/vis/agg_configs.js#L93) serializes list of [AggConfig](https://github.com/elastic/kibana/blob/6.0/src/ui/public/vis/agg_config.js) instances into Elastic Search aggregation DSL.
```
{
  "size": 0,
  "_source": {
    "excludes": []
  },
  "aggs": {
    "2": {
      "date_histogram": {
        "field": "@timestamp",
        "interval": "1w",
        "time_zone": "UCT",
        "min_doc_count": 1
      },
      "aggs": {
        "3": {
          "terms": {
            "field": "machine.os.raw",
            "size": 5,
            "order": {
              "_count": "desc"
            }
          }
        }
      }
    }
  }
}
```

Tabify converts Elastic Search response into table
```
{
  "took": 0,
  "responses": [
    {
      "took": 0,
      "timed_out": false,
      "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
      },
      "hits": {
        "total": 14005,
        "max_score": 0.0,
        "hits": []
      },
      "aggregations": {
        "2": {
          "buckets": [
            {
              "key_as_string": "2017-08-07T00:00:00.000Z",
              "key": 1502064000000,
              "doc_count": 14005,
              "3": {
                "doc_count_error_upper_bound": 0,
                "sum_other_doc_count": 0,
                "buckets": [
                  {
                    "key": "win 7",
                    "doc_count": 2924
                  },
                  {
                    "key": "ios",
                    "doc_count": 2838
                  },
                  {
                    "key": "win 8",
                    "doc_count": 2750
                  },
                  {
                    "key": "win xp",
                    "doc_count": 2659
                  },
                  {
                    "key": "osx",
                    "doc_count": 1385
                  }
                ]
              }
            }
          ]
        }
      },
      "status": 200
    }
  ]
}
```

```
visData.series = [
 {
  "label": "win 7",
  "aggLabel": "Count",
  "aggId": "1",
  "count": 0,
  "values": [
   {
    "x": 1502064000000,
    "y": 2924,
    
      "$parent": {
       "key": 1502064000000,
       "value": 1502064000000,
       "aggConfig": {
        "id": "2",
        "enabled": true,
        "type": "date_histogram",
        "schema": "segment",
        "params": {
         "field": "@timestamp",
         "interval": "auto",
         "customInterval": "2h",
         "min_doc_count": 1,
         "extended_bounds": {}
        }
       },
       "type": "bucket"
      },
      "type": "bucket"
     },
     "type": "metric"
    },
    "extraMetrics": [],
    "aggConfig": {
     "id": "3",
     "enabled": true,
     "type": "terms",
     "schema": "group",
     "params": {
      "field": "machine.os.raw",
      "otherBucket": false,
      "otherBucketLabel": "Other",
      "missingBucket": false,
      "missingBucketLabel": "Missing",
      "size": 5,
      "order": "desc",
      "orderBy": "1"
     }
    },
    "series": "win 7"
   }
  ]
 },
 {
  "label": "ios",
  "aggLabel": "Count",
  "aggId": "1",
  "count": 0,
  "values": [
   {
    "x": 1502064000000,
    "y": 2838,
    "aggConfigResult": {
     "key": 2838,
     "value": 2838,
     "aggConfig": {
      "id": "1",
      "enabled": true,
      "type": "count",
      "schema": "metric",
      "params": {}
     },
     "$parent": {
      "key": "ios",
      "value": "ios",
      "aggConfig": {
       "id": "3",
       "enabled": true,
       "type": "terms",
       "schema": "group",
       "params": {
        "field": "machine.os.raw",
        "otherBucket": false,
        "otherBucketLabel": "Other",
        "missingBucket": false,
        "missingBucketLabel": "Missing",
        "size": 5,
        "order": "desc",
        "orderBy": "1"
       }
      },
      "$parent": {
       "key": 1502064000000,
       "value": 1502064000000,
       "aggConfig": {
        "id": "2",
        "enabled": true,
        "type": "date_histogram",
        "schema": "segment",
        "params": {
         "field": "@timestamp",
         "interval": "auto",
         "customInterval": "2h",
         "min_doc_count": 1,
         "extended_bounds": {}
        }
       },
       "type": "bucket"
      },
      "type": "bucket"
     },
     "type": "metric"
    },
    "extraMetrics": [],
    "aggConfig": {
     "id": "3",
     "enabled": true,
     "type": "terms",
     "schema": "group",
     "params": {
      "field": "machine.os.raw",
      "otherBucket": false,
      "otherBucketLabel": "Other",
      "missingBucket": false,
      "missingBucketLabel": "Missing",
      "size": 5,
      "order": "desc",
      "orderBy": "1"
     }
    },
    "series": "ios"
   }
  ]
 },
 {
  "label": "win 8",
  "aggLabel": "Count",
  "aggId": "1",
  "count": 0,
  "values": [
   {
    "x": 1502064000000,
    "y": 2750,
    "aggConfigResult": {
     "key": 2750,
     "value": 2750,
     "aggConfig": {
      "id": "1",
      "enabled": true,
      "type": "count",
      "schema": "metric",
      "params": {}
     },
     "$parent": {
      "key": "win 8",
      "value": "win 8",
      "aggConfig": {
       "id": "3",
       "enabled": true,
       "type": "terms",
       "schema": "group",
       "params": {
        "field": "machine.os.raw",
        "otherBucket": false,
        "otherBucketLabel": "Other",
        "missingBucket": false,
        "missingBucketLabel": "Missing",
        "size": 5,
        "order": "desc",
        "orderBy": "1"
       }
      },
      "$parent": {
       "key": 1502064000000,
       "value": 1502064000000,
       "aggConfig": {
        "id": "2",
        "enabled": true,
        "type": "date_histogram",
        "schema": "segment",
        "params": {
         "field": "@timestamp",
         "interval": "auto",
         "customInterval": "2h",
         "min_doc_count": 1,
         "extended_bounds": {}
        }
       },
       "type": "bucket"
      },
      "type": "bucket"
     },
     "type": "metric"
    },
    "extraMetrics": [],
    "aggConfig": {
     "id": "3",
     "enabled": true,
     "type": "terms",
     "schema": "group",
     "params": {
      "field": "machine.os.raw",
      "otherBucket": false,
      "otherBucketLabel": "Other",
      "missingBucket": false,
      "missingBucketLabel": "Missing",
      "size": 5,
      "order": "desc",
      "orderBy": "1"
     }
    },
    "series": "win 8"
   }
  ]
 },
 {
  "label": "win xp",
  "aggLabel": "Count",
  "aggId": "1",
  "count": 0,
  "values": [
   {
    "x": 1502064000000,
    "y": 2659,
    "aggConfigResult": {
     "key": 2659,
     "value": 2659,
     "aggConfig": {
      "id": "1",
      "enabled": true,
      "type": "count",
      "schema": "metric",
      "params": {}
     },
     "$parent": {
      "key": "win xp",
      "value": "win xp",
      "aggConfig": {
       "id": "3",
       "enabled": true,
       "type": "terms",
       "schema": "group",
       "params": {
        "field": "machine.os.raw",
        "otherBucket": false,
        "otherBucketLabel": "Other",
        "missingBucket": false,
        "missingBucketLabel": "Missing",
        "size": 5,
        "order": "desc",
        "orderBy": "1"
       }
      },
      "$parent": {
       "key": 1502064000000,
       "value": 1502064000000,
       "aggConfig": {
        "id": "2",
        "enabled": true,
        "type": "date_histogram",
        "schema": "segment",
        "params": {
         "field": "@timestamp",
         "interval": "auto",
         "customInterval": "2h",
         "min_doc_count": 1,
         "extended_bounds": {}
        }
       },
       "type": "bucket"
      },
      "type": "bucket"
     },
     "type": "metric"
    },
    "extraMetrics": [],
    "aggConfig": {
     "id": "3",
     "enabled": true,
     "type": "terms",
     "schema": "group",
     "params": {
      "field": "machine.os.raw",
      "otherBucket": false,
      "otherBucketLabel": "Other",
      "missingBucket": false,
      "missingBucketLabel": "Missing",
      "size": 5,
      "order": "desc",
      "orderBy": "1"
     }
    },
    "series": "win xp"
   }
  ]
 },
 {
  "label": "osx",
  "aggLabel": "Count",
  "aggId": "1",
  "count": 0,
  "values": [
   {
    "x": 1502064000000,
    "y": 1385,
    "aggConfigResult": {
     "key": 1385,
     "value": 1385,
     "aggConfig": {
      "id": "1",
      "enabled": true,
      "type": "count",
      "schema": "metric",
      "params": {}
     },
     "$parent": {
      "key": "osx",
      "value": "osx",
      "aggConfig": {
       "id": "3",
       "enabled": true,
       "type": "terms",
       "schema": "group",
       "params": {
        "field": "machine.os.raw",
        "otherBucket": false,
        "otherBucketLabel": "Other",
        "missingBucket": false,
        "missingBucketLabel": "Missing",
        "size": 5,
        "order": "desc",
        "orderBy": "1"
       }
      },
      "$parent": {
       "key": 1502064000000,
       "value": 1502064000000,
       "aggConfig": {
        "id": "2",
        "enabled": true,
        "type": "date_histogram",
        "schema": "segment",
        "params": {
         "field": "@timestamp",
         "interval": "auto",
         "customInterval": "2h",
         "min_doc_count": 1,
         "extended_bounds": {}
        }
       },
       "type": "bucket"
      },
      "type": "bucket"
     },
     "type": "metric"
    },
    "extraMetrics": [],
    "aggConfig": {
     "id": "3",
     "enabled": true,
     "type": "terms",
     "schema": "group",
     "params": {
      "field": "machine.os.raw",
      "otherBucket": false,
      "otherBucketLabel": "Other",
      "missingBucket": false,
      "missingBucketLabel": "Missing",
      "size": 5,
      "order": "desc",
      "orderBy": "1"
     }
    },
    "series": "osx"
   }
  ]
 }
]
```

Here is an example with split chart.


## UI framework
Re-usable [UI components](https://github.com/elastic/kibana/tree/master/ui_framework). **Warning:** This has been deprecated in 6.2 and will be replaced by [Elastic UI Framework](https://github.com/elastic/eui)


## Plugin examples
6.0 resources
* [Developing Kibana Visualizations - video](https://www.elastic.co/webinars/creating-custom-kibana-visualizations)
* [Developing Visualizations - kibana documentation](https://www.elastic.co/guide/en/kibana/6.0/development-visualize-index.html)
* [Developing new Kibana visualizations - blog](https://www.elastic.co/blog/developing-new-kibana-visualizations)
* Not a plugin link but found this blog post about [custom region maps](https://www.elastic.co/blog/custom-region-maps-in-kibana-6-0) that I think you would be interested in

The best way to build plugins is to look at working examples. Kibana uses its own plugin system so there are lots of great examples in the code base.

### New REST endpoint
Specify [init](https://github.com/elastic/kibana/blob/6.0/src/core_plugins/timelion/index.js#L87) element in plugin definition.

[init](https://github.com/elastic/kibana/blob/6.0/src/core_plugins/timelion/init.js#L4) is a function that gets pass `hapi` server object when called.

Add enpoint by [adding route to server](https://github.com/elastic/kibana/blob/6.0/src/core_plugins/timelion/server/routes/run.js#L15)

```javascript
// Plugin 
export default function (kibana) {
  return new kibana.Plugin({
    require: ['kibana', 'elasticsearch'],
    uiExports: {},
    init: function (server) {
      server.route({
        method: ['POST', 'GET'],
        path: '/api/timelion/run',
        handler: async (request, reply) => {
          // do stuff
          const results = [];
          reply(results);
        }
      });
    }
  });
}
```

### New Advanced Setting config value
Specify [uiSettingDefaults](https://github.com/elastic/kibana/blob/6.0/src/core_plugins/timelion/index.js#L44) property of `uiExports`.

```javascript
export default function (kibana) {
  return new kibana.Plugin({
    require: ['kibana', 'elasticsearch'],
    uiExports: {
      uiSettingDefaults: {
        'timelion:showTutorial': {
          value: false,
          description: 'Should I show the tutorial by default when entering the timelion app?'
        }
      }
    }
  });
}

```


### Add property to kibana.yml
Specify `config` property in plugin definition. 

`config` is a function that gets passed a Joi schema instance.
Update the Joi schema instance with new property(s). 

Use `injectDefaultVars` to expose the property for front-end code

```javascript
// Plugin 
export default function (kibana) {
  id: 'myPlugin',
  configPrefix: 'my.namespaced.plugin',
  return new kibana.Plugin({
    require: ['kibana', 'elasticsearch'],
    uiExports: {
      injectDefaultVars(server, options) {
        return {
          myNewProperty: options.myNewProperty
        };
      }
    },
    config: function (Joi) {
      return Joi.object({
        enabled: Joi.boolean().default(true),
        myNewProperty: Joi.string(),
      }).default();
    }
  });
}
```

Add new property to kibana.yml
```
my.namespaced.plugin.myNewProperty: "hello world"
```

Use chrome to access the property in your plugin
```javascript
import chrome from 'ui/chrome';

const myNewProperty = chrome.getInjected('myNewProperty');
```

### Visualization plugins

Visualization plugins were completely refactored in 6.0.

#### Access Kibana dependencies
Access Kibana dependencies from `vis.API` instead of import providers and calling `Private(Provider)`

https://github.com/elastic/kibana/blob/6.0/src/ui/public/vis/vis.js#L58
```
this.API = {
  savedObjectsClient: savedObjectsClient,
  SearchSource: SearchSource,
  indexPatterns: indexPatterns,
  timeFilter: timefilter,
  filterManager: filterManager,
  queryFilter: queryFilter
};
```

##### Accessing saved objects
`savedObjectsClient` provides a future proof method 

```
import { MyReactTab } from './components/editor/controls_tab';
import { VisFactoryProvider } from 'ui/vis/vis_factory';
import { VisTypesRegistryProvider } from 'ui/registry/vis_types';

function MyVisPluginProvider(Private) {
  const VisFactory = Private(VisFactoryProvider);

  // return the visType object, which kibana will use to display and configure new Vis object of this type.
  return VisFactory.createBaseVisualization({
    editor: 'default',
    editorConfig: {
      optionTabs: [
        {
          name: 'myTab',
          title: 'My tab',
          editor: MyReactTab
        }
      ]
    }
  });
}
```

```
import PropTypes from 'prop-types';
import React, { Component } from 'react';

export class MyReactTab extends Component {

  getIndexPatterns = async (search) => {
    const resp = await this.props.scope.vis.API.savedObjectsClient.find({
      type: 'index-pattern',
      fields: ['title'],
      search: `${search}*`,
      search_fields: ['title'],
      perPage: 100
    });
    return resp.savedObjects;
  }

  render() {
    return (
      <div></div>
    );
  }
}

MyReactTab.propTypes = {
  scope: PropTypes.object.isRequired,
  stageEditorParams: PropTypes.func.isRequired
};
```

Live example - [input controls ControlsTab](https://github.com/elastic/kibana/blob/6.1/src/core_plugins/input_control_vis/public/components/editor/controls_tab.js#L15)

#### Request handlers
Visualization request handler gets called when the dashboard needs to pull new data. This happens when filters are added/removed/changed, when timepicker is updated, or when page is refreshed.

[courier](https://github.com/elastic/kibana/blob/6.0/src/ui/public/vis/request_handlers/courier.js) is the default request handler. `Courier` converts your "Data tab" Aggregation Configurations into an `msearch` request that gets sent to Elasticsearch.

#### Response handler
Function that receives the data from a request handler and converts it into a usable format. The response from `Courier` request handler is Elasticsearch aggregation results.

The Default response handler converts Elastic Search aggregation results into a tabular format.

