# How Drupal 10's core architecture works

When I started learning drupal, I found information about its actual structure difficult to find, and for good reason, the point of Drupal is to abstract that all away as much as we can reasonably manage.  But my developer's curiosity got the better of me, so here we go.

I've tried to break down the architecture of this into chunks that make up the basic flow of how we get from the server calling PHP to where where our controllers are run.  I'm leaving out a LOT of detail to make this outline digestible.  My hope is that this will serve as a map for plumbing the depths of Drupal when that detail is needed.


### 1. Entry

One can bootstrap Drupal a number of ways, but today we'll go over the main index.php entrypoint found at your drupal application root.  This file is short and does a few important things
 
- Create the `$autoloader`  which allows us to find all of our php classes
- Creates a `$request` object with `Symfony\Component\HttpFoundation\Request::createFromGlobals()` that uses the globals liek `$_GET` and `$_POST` to grab all the info we need
- Creates an instance of `Drupal\Core\DrupalKernel($autoloader)`  and then calls `handle($request)` to kickoff the whole application which will return a response object that can be used to echo our final response.

### 2. Drupal Kernel

- Sets up and understands the current environment.  Thins like checking for code caching dependencies or setting PHP session cookie settings.  
- Creates and compiles the Dependency Injection (DI) Container `Drupal\Component\DependencyInjection\Container`.  If you're unfamiliar with dependency injection, I recommend having a quick google before continuing as its an increasingly important design pattern in the PHP world.  In this case the DI container compiles all of our service configurations from our \*services.yml files.  This includes all of our event listeners which will be important later on.
- Creates an instance of the Symfony HttpKernel `Symfony\Component\HttpKernel\HttpKernel` that is instantiated with access to the all the events listeners/subscribers compiled with the DI container.  
- The HttpKernel's `handle($request)` function is called.

### 2. The Symfony Kernel

We don't have to implement an interface of the Syfony Kernel, or extend it to tie in Drupal (although Drupal does give it a very light wrapper).  Instead it uses the events we mentioned earlier:

- Dispatches the event `kernel.request`.  There are a number of listeners that this event triggers, but today we'll talk through two of them, their definitions can be found in `core/core.services.yml`.
 - `router_listener` sets the name and method of a the appropriate controller that matches our route on the request.
 - `router.route_preloader` loads in all of routes so we can verify our requested routes.

- Now that we finally have the controller (maybe a default drupal contoller, or a custom one from our module) we call it and our business logic can finally begin!  At the end of it, the controller should return a `Symfony\Component\HttpFoundation\Response` object that gets passed back to the Drupal Kernel and index.php before call `$response->send()` and terminating the application

### Conclusions

Drupal sits squarly on top of Symfony.  Many Symfony classes have been extended to make performance enhancements or accomodate drupalisms.   Notably the Symfony HttpKernal remains almost entirely untouched.  Drupal ties itself into the Kernel by listening to default Symphony kernel events to do things like match our route and set the controller on the request object.

