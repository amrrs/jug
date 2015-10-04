Hello World!
------------

    library(jug)

    jug() %>%
      get("/", function(req, res, err){
        "Hello World!"
      }) %>%
      serve_it()

    Serving the jug at http://127.0.0.1:8080

What is Jug?
------------

Jug is a webdevelopment framework for R which relies heavily upon the
`httpuv` package.

Jug is not supposed to be either an especially performant nor an uber
stable framework. Other tools (and languages) might be more suited for
that. Nevertheless, it tries to make available a set of functions which
allow you to easily create APIs for your R code. The flexibility of Jug
means that, in theory, you could built an extensive web framework with
it (but I don't especially recommend it).

The basics
----------

### The Jug instance

Everything starts with a Jug instance. This instance is created by
simply calling `jug()`:

    library(jug)
    jug()

    ## [1] "A Jug instance with 0 middlewares attached"

Jug is made to work closely with the piping functionality of `magrittr`
(`%>%`). The rest of the Jug instance is built up by piping the instance
through various functions explained below.

### Middleware

In terms of middleware Jug somewhat follows the specification of
middleware by `Express`. In Jug, middleware is a function with access to
the **request** (`req`), **response** (`res`) and **error** (`err`)
object.

Multiple middlewares can be defined. Order in which the middlewares are
added matters. A request will start with being passed through the first
middleware added (more specifically the `func` specified in it - see
next paragraph). It will continue to be passed through the added
middlewares until a middleware does not return `NULL`. Whatever will be
passed by the middleware will be set as the response body.

Most middleware will accept a `func` argument to which a function should
be passed. To this function the `req`, `res` and `err` objects wwill be
passed (and thus should accept them). The result of evaluation the
specified function will be used as the response body.

#### Method insensitive middleware

The `use` function is a method insensitive middleware specifier. While
it is method insensitive, it can be bound to a specific path. If the
`path` argument (accepts a regex string with `grepl` setting
`perl=TRUE`) is set to `NULL` it also becomes path insensitive and will
process *every* request.

A path insensitive example:

    jug() %>%
      use(path = NULL, function(req, res, err){
        "test 1,2,3!"
        }) %>%
      serve_it()

    $ curl 127.0.0.1:8080/xyz
    test 1,2,3!

The same example, but path sensitive:

    jug() %>%
      use(path = "/", function(req, res, err){
        "test 1,2,3!"
        }) %>%
      serve_it()

    $ curl 127.0.0.1:8080/xyz
    curl: (52) Empty reply from server

    $ curl 127.0.0.1:8080
    test 1,2,3!

Note that in the above error / missing route handling is missing, more
on that later.

#### Method sensitive middleware

In the same style ass the request method insensitive middleware, there
is request method sensitive middleware available. More specifically, you
can use the `get`, `post`, `put` and `delete` functions.

This type of middleware is bound to a path using the `path` argument. If
`path` is set to `NULL` it will bind to every request to the path, given
that it is of the corresponding request type.

    jug() %>%
      get(path = "/", function(req, res, err){
        "get test 1,2,3!"
        }) %>%
      serve_it()

    $ curl 127.0.0.1:8080
    get test 1,2,3!

Middlewares are meant to be chained, so to bind different functions to
different paths:

    jug() %>%
      get(path = "/", function(req, res, err){
        "get test 1,2,3 on path /"
        }) %>%
      get(path = "/my_path", function(req, res, err){
        "get test 1,2,3 on path /my_path"
        }) %>%
      serve_it()

    $ curl 127.0.0.1:8080
    get test 1,2,3 on path /

    $ curl 127.0.0.1:8080/my_path
    get test 1,2,3 on path /my_path

### Predefined middleware

#### Error handling

A simple error handling middleware (`simple_error_handler`) which
catches unbound paths and `func` evaluation errors. If no custom error
handler is implemented I suggest you add this to your Jug instance.

    jug() %>%
      simple_error_handler() %>%
      serve_it()

    $ curl 127.0.0.1:8080
    <!DOCTYPE html>
    <html lang="en">
      <head>
        <meta charset="utf-8">
        <title>Not found</title>
      </head>
      <body>
        <p>No handler bound to path</p>
      </body>
    </html>

If you want to implement your own custom error handling just have a look
at the `jug::simple_error_handler()` function.

### Easily using your own functions

The main reason Jug was created is to easily allow access to your own
custom R functions. The convenience function `decorate` is built with
this in mind.

If you `decorate` your own function it will translate all arguments
passed in the query string of the request as arguments to your function.
It will also pass all headers to the function as arguments. Note that
for the passed headers they will be capitalized and prefixed by `HTTP_`
as per `httpuv`'s internals.

If your function does not accept a `...` argument, all query/header
parameters that are nor explicitly requested by your function are
dropped. If your function requests a `req`, `res` or `err` argument (or
`...`) the corresponding objects will be passed.

    say_hello<-function(name){paste("hello",name,"!")}

    jug() %>%
      get("/", decorate(say_hello)) %>%
      serve_it()

    $ curl 127.0.0.1:8080/?name=Bart
    hello Bart !

### The request, response and error objects

#### Request (`req`) object

The `req` object contains the request specifications. It has different
attributes:

-   `req$query_params` a list of the query parameters contained in the
    URL
-   `req$path` the requested path
-   `req$method` the request method
-   `req$raw` the raw request object as passsed by `httpuv`
-   `req$post_data` the post data (has some limitations for now...)

#### Response (`res`) object

The `res` object contains the response specifications. It has different
attributes:

-   `res$headers` a named list of the set headers
-   `res$status` the status of the response (defaults to 200)
-   `res$body` the body of the response (is automatically set by the
    content of the not `NULL` returning middleware)

It also has a set of functions:

-   `res$set_header(key, value)` set a custom header
-   `res$content_type(type)` set your own content type
-   `res$set_status(status)` set the status of the response
-   `res$set_body(body)` set the body of the response

#### Error (`err`) object

The `err` object contains a list of errors, accessible through
`err$errrors`. You can add an error to this list by calling
`err$set(error)`. The error will be converted to a character.

Have a look at the `simple_error_handler` middleware function see how an
error handler is implemented.

### Serving the Jug

Simple call `serve_it()` at the end of your piping chain (see Hello
World! example).
