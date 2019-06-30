- Start Date: 2019-06-14
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: https://github.com/emberjs/rfcs/pull/502
- Tracking: (leave this empty)

# Explicit Service Injection

## Summary

When learning Ember and/or Dependency Injection, the question comes about how magic the injection strings are. The goal of this RFC is to propose an alternate default for injections that allow ctrl+clickability / "go to definition" from the injection to the class that defines the type of the injected instance.

Note: while this RFC mainly talks in terms in services, this applies to all injections.

## Motivation

The main goal is to _use the platform_ and enable "go to definition" support from service definitions so developers can more easily discover the where and how their service is defined.

## Detailed design

instead of:
```ts
@service notifications;
@service('notifications') notifications;
@service('messages/dispatcher') dispatcher;
```
This will be the new default:
```ts
@service(NotificationsService) notifications;
@service(MessageDispatcherService) dispatcher;
```

This means that the shorthand syntax of using `@service` without a parameter should be discouraged, and the use of the service class should be used in its place.

This isn't a new idea as it's very similar to how other platforms do dependency injection. 
- Java's [Dagger](https://square.github.io/dagger/):
    ```java
    class CoffeeMaker {
      @Inject Heater heater;
      @Inject Pump pump;

      ...
    }
  ```
- Java's [Guice](https://github.com/google/guice/wiki/Injections) does constructor-based injections:
    ```java
    public class RealBillingService implements BillingService {
        private final CreditCardProcessor processorProvider;
        private final TransactionLog transactionLogProvider;

        @Inject
        public RealBillingService(
            CreditCardProcessor processorProvider,
            TransactionLog transactionLogProvider
        ) {
            this.processorProvider = processorProvider;
            this.transactionLogProvider = transactionLogProvider;
        }
    }
    ```
