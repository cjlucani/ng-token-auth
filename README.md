# Angular Token Auth

This module aims to provide a simple method of client authentication that can be configured to work with any api.

This module was designed to work out of the box with the outstanding [devise token auth](https://github.com/lynndylanhurley/devise_token_auth) gem, but I've been able to use it in other environments as well ([go](http://golang.org/), [gorm](https://github.com/jinzhu/gorm) and [gomniauth](https://github.com/stretchr/gomniauth) for example). 

Token based authentication requires coordination between the client and the server. Diagrams are included to illustrate this relationship.


# Installation

* `bower install ng-token-auth --save`
* include `ng-token-auth` in your app.

#### Example
~~~javascript
angular.module('myApp'), ['ng-token-auth'])
~~~

Your controllers, directives, filters etc. will now be able to inject the `$auth` service, and your config block will be able to inject the `$authProvider` provider for configuration.

## Configuration
### $authProvider.configure

The `$authProvider` is available for injection during the app's configuration phase. Configure this module for use with the API server using a `config` block. [Read more about configuring providers](https://github.com/angular/angular.js/wiki/Understanding-Dependency-Injection#configuring-providers)

### Config example
~~~javascript
angular.module('myApp'), ['ng-token-auth'])

	.config(function($authProvider) {

    // the following shows the default values. values passed to this method
    // will extend the defaults using angular.extend

		$authProvider.configure({
			apiUrl:                 '/api',
			tokenValidationPath:    '/auth/validate_token',
			signOutUrl:             '/auth/sign_out',
			emailRegistrationPath:  '/auth',
			confirmationSuccessUrl: window.location.href,
			emailSignInPath:        '/auth/sign_in',
			proxyIf:                function() { return false; },
			proxyUrl:               '/proxy',
			authProviderPaths: {
        		github:   '/auth/github',
        		facebook: '/auth/facebook',
        		google:   '/auth/google'
			}

		});
	});
~~~

### Config params

* **apiUrl**: the base route to your api. Each of the following paths will be relative to this URL.
* **authProviderPaths**: an object containing paths to auth endpoints. keys are names of the providers, values are their auth paths relative to the `apiUrl`.
* **tokenValidationPath**: relative path to validate authentication tokens.
* **signOutUrl**: relative path to sign user out. this will destroy the user's token both server-side and client-side.
* **emailRegistrationPath**: path for submitting new email registrations.
* **confirmationSuccessUrl**: this value is passed to the API for email registration. I use it to redirect after email registration, but that can also be set server-side or ignored. this is useful when working with APIs that have multiple client domains.
* **emailSignInPath**: path for signing in using email credentials.
* **proxyIf**: older browsers have trouble with CORS. pass a method here to determine whether or not a proxy should be used. example: `function() { return !Modernizr.cors }`
* **proxyUrl**: proxy url if proxy is to be used


These settings correspond to the paths that are available when using the [devise token auth](https://github.com/lynndylanhurley/devise_token_auth#usage) gem for Rails. If you're using the gem, just set the `apiUrl` to your server's base auth route.

##### Example configuration when using devise token auth
~~~javascript
angular.module('myApp'), ['ng-token-auth'])
	.config(function($authProvider) {
		$authProvider.configure({
			apiUrl: 'http://api.example.com/auth'
		});
	});
~~~

# Usage

**TODO**: add TLDR;

## Oauth2 authentication

### Conceptual

The following diagram illustrates the steps necessary to authenticate a client using an oauth2 provider.

