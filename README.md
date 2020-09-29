ApiGuard
========

# History

This package is a fork of [chrisbjr/api-guard](https://github.com/chrisbjr/api-guard) designed to provide some forward support into Laravel 6 and 7, to help people who want to migrate to Laravel Sanctum.

If you're thinking of downloading it for the first time in the year 2020, please don't.

Laravel already has a similar feature built-in since Laravel 5.8 and Laravel 6.x. Documentation on Laravel's API authentication can be found [here](https://laravel.com/docs/5.8/api-authentication). Furthermore, since Laravel 7.x, there is a new Laravel package called [Airlock/Sanctum](https://laravel.com/docs/7.x/sanctum) that would better serve API authentication purposes.

A simple way of authenticating your APIs with API keys using Laravel. This package uses the following libraries:

- philsturgeon's [Fractal](https://github.com/thephpleague/fractal)
- maximebeaudoin's [api-response](https://github.com/ellipsesynergie/api-response)

## Support Matrix

| Laravel Version  | api-guard Version |
| ------------- | ------------- |
| 6.x | `^6.*` |
| 5.3 and up  | `~4.*` |
| 5.1.x to 5.2.x  | `~3.*` |

## Quick start

### Installation 

Run `composer require JWadhams/api-guard`

In your `config/app.php` add `JWadhams\ApiGuard\Providers\ApiGuardServiceProvider` to the end of the `providers` array

```php
'providers' => array(

    ...
    JWadhams\ApiGuard\Providers\ApiGuardServiceProvider::class,
),
```

Now publish the migration and configuration files for api-guard:

    $ php artisan vendor:publish --provider="JWadhams\ApiGuard\Providers\ApiGuardServiceProvider"

Then run the migration:

    $ php artisan migrate

It will setup  `api_keys` table.

### Generating your first API key

Once you're done with the required setup, you can now generate your first API key.

Run the following command to generate an API key:

`php artisan api-key:generate`

Generally, the `ApiKey` object is a polymorphic object meaning this can belong to more than one other model.

To generate an API key that is linked to another object (a "user", for example), you can do the following:

+`php artisan api-key:generate --id=1 --type="App\User"`

To specify that a model can have API keys, you can attach the `Apikeyable` trait to the model:

```php
use JWadhams\ApiGuard\Models\Mixins\Apikeyable;

class User extends Model
{
    use Apikeyable;

    ...
}

```

This will attach the following methods to the model:

```php
// Get the API keys of the object
$user->apiKeys();

// Create an API key for the object
$user->createApiKey();
```

To generate an API key from within your application, you can use the following method in the `ApiKey` model:

```php
$apiKey = JWadhams\ApiGuard\Models\ApiKey::make()

// Attach a model to the API key
$apiKey = JWadhams\ApiGuard\Models\ApiKey::make($model)
```

## Usage

You can start using ApiGuard by simply attaching the `auth.apikey` middleware to your API route:

```php
Route::middleware(['auth.apikey'])->get('/test', function (Request $request) {
    return $request->user(); // Returns the associated model to the API key
});
```

This effectively secures your API with an API key which needs to specified in the `X-Authorization` header. This can be configured in `config/apiguard.php`.

Here is a sample cURL command to demonstrate:

```
curl -X GET \
  http://apiguard.dev/api/test \
  -H 'x-authorization: api-key-here'
```

You might also want to attach this middleware to your `api` middleware group in your `app/Http/Kernel.php` to take advantage of other Laravel features such as
throttling.

```php
/**
 * The application's route middleware groups.
 *
 * @var array
 */
protected $middlewareGroups = [
    ...

    'api' => [
        'throttle:60,1',
        'bindings',
        'auth.apikey',
    ],
];
```

If you noticed in the basic example, you can also access the attached model to the API key by calling `$request->user()`. We are attaching the related model in
this method because in most use cases, this is actually the user.

### Unauthorized Requests

Unauthorized requests will get a `401` status response with the following JSON:

```json
{
  "error": {
    "code": "401",
    "http_code": "GEN-UNAUTHORIZED",
    "message": "Unauthorized."
  }
}
```

### ApiGuardController

The `ApiGuardController` takes advantage of [Fractal](http://fractal.thephpleague.com/) and [api-response](https://github.com/ellipsesynergie/api-response) libraries.

This enables us to easily create APIs with models and use transformers to give a standardized JSON response.

Here is an example:

Let's say you have the following model:

```php
use Illuminate\Database\Eloquent\Model;

class Book extends Model
{
    protected $fillable = [
        'name',
    ];
}
```

You can make a basic controller which will return all books like this:

```php
use JWadhams\ApiGuard\Http\Controllers\ApiGuardController;
use App\Transformers\BookTransformer;
use App\Book;

class BooksController extends ApiGuardController
{
    public function all()
    {
        $books = Book::all();

        return $this->response->withCollection($books, new BookTransformer);
    }
}
```

Now, you'll need to make the transformer for your Book object. Transformers help with defining and manipulating the variables you want to return to your JSON response.

```php
use League\Fractal\TransformerAbstract;
use App\Book;

class BookTransformer extends TransformerAbstract
{
    public function transform(Book $book)
    {
        return [
            'id'         => $book->id,
            'name'       => $book->name,
            'created_at' => $book->created_at,
            'updated_at' => $book->updated_at,
        ];
    }
}
```

Once you have this accessible in your routes, you will get the following response from the controller:

```json
{
  "data": {
    "id": 1,
    "title": "The Great Adventures of Chris",
    "created_at": {
      "date": "2017-05-25 18:54:18",
      "timezone_type": 3,
      "timezone": "UTC"
    },
    "updated_at": {
      "date": "2017-05-25 18:54:18",
      "timezone_type": 3,
      "timezone": "UTC"
    }
  }
}
```

More examples can be found on the Github page: [https://github.com/ellipsesynergie/api-response](https://github.com/ellipsesynergie/api-response).

To learn more about transformers, visit the PHP League's documentation on Fractal: [Fractal](http://fractal.thephpleague.com/)

### API Validation Responses

ApiGuard comes with a request class that can handle validation of requests for you and throw a standard response.

You can create a `Request` class as you usually do but in order to get a standard JSON response you'll have to extend the `ApiGuardFormRequest` class.

```php
use JWadhams\ApiGuard\Http\Requests\ApiGuardFormRequest;

class BookStoreRequest extends ApiGuardFormRequest
{
    public function authorize()
    {
        return true;
    }

    public function rules()
    {
        return [
            'name' => 'required',
        ];
    }
}
```

Now you can use this in your controller as you normally do with Laravel:

```php
use JWadhams\ApiGuard\Http\Controllers\ApiGuardController;
use App\Transformers\BookTransformer;
use App\Book;

class BooksController extends ApiGuardController
{
    public function store(BookStoreRequest $request)
    {
        // Request should already be validated

        $book = Book::create($request->all())

        return $this->response->withItem($book, new BookTransformer);
    }
}
```

If the request failed to pass the validation rules, it will return with a response like the following:

```json
{
  "error": {
    "code": "GEN-UNPROCESSABLE",
    "http_code": 422,
    "message": {
      "name": [
        "The name field is required."
      ]
    }
  }
}
```