- Guice is similar to [asp.net's default injection](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-2.2) system
  ```csharp
  public class MyDependency : IMyDependency
  {
      private readonly ILogger<MyDependency> _logger;

      public MyDependency(ILogger<MyDependency> logger)
      {
          this._logger = logger;
      }
  }
    ```

    These constructor-based dependency injection techniques are common in languages without decorators (or where decorators are hard to without sacrificing performance (common in Java)), but are much more verbose than decorator-based injection using properties on the class. Because Ember's current string-based dependency injection is powered by decorators _and_ concise, this RFC will not explore the option of constructor-based dependency injection. The primary goal is to _add_ functionality to the existing dependency injection system in Ember.


At present, the `@service` decorator wraps around the `ApplicationInstance#lookup` method -- something like this (roughly / hand-waiving the implementation details of decorators and getting access to the app instance):

```ts
function service(name) {
  return appInstance.lookup(`service:${name}`);
}
```

In addition to string-based keys, with this RFC, the service pseudo-function should check for the type of the parameter, and:

```ts
function service(nameOrClass) {
  if (typeof nameOrClass === 'string') {
    return appInstance.lookup(`service:${name}`);
  }

  return appInstance.lookup(nameOrClass);
}
```

In order for `lookup` to be able to take a class definition as an argument, there will need to be an alternative way to _lookup_ instances of services by the class. And as a convenience, when a class is used for lookup, if there is no existing registration found, lookup _will register the class and instantiate it for you_.

Consider:

```ts
appInstance.register(MyClass, MyClass);
appInstance.lookup(MyClass);
```

is the same as 
```ts
// no registration
appInstance.lookup(MyClass);
```

This will be most handy for decorators or other common abstractions that desire to interact with services (such as the router or store), but only have direct access to them via a component or route context.

On the `Registry`, there already exists a reference to the class definition when registering an entry to the container.

```ts
registry.register('model:user', Person, {singleton: false });
registry.register('fruit:favorite', Orange);
registry.register('communication:main', Email, {singleton: false});
```

A new map can exist on the registry that can appropriately be wired up to register, unregister, etc to handle the lookup-by-class-definition at the same time as the current registry behaves, allowing for backward compatibility with todays injection usage.

Examples: 

 - **service registration and override**

    This is a common scenario where a service may be defined explicitly for use an application, but in a _test_, some behavior may need to be overridden.

    ```ts
    class MyFooService extends Service {
      @tracked foo = 0;

      add() {
        this.foo++;
      }
    }
    ```
    When the application boots, as it is today, each service found in the `app` namespace, will be registered using the class reference.
    ```ts
    appInstance.register(MyFooService, MyFooService);
    // this would register MyFooService on the *key* MyFooService,
    // just as today, it would look like
    appInstance.register('service:my-foo-service', MyFooService);
    ```
    For full compatibility with today's applications, each server will have two registrations: one string key, and one class key.

    Coming back to the the intent of this example: wanting to override behavior in a test. This can be achieved by creating a class that subclasses the original service.
    ```ts
    class MyFooOverrideService extends MyFooService {
      add() {
        this.foo += 2;
      }
    }

    // both stubbing (in a test), or clobbering the originally registered service,
    //  would look the same:
    appInstance.register(MyFooService, MyFooOverrideService);
    // the key in this registration, MyFooService, must be an
    // ancestor of MyFooOverrideService.
    // The restriction there is to encourage to not simply 
    // replace unrelated services with each other. For example, 
    // replacing the router service with the ember-data store
    // does not make sense, and is disallowed.

    const service = appInstance.lookup(MyFooService);

    service instanceof MyFooOverrideService // true
    service instanceof MyFooService // true
    ```
    Even though the lookup is performed with `MyFooService`, we are given back an instance of `MyFooOverrideService` is returned.  

 - **less typing for lazy registration**
    ```ts
    class MyFooService extends Service {
      @tracked foo = 0;

      add() {
        this.foo++;
      }
    }

    // the above service has not been registered yet
    const myFooService = appInstance.lookup(MyFooService);
    // first time lookup without registration will register for you.

    myFooService instanceof MyFooService // true
    ```

 - **typescript fastboot example with instance initializers**
  
    ```ts
    // app/services/cookie/cookie-service.ts
    abstract class CookieService {
      abstract getValue(key: string): string {}

      abstract setValue(key: string, value: string): void {}
    } 

    // app/components/my-component.js
    import Component from '@glimmer/component';
    import { inject as service } from '@ember/service';

    class MyComponent extends Component {
      @service(CookieService) cookie;
    }

    // app/services/cookie/fastboot-service.js
    import CookieService from 'app-name/services/cookies/cookie-service';

    class FastbootCookieService extends CookieService {
      getValue(key: string) {
        return '';
      }
      // ...
    }

    // app/services/cookie/browser-service.js
    import CookieService from 'app-name/services/cookies/cookie-service';

    class BrowserCookieService extends CookieService {
      getValue(key: string) {
        return browser.cookies.get({
          name: key,
          url: window.location.href
        })
      }
      // ...
    }

    // app/instance-initializers/register-cookie-service;
    import CookieService from 'app-name/services/cookies/cookie-service';
    import FastbootCookieService from 'app-name/services/cookies/fastboot-service';
    import BrowserCookieService from 'app-name/services/cookies/browser-service';

    export function initialize(appInstance) {
      cost fastboot = appInstance.lookup('service:fastboot');
      
      if (fastboot.isFastBoot) {
        appInstance.register(CookieService, FastbootCookieService);
      } else {
        appInstance.register(CookieService, BrowserCookieService);
      }
    }

    export default { initialize };
    ```

Logic will be added to the register method to ensure that the lookup type either is the same as the service instance's type or is an ancestor type. This will prevent the ability to register unrelated classes that would break the implied class hierarchy that is assumed with dependency injection. 

### Usage in testing

#### Acceptance Tests

Given that we have a service:
```ts
// app-name/services/my-service
export default class MyService extends Service {
  someMethodThatIsInTheSuperClass() {
    return 'not stubbed';
  }
}
```

And assuming that in a route there is a service injection like the following:
```ts
import Route from '@ember/routing/route';
import ServiceToOverride from 'app-name/services/my-service';

export default class TheRoute extends Route {
  @service(ServiceToOverride) myService;
  // ...
}
```

During an acceptance test, it can be overwritten by doing the following:
```ts
import { module, test } from 'qunit';
import { setupApplicationTest } from 'ember-qunit';
import { getContext } from '@ember/test-helpers';

import ServiceToOverride from 'app-name/services/my-service';

module('Acceptance | test a thing', function(hooks) {
  setupApplicationTest(hooks);

  hooks.beforeEach(function() {
    let { owner } = getContext();

    class MockService extends ServiceToOverride {
      someMethodThatIsInTheSuperClass() {
        return 'stubbed';
      }
    }

    // register under the same 'key', ServiceToOverride
    owner.register(ServiceToOverride, MockService);

    // re-inject on the-route
    owner.application.inject('route:the-route', 'myService', ServiceToOverride);
  });
});
```

### Integration Tests

```ts
import { module, test } from 'qunit';
import { setupRenderingTest } from 'ember-qunit';
import { render } from '@ember/test-helpers';
import hbs from 'htmlbars-inline-precompile';
import Service from '@ember/service';

import LocationService from 'app-name/services/location';

class LocationStub extends LocationService {
  getCurrentCity() {
    return 'Indianapolis';
  }

  getCurrentCountry() {
    return '???';
  }
});

module('Integration | Component | location-indicator', function(hooks) {
  setupRenderingTest(hooks);

  hooks.beforeEach(function(assert) {
    this.owner.register(LocationService, LocationStub);
  });
});
```


## How we teach this

Instead of using strings or inferred injections, the guides should be updated to use the Class definition of a service that it intends to inject.

Ember Inspector and the unstable-ember-language-server will likely need to be updated to support this kind of lookup.  Additionally, we may want to explore the ember development server exposing a socket where tools can get runtime data about the application. This would allow the ember-language-server to not only know which concrete class is used for a dependency injection reference, but also it would be able to show all other possible overrides of a particular service.

There will no doubt be some resistence to this change, and for those who ponder the possibility of "just using a module" instead of dependency injection altogether, they may want to read [this article](https://blogs.endjin.com/2014/04/understanding-dependency-injection/) or [this article](https://blog.thecodewhisperer.com/permalink/keep-dependency-injection-simple) 

## Drawbacks

- slightly more verbose

## Alternatives

 - Convert the dependency injection lookup to a `WeakMap<Klass, instanceof Klass>`

    This would likely result in faster lookup, but would require more upfront work to change how the lookup method works on the `ApplicationInstance` / "owner". This _could_ be a separate RFC, but without benchmarking it's hard to say if this massive of a change would be worth it.

 - in C# + asp.net core, dependency injections are resolved in the constructor of a class. This would be _even more verbose_ than what is being proposed in this RFC, as for many situations in Ember, the constructor can be omitted. 


 ## Unresolved Questions

 These may be more for implementation, but as I was tracing how injections work.. it looks like the control flow path is:
  - `@service`
  - `appInstance.lookup`
    - `engineInstance.lookup` (Application inherits from Engine) 
      - registry defined on `RegistryProxyMixin`
      - instantiated via `this.buildRegistry();`
      - registry is an instance of `Registry`
        - defined at `ember.js/packages/@ember/-internals/container/lib/registry.ts` 
          - in here is where the bulk of the implementation of this RFC would live? (along with updating the type signatures of the methods all the way up the call tree (where not inferred))
