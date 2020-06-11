- Start Date: 2018-09-21
- Relevant Team(s): Ember.js, Learning, Steering
- RFC PR: [https://github.com/emberjs/rfcs/pull/380](https://github.com/emberjs/rfcs/pull/380)
- Tracking: (leave this empty)

# Query Params Service

## Summary

This RFC proposes a new built-in Service for managing query params which would 
allow for query params to be accessed and manipulated from anywhere in the app.

Access to query params is currently restricted to the controller and the 
corresponding route. This results in some limitations with which users may 
consume query param data. By exposing query params on a service, application 
developers will be able to easily access the query params from deep within a 
component tree, removing the need to pass query param related data and actions 
down through many layers of components from the route template.


## Motivation

Modern SPA concepts have converged on the idea that query params should be 
easily accessible -- independent from the construct responsible for handling 
routing -- yet not forgotten, as query params are very tied to the URL. 
Like with the [RouterService](https://github.com/emberjs/rfcs/blob/master/text/0095-router-service.md),
it is common to have a need to perform routing behavior from deep down a 
component tree. 

Additionally, the current query params implementation feels 
very verbose for "just wanting to access a property" and has been frustrating 
to have to explain awkward behavior when on-boarding new developers who may be 
unfamiliar with Ember.

Accessing data within the url **should feel easy**.

**What's wrong with the existing query params?**
- For all but one use case, controllers _can_ be avoided. Query params force 
  controllers into existence for those who are trying to avoid them. 

- The caching mechanism is persistent beyond just child routes. _Any_ time a 
  route with query params is re-visited, the query params values will be 
  restored. This can be useful for those who are needing this behavior, but 
  for those who want to manage query params via transition or navigations,
  they'd need to set up some mechanism for resetting or clearing query params
  on activation or transition away from routes -- the query params 
  implementation becomes more of an obstacle than a feature.

## Detailed Design

------------ implementation idea? --------------

```ts
// Route at '@ember/routing/route';
export default class Route {
  @service('queryParams') qps;
  @controller(this.name) controller; // does this work like this? never used this

  @use [Symbol('QP')] entangleQueryParams(this, ...this.qps.all, ...this.controller.queryParams)
}

function entagleQueryParams(routeInstance, ...queryParams) {
  // ... details
}

class RouteQueryParamsManager {
  @service router;

  declare route: Route;

  updateUsable() {
    // re-run model hooks?
    this.router.refresh(this.route);
  }
}

```

### Accessing Query Params

```ts
import Component from "@glimmer/component";
import { inject as service } from '@ember/service';

export default class Pagination extends Component {
  @service queryParams;

  get currentPage() {
    // returns "1" from ?page=1
    return this.queryParams.page;
  }
  set currentPage(value) {
    // sets value to ?page=stringified-value
    // NOTE: this API is only possible if the service itself is a proxy to itself
    //       otherwise we need helper methods,
    this.queryParams.page = value;
  }

}
```

Having query params accessible on the router service would allow users to implement:

 - query param aware modals that may hide or show depending on the presence of a query param.
 - fill in form fields from a link.
 - filter / search components could update the query param property.
 - whatever else query params are used for outside of a SPA.

### Serialization / Deserialization

Everyone has different query param serialization and deserialization needs depending on a variety of factors.

By default, the query params will be serialized and deserialized via the builtin URLSearchParams API, and polyfilled for IE11.

Should someone decide to customize how serialization and deserialization transforms the query params, that can be done directly on the router:

```ts
import EmberRouter from '@ember/routing/router';

import config from '../config/environment';

export default class Router extends EmberRouter {
  location = config.locationType;
  rootURL = config.rootURL;

  static queryParamsConfig = {
    serialize(queryParams: object): string {
      // serialize object for query string
      // default to URLSearchParams, polyfilled for IE11
    },
    deserialize(queryString: string): object {
      // parse to object for use in `injectedRouter.queryString`
      // also default to URLSerachParams
    }
  };
};
```


This will address a long standing issues from as far back as 2016,
some new functionality for serialization and deserialization could be powered by [qs](https://www.npmjs.com/package/qs) ([3.4kb (gzip+min)](https://bundlephobia.com/result?p=qs@6.7.0)) or a lternatively, [URLSearchParams](https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams) -- this would enable the setting of arrays and objects which is not possible today.

### Translation between old and new

Some of the following is taken from the [Ember 3.18.0 Guides](https://guides.emberjs.com/v3.18.0/routing/query-params/)

<details>
<summary>transitionTo</summary>

</details>

<details>
  <summary>Causing the `model` Data to Refresh</summary>

```ts
export default class ArticlesRoute extends Route {
  queryParams = {
    category: {
      refreshModel: true
    }
  };

  model(params) {
    // This gets called upon entering 'articles' route
    // for the first time, and we opt into refiring it upon
    // query param changes by setting `refreshModel:true` above.

    // params has format of { category: "someValueOrJustNull" },
    // which we can forward to the server.
    return this.store.query('article', params);
  }
}
```

</details>


<details>
<summary>Updating the URL with replaceState</summary>

</details>

<details>
<summary>Map a controller's property to a different query param key</summary>

```ts
import Controller from '@ember/controller';

export default class ArticlesController extends Controller {
  queryParams = [{
    category: 'articles_category'
  }];

  category = null;
}
```

</details>


<details>
<summary>Re-setting Query Params on Transition</summary>


</details>

<details>
<summary>Deriving Initial Query Param Values from the `model` Data</summary>

</details>

<details>
<summary>Sticky Query Params Scoped to the Controller Instead of the `Model` Data</summary>

</details>

<details>
<summary>Sticky Query Params</summary>


By default, transitionTo will clear the query params, unless specified inside the transiion.

If query params are defined ahead of time as sticky, they will persist in the URL between sub routes.

This can be configured in the `Router.map` function:
```ts
Router.map(function() {
  this.route('application', { queryParams: 'bar' }, function() {
    this.route('faq');
  
    this.route('posts', { queryParams: ['foo', 'baz']} function() {  
      this.route('new');
      this.route('index');
      this.route(
        'show',
        { path: '/:postId', queryParams: ['hideComments', 'invertColors'] },
        function() {
          this.route('edit');
          this.route('comment');
          this.route('share');
        }
      );
    });
  });
});
```

This method of dealing with query params implies the following:
 - All query params, even if not specified are allowed. given the above example, I could use the qp "strongestAvenger" anywhere
 - Any sort of transition will clear the query params, unless it is defined as a sticky queryParam. So, if I'm on the posts/new route with the query params "foo" and "baz", and transition to posts/show, those query params are still available. If I navigate to the faq page, foo and baz are cleared from the URL.
 - The globally defined query params will stick around until cleared manually. If I visit faq?bar=2, and then transition to posts. the bar=2 query param will still be present.
 - The `this.queryParams` function will be available at every nesting of the route tree, but `queryParams` are also configurable in the route options hash.

Notes:
 - This does not imply a caching mechanism (like what controllers today allow)
 - If a certain app wants caching of query params as they exist today, an addon could be made to fill that gap.

 **A more concrete example: filter query params**
Given we want a way to search over a list of products, be able to view additional information about a selected search result, and save the search result for later, sticky params help maintain the search throughout all of the route navigations (similar to Amazon). 


```ts
Router.map(function() {
  this.route('application', { queryParams: 'bar' }, function() {
    this.route('search', { queryParams: [
      'term', 'isPrime', 'department', 'averageReview', 'brand', 
      'memoryType', 'processor', 'vRamCapacity', 'certification',
    ]}, function() {
      // index route will be the search results page
  
      // shows a selected result with additional information
      this.route('summary', { path: '/summary/:itemId' });
  
      // shows a modal with a field to name the search to be loaded later
      this.route('save');
    });
  });
});
```

1. visit `/search`.
2. type "RTX" into the search bar press the enter key.
3. the URL now shows `/search?term=RTX` and some results display as rows.
   The code for appending `term` may look like the following:

   ```ts
   @service router;

   this.router.setQueryParam('term', 'RTX');
   ```

4. maybe there is a side panel that has dynamically updated with relevant 
   potential search values, you check that you want "Prime" only results.
5. the URL now shows `/search?term=RTX&isPrime=true`.
6. clicking on the first result expands the row to fill more of the screen.
   The transition to the new route may look like the following:

   ```ts
   this.transitionTo('search.summary', '001');
   ```
7. the URL is now `/search/summary/001?term=RTX&isPrime=true`.
   the query params are still present 
   because the parent route defined sticky query params.
8. clicking "close" on the expanded row transitions back to `/search?term=RTX&isPrime=true`

   ```ts
   this.transitionTo('search');
   ```
9. clicking "save search" opens a modal where you may save your search results.
   implementation of the submit action make look like:

   ```ts
   @service queryParams;

   @action submit() {
     await fetch('some-url', {
       method: 'POST',
       body: JSON.stringify({
         name: this.name,
         search: this.queryParams.all
       }),
     });
   }

   ```

</details>


-------------------------------------------

The biggest change needed, which could _potentially_ be a breaking change, is that the allow-list on routes' queryParams hash will need to be removed. The controller-based way is static in that all known query params must be specified on the controller ahead of time. This has been a great deal of frustration amongst many developers, both new and old to ember.
This is a dynamic way to manage query params which hopefully aligns more with people's mental model of query params.

### Setting Query Params

Until IE11 support is dropped, we cannot wrap and set query params intuitively as a normal getter/setter as is proposed by this addon.

For example, this is not possible until IE11 support is dropped:

```ts
this.router.queryParams.strongestAvenger = 'Hulk';
```

This is due to the fact that IE11 only supports ES5, which [does not have Proxy support](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy#Browser_compatibility) and [cannot be polyfilled](https://babeljs.io/docs/en/learn#proxies).

> Due to the limitations of ES5, Proxies cannot be transpiled or polyfilled.

- _[https://babeljs.io/docs/en/learn#proxies](https://babeljs.io/docs/en/learn#proxies)


Instead, functions must be defined on the router that would take care of the setting of the query param's value.
```ts
// set a single parameter
this.router.setQueryParam('strongestAvenger', 'Captain Marvel');
// set many parameters
// this'll replace
this.router.setQueryParams({
  strongestAvenger: 'Carol Danvers',
  secondStrongest: 'Thor or Hulk?',
  ['kebab-case-param']: `kebab o' Thanos' armies`,
});
```

Just as it is today, setting query params in this way would not trigger a transition.
The key-value state of set query params would be available on `<RouterService>#queryParams` as well as an existing controller's computed properties as they exist today.

### Interop with Today's Query Params

Controllers will still need to have their queryParams explicitly set, but for maximum ineteropability:

**Setting a query param with the new API**
```ts
this.router.setQueryParam('key', 'value');
```
The current route is known, and the hierarchy of active controllers are also known. `setQueryParam` could look through the controllers to see if any of them define a query param `key` and then set it.

**Getting a query param with the new API**
```ts
this.router.queryParams.key;
```
Simalar to setting the query param, 
the getter could also look through the hierarchy of active controllers 
to see if a value exists. 

Accessing or Setting query params using the new APIs that happen to collide 
with controller-defined query params should throw a warning, 
as the interop would give the router's query params an implicit cache, 
which may not be present in a future without controllers.

**Setting a query param with the old API**
```ts
this.set('controllerQp', 'value');
```
It's possible to have `set` use `this.router.setQueryParam`, 
but it would need to have additional code defined in `set` 
which checks if the current instance is a `Controller` 
and if the `queryParams` array defines the key. 

**Getting a query param with the old API**
Because this is only possible with query params defined on controllers, 
the value of the query param *must* exist on the controller. 
There is no need to have compatibility with the router's queryParams here.

## How we teach this

Currently, query params _must_ be [specified on the controller](https://guides.emberjs.com/release/routing/query-params/):
```ts
export default class ArticlesController extends Controller {
  queryParams = ['page', 'filter', {
    // QP 'articles_category' is mapped to 'category' in our route and controller
    category: 'articles_category'
  }];

  category = null;
  page = 1;
  filter = 'recent';
}
```

Having query-param-related computed properties available everywhere will be a shift in thinking that "the controller manages query params" to "query params are a concern of the query params service".

The above example from the guides could be consumed in a route as the following.

```ts
import Route from '@ember/routing/route';
import { inject as service } from '@ember/service';

export default class ArticlesRoute extends Route {
  @service queryParams;

  // model is entangled with the query params?
  async model() {
    // all query params defined on a controller are available via the service
    let { page, filter, category } = this.queryParams;

    let posts = await fetch(`/api/posts?page=${page}&filter=${filter}&category=${category}`);
    
    return { posts };
  }
}
```

Even with no controller required, page,filter, and category would still be 
available -- however, the default value would be `''`.

To set a default value, people should be encouraged to use the same semantics
as with components and receiving args.

```ts
export default class ArticlesRoute extends Route {
  @service queryParams;

  get filter() {
    return this.queryParams.filter ?? 'recent';
  }

  // ...
  // model is entangled with the query params?
  // maybe a resource needs to be involved
  async model() {
    let { page, filter, category } = this;
    // ...
  }
}
```

## Drawbacks

- Some people may be relying on the controller query-params allow-list.
- Some people may be super tied in to controller query params cacheing.
