# sanitizer    

[![Build Status](https://travis-ci.org/kalfheim/sanitizer.svg?branch=master)](https://travis-ci.org/kalfheim/sanitizer)
[![StyleCI](https://styleci.io/repos/54579807/shield)](https://styleci.io/repos/54579807)
[![Code Coverage](https://scrutinizer-ci.com/g/kalfheim/sanitizer/badges/coverage.png?b=master)](https://scrutinizer-ci.com/g/kalfheim/sanitizer/?branch=master)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/kalfheim/sanitizer/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/kalfheim/sanitizer/?branch=master)
[![Latest Stable Version](https://poser.pugx.org/kalfheim/sanitizer/v/stable)](https://packagist.org/packages/kalfheim/sanitizer)
[![Latest Unstable Version](https://poser.pugx.org/kalfheim/sanitizer/v/unstable)](https://packagist.org/packages/kalfheim/sanitizer)
[![License](https://poser.pugx.org/kalfheim/sanitizer/license)](https://packagist.org/packages/kalfheim/sanitizer)

> Data sanitizer for PHP with built-in Laravel support.

## Installation

    composer require kalfheim/sanitizer

## Usage

``` php
use Alfheim\Sanitizer\Sanitizer;

// Create a new sanitizer instance by passing an array of rules to the `Sanitizer::make` method...
$sanitizer = Sanitizer::make([
    'email' => 'trim',
]);

// Simulate some user input...
$input = [
    'email' => 'name@example.com ', // Notice the space.
];

// Now we will sanitize some data by passing an array to the `sanitize` method...
var_dump($sanitizer->sanitize($input)); // ['email' => 'name@example.com']

// It is also possible to pass the data by reference using the `sanitizeByRef` method...
$sanitizer->sanitizeByRef($input);
var_dump($input); // ['email' => 'name@example.com']
```

### Examples

``` php
// Wildcard example...

$input = [
    'name'  => 'Ola nordmann',
    'email' => 'Name@example.com ',
];

$sanitizer = Sanitizer::make([
    '*'     => 'trim',          // `*` is a wildcard which will apply to all fields.
    'name'  => 'ucwords',       // Uppercase first char of each word in the name field.
    'email' => 'mb_strtolower', // Lowercase each letter in the email field.
]);

var_dump($sanitizer->sanitize($input));
// ['name' => 'Ola Nordmann', 'email' => 'name@example.com']
```

``` php
// Multiple rules and arguments...

$sanitizer = Sanitizer::make([
    'name'  => 'trim|ucwords', // Trim, then uppercase first char of each word.
    'email' => 'preg_replace:/\+\w+/::{{ VALUE }}',
]);

// The `email` rule might be a handful, but it is really quite simple. 
// The rule translates to `$sanitizedValue = preg_replace('/\+\w+/', '', $value)`.
// It will sanitize an email like `name+foo@example.com` to `name@example.com`.

// The `{{ VALUE }}` string is a magic constant that the sanitizer will replace
// with the value currently being sanitized.

// By default, the value will be implicitly bound to the first argument in the list,
// however, you can place it where ever you need to satisfy the function being called.

$sanitizer = Sanitizer::make([
    'foo' => 'mb_substr:0:1',
    'bar' => 'mb_substr:{{ VALUE }}:0:1',
]);

// In the example above, both rules will achieve the same end result.
```

## Registrars

A registrar allows you to bind custom sanitizer functions to the sanitizer.

``` php
use Alfheim\Sanitizer\Sanitizer;
use Alfheim\Sanitizer\Registrar\BaseRegistrar;

// Create a new registrar instance...
$registrar = new BaseRegistrar;

// Add custom sanitation rules to the registrar...
$registrar->register('palindromify', function (string $value) {
    return sprintf('%s%s', $value, strrev($value));
});

// Create a new sanitizer and bind the registrar...
$sanitizer = Sanitizer::make([
    'number' => 'palindromify',
])->setRegistrar($registrar);


$input = $sanitizer->sanitize([
    'number' => '123',
]);

var_dump($input); // ['number' => '123321']
```

## Laravel Support

Register the service provider in your `config/app.php` as per usual...

    Alfheim\Sanitizer\SanitizerServiceProvider::class,

### Extending the FormRequest

**This is where the package shines.**
By extending the `Alfheim\Sanitizer\Laravel\FormRequest` on your base `App\Http\Requests\Request` class (instead of the default `Illuminate\Foundation\Http\FormRequest`), you'll be able to define sanitation rules in a `sanitize` method on the given form request, similar to how you define validation rules in the `rules` method.

Let me show you in code...

``` php
// app/Http/Requests/Request.php

namespace App\Http\Requests;

use Alfheim\Sanitizer\Laravel\FormRequest;
// Instead of `Illuminate\Foundation\Http\FormRequest`

abstract class Request extends FormRequest
{
    //
}
```

That's it! Now it's trivial to define sanitation rules on your form requests...

``` php
// app/Http/Requests/FooRequest.php

namespace App\Http\Requests;

class FooRequest extends Request
{
    // Sanitation rules...
    public function sanitize()
    {
        return [
            'name'  => 'trim|ucwords',
            'email' => 'trim|mb_strtolower',
        ];
    }

    // And of course, validation is defined as per usual...
    public function rules()
    {
        return [
            'name'  => 'required',
            'email' => 'required|email',
        ];
    }
}
```

For completeness, I'll show you the controller...

``` php
namespace App\Http\Controllers;

use App\Http\Requests\FooRequest;

class FooController extends Controller
{
    public function create(FooRequest $request)
    {
        // At this point, the $request will be both sanitized and validated.
        // You may go ahead and access the input as usual:

        $request->all();
        $request->input('name');
        $request->only(['name', 'email']);
        // etc...
    }
}
```

### Helper Trait

#### `Alfheim\Sanitizer\Laravel\SanitizesRequests`

This trait adds a `sanitize` method on the class.
May be useful if you want to sanitize user input in a controller without setting up a custom request class (however, it _can_ be used from anywhere.)

``` php
public function sanitize(Illuminate\Http\Request $request, array $ruleset): array
```

Example usage...

``` php
namespace App\Http\Controllers\FooController;

use Illuminate\Http\Request;
use Alfheim\Sanitizer\Laravel\SanitizesRequests;

class FooController extends Controller
{
    use SanitizesRequests;

    public function store(Request $request)
    {
        $input = $this->sanitize($request, [
            'name'  => 'trim|ucwords',
            'email' => 'trim|mb_strtolower',
        ]);

        // $input now contains the sanitized request input.
    }
}
```

### Registering custom sanitizer functions with Laravel

The service provider will register a shared `Alfheim\Sanitizer\Registrar\RegsitrarInterface` instance with the IoC container, which will then be set on subsequent `Alfheim\Sanitizer\Sanitizer` instances. This means you can easily register custom sanitizer functions...

``` php
use Alfheim\Sanitizer\Registrar\RegistrarInterface;

// Standalone...
app(RegistrarInterface::class)->register('yell', $callable);

// In a method resolved by the container, perhaps a service provider...
public function registerSanitizers(RegistrarInterface $registrar)
{
    $registrar->register('yell', function (string $value) {
        return mb_strtoupper($value);
    });
}

// You may also resolve an object from the IoC container using `class@method` notation...
app(RegistrarInterface::class)->register('foo', 'some.service@sanitizerMethod');
```

## License

[MIT](LICENSE) &copy; [Kristoffer Alfheim](http://github.com/kalfheim)
