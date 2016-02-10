# Router - HTTP Request Router

[![Build Status](https://travis-ci.org/crysalead/router.svg?branch=master)](https://travis-ci.org/crysalead/router)
[![Code Coverage](https://scrutinizer-ci.com/g/crysalead/router/badges/coverage.png?b=master)](https://scrutinizer-ci.com/g/crysalead/router/)

This library extends the [FastRoute](https://github.com/nikic/FastRoute) implementation with some additional features:

 * Supports named routes and reverse routing
 * Supports sub-domain and/or prefix routing
 * Allows to set a custom dispatching strategy

## Installation

```bash
composer require crysalead/router
```

## API

### Defining routes

Example of routes definition:

```php
use Lead\Router\Router;

$router = new Router();

$router->bind($pattern, $handler);                      # route matching any request method
$router->bind($pattern, $options, $handler);            # alternative syntax with some options.
$router->bind($pattern, ['method' => 'get'], $handler); # route matching only get requests

// Alternative syntax
$router->get($pattern, $handler);    # route matching only get requests
$router->post($pattern, $handler);   # route matching only post requests
$router->delete($pattern, $handler); # route matching only delete requests
```

In the above example `$router` is a collection of routes. A route is registered using the `bind()` method and takes as parametters a route pattern, an optionnal options array and an handler.

A route pattern is a string representing an URL path. Placeholders can be specified using brackets (e.g `{foo}`) and matches `[^/]+` by default. You can however specify a custom pattern using the following syntax `{foo:[0-9]+}`. You can also add an array of patterns for a sigle route.

Furthermore you can use square brackets (i.e `[]`) to make parts of the pattern optional. For example `/foo[/bar]` will match both `/foo` and `/foobar`. Optional parts are only supported in a trailing position (i.e. not allowed in the middle of a route). You can also nest optional parts with the following syntax `/{controller}[/{action}[/{args:.*}]]`.

The second parameter is an `$options`. Possible values are:

* `'scheme'`: the scheme constraint (default: `'*'`)
* `'host'`: the host constraint (default: `'*'`)
* `'method'`: the method constraint (default: `'*'`)
* `'name'`: the name of the route (optional)
* `'namespace'`: the namespace to attach to a route (optional)

The last parameter is the `$handler` which contain the dispatching logic. The `$handler` is dynamically binded to the founded route so `$this` will stands for the route instance. The available data in the handler is the following:

```php
$router->bind('foo/bar', function($route, $response) {
    $route->scheme;     // The scheme contraint
    $route->host;       // The host contraint
    $route->method;     // The method contraint
    $route->params;     // The matched params
    $route->namespace;  // The namespace
    $route->name;       // The route's name
    $route->request;    // The routed request
    $route->response;   // The response (same as 2nd argument, can be `null`)
    $route->patterns(); // The patterns contraint
    $route->handlers(); // The route's handler
});
```

### Named Routes And Reverse Routing

To be able to do some reverse routing, you must name your route first using the following syntax:

```php
$route = $router->bind('foo#foo/{bar}', function () { return 'hello'; });

$router['foo'] === $route; // true
```

Then the reverse routing is done through the `link()` method:

```php
$link = $router->link('foo', ['bar' => 'baz']);
echo $link; // /foo/baz
```

### Grouping Routes

It's possible to apply contraints to a bunch of routes all together by grouping them into a dedicated of decicated scope using the router `->group()` method.

```php
$router->group('admin', ['namespace' => 'App\Admin\Controller'], function($r) {
    $router->bind('{controller}[/{action}]', function($route, $response) {
        $controller = $route->namespace . $route->params['controller'];
        $instance = new $controller($route->params, $route->request, $route->response);
        $action = isset($route->params['action']) ? $route->params['action'] : 'index';
        $instance->{$action}();
        return $route->response;
    });
});
```

### Sub-Domain And/Or Prefix Routing

To supports some sub-domains routing, the easiest way is to group routes related to a specific sub-domain using the `group()` method like in the following:

```php
$router->group(['host' => 'foo.{domain}.bar'], function($router) {
    $router->group('admin', function($router) {
        $router->bind('{controller}[/{action}]', function () {});
    });
});
```

### Dispatching

Dispatching is the outermost layer of the framework, responsible for both receiving the initial HTTP request and sending back a response at the end of the request's life cycle.

This step has the responsibility to loads and instantiates the correct controller, resource or class to build a response. Since all this logic depends on the application architecture, the dispatching has been splitted in two steps for being as flexible as possible.

#### Dispatching A Request

The URL dispatching is done in two steps. First the `route()` method is called on the router instance to find a route matching the URL. The route accepts as arguments:

* An instance of `Psr\Http\Message\RequestInterface`
* An url or path string
* An array containing at least a path entry
* A list of parameters with the following order: path, method, host and scheme

Then if the `route()` method returns a matching route, the `dispatch()` method is called on it to execute the dispatching logic contained in the route handler.

```php
use Lead\Router\Router;

$router = new Router();

$router->bind('foo/bar', function() { return "Hello World!"; });

$route = $router->route('foo/bar', 'GET', 'www.domain.com', 'https');

echo $route->dispatch(); // Can throw an exception if the route is not valid.
```

#### Dispatching A Request Using Some PSR-7 Compatible Request/Response

It also possible to use compatible Request/Response instance for the dispatching.

```php
use Lead\Router\Router;
use Lead\Net\Http\Cgi\Request;
use Lead\Net\Http\Response;

$request = Request::ingoing();
$response = new Response();

$router = new Router();
$router->bind('foo/bar', function($route, $response) { $response->body("Hello World!"); });

$route = $router->route($request);

echo $route->dispatch($response); // Can throw an exception if the route is not valid.
```

### Setting up a custom dispatching strategy.

To use your own strategy you need to create it first using the `strategy()` method.

Bellow an example of a RESTful strategy:

```php
use Lead\Router\Router;
use My\Custom\Namespace\ResourceStrategy;

Router::strategy('resource', new ResourceStrategy());

$router = new Router();
$router->resource('Home', ['namespace' => 'App\Resource']);

// Now all the following URL can be routed
$router->route('home');
$router->route('home/123');
$router->route('home/add');
$router->route('home', 'POST');
$router->route('home/123/edit');
$router->route('home/123', 'PATCH');
$router->route('home/123', 'DELETE');
```

```php
namespace use My\Custom\Namespace;

class ResourceStrategy {

    public function __invoke($router, $resource, $options = [])
    {
        $path = strtolower(strtr(preg_replace('/(?<=\\w)([A-Z])/', '_\\1', $resource), '-', '_'));

        $router->get($path, $options, function($route) {
            return $this->_dispatch($route, $resource, 'index');
        });
        $router->get($path . '/{id:[0-9a-f]{24}|[0-9]+}', $options, function($route) {
            return $this->_dispatch($route, $resource, 'show');
        });
        $router->get($path . '/add', $options, function($route) {
            return $this->_dispatch($route, $resource, 'add');
        });
        $router->post($path, $options, function($route) {
            return $this->_dispatch($route, $resource, 'create');
        });
        $router->get($path . '/{id:[0-9a-f]{24}|[0-9]+}' .'/edit', $options, function($route) {
            return $this->_dispatch($route, $resource, 'edit');
        });
        $router->patch($path . '/{id:[0-9a-f]{24}|[0-9]+}', $options, function($route) {
            return $this->_dispatch($route, $resource, 'update');
        });
        $router->delete($path . '/{id:[0-9a-f]{24}|[0-9]+}', $options, function($route) {
            return $this->_dispatch($route, $resource, 'delete');
        });
    }

    protected function _dispatch($route, $resource, $action)
    {
        $resource = $route->namespace . $resource . 'Resource';
        $instance = new $resource();
        return $instance($route->params, $route->request, $route->response);
    }

}
```

### Acknowledgements

- [Li3](https://github.com/UnionOfRAD/lithium)
- [FastRoute](https://github.com/nikic/FastRoute)
