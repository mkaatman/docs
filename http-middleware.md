# HTTP Middleware

## Table of Contents
1. [Introduction](#introduction)
2. [Manipulating the Request](#manipulating-the-request)
3. [Manipulating the Response](#manipulating-the-response)
4. [Global Middleware](#global-middleware)
5. [Route Middleware](#route-middleware)
  
<h2 id="introduction">Introduction</h2>
HTTP middleware are classes that sit in between the `Kernel` and `Controller`.  They manipulate the request and response to do things like authenticate users or enforce CSRF protection for certain routes.  They are executed in series in a [pipeline](pipelines).  Let's take a look at an example:

```php
use MyApp\Authentication;
use RDev\HTTP\Middleware;
use RDev\HTTP\Requests;
use RDev\HTTP\Responses;

class Authentication implements Middleware\IMiddleware
{
    private $authenticator = null;
    
    // Inject any dependencies your middleware needs
    public function __construct(Authentication\Authenticator $authenticator)
    {
        $this->authenticator = $authenticator;
    }

    // $next consists of the next middleware in the pipeline
    public function handle(Requests\Request $request, \Closure $next)
    {
        if(!$this->authenticator->isLoggedIn())
        {
            return new Responses\RedirectResponse("/login");
        }
        
        return $next($request);
    }
}

// Add this middleware to a route
$router->post("/users/posts", [
    "controller" => "MyApp\\UserController@createPost",
    "middleware" => "MyApp\\Authenticate" // Could also be an array of middleware
]);
```

Now, the `Authenticate` middleware will be run before the `createPost()` method is called.  If the user is not logged in, he'll be redirected to the login page.

> **Note:** If middleware does not specifically call the `$next` closure, none of the middleware after it in the pipeline will be run.

<h2 id="manipulating-the-request">Manipulating the Request</h2>
To manipulate the request before it gets to the controller, make changes to it before calling `$next($request)`:

```php
use RDev\HTTP\Middleware;

class RequestManipulator implements Middleware\IMiddleware
{
    public function handle(Requests\Request $request, \Closure $next)
    {
        // Do our work before returning $next($request)
        $request->getHeaders()->add("SOME_HEADER", "foo");
        
        return $next($request);
    }
}
```

<h2 id="manipulating-the-response">Manipulating the Response</h2>
To manipulate the response after the controller has done its work, do the following:

```php
use RDev\HTTP\Middleware;
use RDev\HTTP\Responses;

class ResponseManipulator implements Middleware\IMiddleware
{
    public function handle(Requests\Request $request, \Closure $next)
    {
        $response = $next($request);
        
        // Make our changes
        $cookie = new Responses\Cookie("my_cookie", "foo", \DateTime::createFromFormat("+1 week"));
        $response->getHeaders()->setCookie($cookie);
        
        return $response;
    }
}
```

<h2 id="global-middleware">Global Middleware</h2>
Global middleware is middleware that is run on every route.  To add middleware to the list of global middleware, add the fully-qualified middleware class' name to the array in `configs/http/middleware.php`.

<h2 id="route-middleware">Route Middleware</h2>
To learn how to register middleware with routes, read the [routing tutorial](routing#middleware).  You can also learn how to [add middleware to route groups](routing#group-middleware) there.