![oauth flow](https://github.com/lynndylanhurley/ng-token-auth/raw/master/test/app/images/flow/omniauth-flow.jpg)

When authenticating with a 3rd party provider, the following steps will take place.

1. An external window will be opened to the provider's authentication page. 
1. Once the user signs in, they will be redirected back to the API at the callback uri that was registered with the oauth2 provider.
1. The API will send the user's info back to the client via `postMessage` event, and then close the external window.

The postMessage event must include the following a parameters:
* **message** - this must contain the value `"deliverCredentials"`
* **auth_token** - a unique token set by your server.
* **uid** - the id that was returned by the provider. For example, the user's facebook id, twitter id, etc.

Rails example: [controller](https://github.com/lynndylanhurley/ng-token-auth-api-rails/blob/master/app/controllers/users/auth_controller.rb#L21), [layout](https://github.com/lynndylanhurley/ng-token-auth-api-rails/blob/master/app/views/layouts/oauth_response.html.erb), [view](https://github.com/lynndylanhurley/ng-token-auth-api-rails/blob/master/app/views/users/auth/oauth_success.html.erb).

#### Example redirect_uri destination:

~~~html
<!DOCTYPE html>
<html>
  <head>
    <script>
      window.addEventListener("message", function(ev) {

        // this page must respond to "requestCredentials"
        if (ev.data === "requestCredentials") {

          ev.source.postMessage({
             message: "deliverCredentials", // required
             auth_token: 'xxxx', // required
             uid: 'yyyy', // required

             // additional params will be added to the user object
             name: 'Slemp Diggler'
             // etc.

          }, '*');

          // close window after message is sent
          window.close();
        }
      });
    </script>
  </head>
  <body>
    <pre>
      Redirecting...
    </pre>
  </body>
</html>
~~~

### Oauth2 Service methods

The `$auth` service is available for injection during the app's run phase.

### $auth.authenticate

The `$auth.authenticate` method is used to authenticate using an oauth2 provider. This method takes a single argument, a string containing the name of the provider. This method opens an external window to the corresponding `authProviderPaths` value that was set in the config.  This method is also attached to the `$rootScope` for use in templates.

#### Example use in a controller
~~~javascript
angular.module('ngTokenAuthTestApp')
	.controller('IndexCtrl', function($auth) {

		$auth.authenticate('github')

	});
~~~

#### Example use in a template
~~~html
<button ng-click="authenticate('github')">
  Sign in with Github
</button>
~~~

## Token validation

The client's tokens are stored in cookies using the ngCookie module. This is done so that users won't need to re-authenticate each time they return to the site or refresh the page.

### Conceptual

![validation flow](https://github.com/lynndylanhurley/ng-token-auth/raw/master/test/app/images/flow/validation-flow.jpg)

Rails example [here](https://github.com/lynndylanhurley/ng-token-auth-api-rails/blob/master/app/controllers/users/auth_controller.rb#L5)

### $auth.validateUser

`$auth.validateUser()` is called on page load during the app's run phase.

This method returns a `$q` promise. These promises can be used to prevent users from viewing certain pages when using angular ui router resolvers.

#### Example

~~~coffeescript
angular.module('ngTokenAuthTestApp', [
  'ui.router',
  'ng-token-auth'
])
  .config(function($stateProvider) {
    $stateProvider
      // this state will be visible to everyone
      .state('index', {
        url: '/',
        templateUrl: 'index.html',
        controller: 'IndexCtrl'
      })

      // only authenticated users will be able to see routes that are
      // children of this state
      .state('admin', {
        url: '/admin',
        abstract: true,
        resolve: {
          auth: function($auth) {
            return $auth.validateUser();
          }
        }
      })

      // this route will only be available to authenticated users
      .state('admin.dashboard', {
        url: '/dash',
        templateUrl: '/admin/dash.html',
        controller: 'AdminDashCtrl'
      });
  });
~~~

Note that this is not secure, and that any access to any restricted content should be limited by the server as well.

## Email registration

This module also provides support for email registration. The following diagram illustrates this process.

![email registration flow](https://github.com/lynndylanhurley/ng-token-auth/raw/master/test/app/images/flow/email-registration-flow.jpg)

### $auth.submitRegistration

Users can be registered by email using the `$auth.submitRegistration` method. This method accepts an object with the following params.

* **email**
* **password**
* **password_confirmation**

The `$auth.submitRegistration` method is available to the `$rootScope`.

#### Example
~~~html
<form ng-submit="submitRegistration(registrationForm)" role="form" ng-init="registrationForm = {}">
  <div class="form-group">
    <label>email</label>
    <input type="email" name="email" ng-model="registrationForm.email" required="required" class="form-control"/>
  </div>

  <div class="form-group">
    <label>password</label>
    <input type="password" name="password" ng-model="registrationForm.password" required="required" class="form-control"/>
  </div>

  <div class="form-group">
    <label>password confirmation</label>
    <input type="password" name="password_confirmation" ng-model="registrationForm.password_confirmation" required="required" class="form-control"/>
  </div>

  <button type="submit" class="btn btn-primary btn-lg">Register</button>
</form>
~~~

## Email sign in

### Conceptual

![email sign in flow](https://github.com/lynndylanhurley/ng-token-auth/raw/master/test/app/images/flow/email-sign-in-flow.jpg)

### $auth.submitLogin

Once a user has completed email registration, they will be able to sign in using the `$auth.submitLogin` method. This method accepts an object with the following params.

* **email**
* **password**

The `$auth.submitRegistration` method is available to the `$rootScope`.

#### Example

~~~html
<form ng-submit="submitLogin(loginForm)" role="form" ng-init="loginForm = {}">
  <div class="form-group">
    <label>email</label>
    <input type="email" name="email" ng-model="loginForm.email" required="required" class="form-control"/>
  </div>

  <div class="form-group">
    <label>password</label>
    <input type="password" name="password" ng-model="loginForm.password" required="required" class="form-control"/>
  </div>

  <button type="submit" class="btn btn-primary btn-lg">Sign in</button>
</form>
~~~

## Identifying users on the server.

The user's authentication information is included by the client in the `Authorization` header of each request. If you're using the [devise token auth](https://github.com/lynndylanhurley/devise_token_auth) gem, the header must follow this format:

~~~
token=xxxxx uid=yyyyy
~~~

Replace `xxxxx` with the user's `auth_token` and `yyyyy` with the user's `uid`.

This will all happen by default when using this module.

**Note**: If you require a different authorization header format, post an issue. I will make it a configuration option if there is a demand.


## Proxy CORS requests
Shit browsers (IE8, IE9) have trouble with CORS requests. You will need to set up a proxy to support them.

##### Example proxy using express for node.js
~~~javascript
var express   = require('express');
var request   = require('request');
var httpProxy = require('http-proxy');
var CONFIG    = require('config');

// proxy api requests (for older IE browsers)
app.all('/proxy/*', function(req, res, next) {
  // transform request URL into remote URL
  var apiUrl = 'http:'+CONFIG.API_URL+req.params[0];
  var r = null;

  // preserve GET params
  if (req._parsedUrl.search) {
    apiUrl += req._parsedUrl.search;
  }

  // handle POST / PUT
  if (req.method === 'POST' || req.method === 'PUT') {
    r = request[req.method.toLowerCase()]({
      uri: apiUrl, 
      json: req.body
    });
  } else {
    r = request(apiUrl);
  }

  // pipe request to remote API
  req.pipe(r).pipe(res);
});
~~~

The above example assumes that you're using [express](http://expressjs.com/), [request](https://github.com/mikeal/request), and [http-proxy](https://github.com/nodejitsu/node-http-proxy), and that you have set the API_URL value using [node-config](https://github.com/lorenwest/node-config).

---

## Development

There is a test project in the `test` directory of this app. To start a dev server, perform the following steps.

1. `cd` to the root of this project.
1. `npm install`
1. `cd test && bundle install`
1. `cd ..`
1. `gulp dev`

A dev server will start on [localhost:7777](http://localhost:7777).

This module was built against [this API](https://github.com/lynndylanhurley/ng-token-auth-api-rails). You can use this, or feel free to use your own.

There are more detailed instructions in `test/README.md`.

## Contributing

Just send a pull request. You will be granted commit access if you send quality pull requests.

Guidelines will be posted if the need arises.

## TODO

* Tests. This will be difficult because test will require both an API and an oauth2 provider. Please open an issue if you have any suggestions.
* Example site coming soon.

## License

This project uses the WTFPL

