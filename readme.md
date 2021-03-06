# Eloquent OAuth

[![Code Climate](https://codeclimate.com/github/adamwathan/eloquent-oauth/badges/gpa.svg)](https://codeclimate.com/github/adamwathan/eloquent-oauth)

Eloquent OAuth is a package for Laravel 4 designed to make authentication against various OAuth providers *ridiculously* brain-dead simple. Specify your app keys/secrets in a config file, run a migration and from then on it's just two method calls and you have OAuth integration.

## Usage

Authentication against an OAuth provider is a multi-step process, but I have tried to simplify it as much as possible.

### Authorizing with the provider

First you will need to define the authorization route. This is the route that your "Login" button will point to, and this route redirects the user to the provider's domain to authorize your app. After authorization, the provider will redirect the user back to your second route, which handles the rest of the authentication process.

To authorize the user, simply return the `OAuth::authorize()` method directly from the route.

```php
Route::get('facebook/authorize', function() {
    return OAuth::authorize('facebook');
});
```

### Authenticating within your app

Next you need to define a route for authenticating against your app with the details returned by the provider.

For basic cases, you can simply call `OAuth::login()` with the provider name you are authenticating with. If the user
rejected your application, this method will throw an `ApplicationRejectedException` which you can catch and handle
as necessary.

The `login` method will create a new user if necessary, or update an existing user if they have already used your application
before.

Once the `login` method succeeds, the user will be authenticated and available via `Auth::user()` just like if they
had logged in through your application normally.

```php
use \AdamWathan\EloquentOAuth\ApplicationRejectedException;
use \AdamWathan\EloquentOAuth\InvalidAuthorizationCodeException;

Route::get('facebook/login', function() {
    try {
        OAuth::login('facebook');
    } catch (ApplicationRejectedException $e) {
        // User rejected application
    } catch (InvalidAuthorizationCodeException $e) {
        // Authorization was attempted with invalid
        // code,likely forgery attempt
    }

    // Current user is now available via Auth facade
    $user = Auth::user();

    return Redirect::intended();
});
```

If you need to do anything with the newly created user, you can pass an optional closure as the second
argument to the `login` method. This closure will receive the `$user` instance and a `ProviderUserDetails`
object that contains basic information from the OAuth provider, including:

- User ID
- Nickname
- First Name
- Last Name
- Email
- Image URL
- Access Token

```php
OAuth::login('facebook', function($user, $details) {
    $user->nickname = $details->nickname;
    $user->name = $details->firstName . ' ' . $details->lastName;
    $user->profile_image = $details->imageUrl;
    $user->save();
});
```

> Note: The Instagram API does not allow you to retrieve the user's email address, so unfortunately that field will always be `null` for the Instagram provider.

## Supported Providers

- Facebook
- GitHub
- Google
- LinkedIn
- Instagram

>The package is still in it's early infancy obviously. Support will be added for other providers as time goes on.

>Feel free to open an issue if you would like support for a particular provider, or even better, submit a pull request.

## Installation

Require this package in your `composer.json` file to install via Packagist:

`"adamwathan/eloquent-oauth": "dev-master"`

...then run `composer update` to download the package to your vendor directory.

Add the service provider to the `providers` array in `app/config/app.php`:

```php
'providers' => array(
    // ...
    'AdamWathan\EloquentOAuth\EloquentOAuthServiceProvider',
    // ...
)
```

Add the facade to the `aliases` array in `app/config/app.php`:

```php
'aliases' => array(
    // ...
    'OAuth' => 'AdamWathan\EloquentOAuth\Facades\OAuth',
    // ...
)
```

Publish the configuration file:

`php artisan config:publish adamwathan/eloquent-oauth`

Update your app information for the providers you are using in `app/config/packages/adamwathan/eloquent-oauth/config.php`:

```php
'providers' => array(
    'facebook' => array(
        'id' => '12345678',
        'secret' => 'y0ur53cr374ppk3y',
        'redirect' => URL::to('facebook/login'),
        'scope' => array(),
    )
)
```
> Note: Each provider is preconfigured with the necessary scope to retrieve basic user information as well as the user's email address, so the scope array can usually be left empty unless you need specific additional permissions. Consult the provider's API documentation to find out what permissions are available for the various services.

If you need to change the name of the table used to store OAuth identities, you can do so in the same config file:

```php
'table' => 'social_login_tokens',
```

Publish and run the migration:

```
php artisan migrate:publish adamwathan/eloquent-oauth
php artisan migrate
```

All done!

## Notes

Eloquent OAuth is designed to integrate with Laravel's Eloquent authentication driver, so be sure you are using the `eloquent`
driver in `app/config/auth.php`. You can define your actual `User` model however you choose and add whatever behavior you need,
just be sure to specify the model you are using with its fully qualified namespace in `app/config/auth.php` as well.
