---
layout: post
title: "AngularJs Unit Testing: ui-router"
quote: "Test your application routing with jasmine"
video: false
comments: true
theme_color: 302F2D
tags: [angularjs, jasmine, unit-test, ui-router]
---

#Unit Testing ui-router with Jasmine

I started to realize that it's important to test a router since I began to put a considerably amount of code in resolve block and controller required these resolves to function. This means it is crucial to make sure that resolves are resolved and work correctly. It's also a good idea to make sure the configuration for each state is correct.


At first, this seems to be a daunting task. I didn't even know how to start. Thanks to [this post](http://nikas.praninskas.com/angular/2014/09/27/unit-testing-ui-router-configuration/). It gave me an idea how to write a test for my application. However, there are still some parts to figure out since my ui-router is a lot more complicate than the example.


Basically, I have a role-based access application. Users with different roles have access to different page(state) of the application. It is controlled by the permission assigned to each state which can be either 'admin' or 'user'. This permission will be validated when transition between state happen by watching for **'$stateChangeStart'** event. It will check for permission of the toState and redirect to error page if not allowed. There are also a significant amount of code in the resolves of many states. Mostly load required files with $ocLazyLoad and call other services in the application.


To get a better idea, this is part of my ui-router config.


{% highlight javascript %}
angular.module('app.routing')
  .run(
    function($rootScope, $state, $stateParams, securityService) {
      $rootScope.$on("$stateChangeStart", function(event, toState, toParams, fromState, fromParams) {
        if (!toState.data || !toState.data.permissions) {
          return true;
        }

        if (!securityService.isAuthenticated() && toState.data.permissions) {
          event.preventDefault();
          return $state.go('login');
        }

        if (!securityService.isAuthorized(toState.data.permissions)) {
          event.preventDefault();
          return $state.go('error404');
        }

        return true;
      });
    })
  .config(
    function($stateProvider, $urlRouterProvider, permission) {
      $stateProvider
        .state('login', {
          url: '/login',
          templateUrl: 'login.html',
          resolve: {
            deps: function($ocLazyLoad) {
              return $ocLazyLoad.load('loginCtrl.js');
            }
          }
        })
        .state('error404', {
          url: '^/error404',
          templateUrl: 'error404.html'
        })

        // Authorized pages
        .state('app', {
          abstract: true,
          url: '/',
          templateUrl: 'layout.html',
          resolve: {
            deps: function($ocLazyLoad) {
              return $ocLazyLoad.load([...]);
            }
          }
        })
        .state('app.product', {
          abstract: false,
          url: '^/product',
          templateUrl: 'product.html',
          controller: 'ProductCtrl',
          data: {
            permissions: permission.user
          },
          resolve: {
            products: function($ocLazyLoad, $injector) {
              return $ocLazyLoad.load('productService.js')
                .then( function() {
                  var productService = $injector.get('productService');
                  return productService.getAll();
                })
            },
            deps: function($ocLazyLoad) {
              return $ocLazyLoad.load([...]);
            }
          }
        })
        .state('app.product.new', {
          abstract: false,
          url: '/new',
          templateUrl: 'createProduct.html',
          controller: 'CreateProductCtrl',
          data: {
            permissions: permission.admin
          },
          resolve: {
            deps: function($ocLazyLoad) {
              return $ocLazyLoad.load([...]);
            }
          }
        })
    });
{% endhighlight %}
 
 
Here, I have 'login' and 'error404' states that anyone can access and 'app.product' and 'app.product.new' for authenticated users with authorization as user and admin respectively. 

To begin, let's think about what should be test in this router

 - Verify anyone can access 'login' and config is correct
 - Verify authenticated users(user or admin) can access 'app.product', config is correct, and resolve is resolved
 - Verify users with admin role can access 'app.product.new' and config is correct
 - Verify users with user role cannot access 'app.product.new' and will be redirect to 'error404'
              

This can achieve these with a few simple steps. Most of the work will be about setting up test and mocking services(securityService, productService).


First, we can change state or navigate to other pages by using $state or $location. We also need to trigger a digest cycle with $rootScope.$digest() to revole promises in resolve block in state config.

{% highlight javascript %}
$location.url('/product');
$rootScope.$digest();
{% endhighlight %}

Next is template. We can use $templateCache to load the template for each state before running test.

{% highlight javascript %}
$templateCache.put('login.html', 'This is the template for login');
{% endhighlight %}

If we do not load templates into $templateCache, angular will try to get a template from the server by using $http and it will throw an error - Unexpected $http.GET('...').

We need to do this for every template we are going to use and that sounds like we are going to have a lot of duplicate line of code to just load the templates. Luckily, there is a better way with [karma-ng-html2js-preprocessor](https://github.com/karma-runner/karma-ng-html2js-preprocessor). This is a plugin to covert html files to angular templates automatically. Just follow the document and load the module that contains your templates and we are done.

Here is what it's going to look like for 'login'.

{% highlight javascript %}
it('should go to /login', function() {
  $location.url('/login');
  $rootScope.$digest();

  var currentState = $state.current;
  expect(currentState.name).toEqual('login');
  expect(currentState.url).toEqual('/login');
  expect(currentState.templateUrl).toEqual('login.html');
});
{% endhighlight %}

The login is pretty easy. Let's see another test for 'app.product'. 

The parent state of 'app.product' is 'app' which is an abstract state that has a resolve named 'deps'. The 'deps' will load all required files with [$ocLazyLoad](https://oclazyload.readme.io/docs/with-your-router). Since we don't want to use the actual service, the easiest way to mock $ocLazyLoad would be to inject $ocLazyLoad and spy on 'load' method and return promise.

{% highlight javascript %}
var $ocLazyLoad;
inject(function(_$ocLazyLoad_) {
  $ocLazyLoad = _$ocLazyLoad_;
});

spyOn($ocLazyLoad, 'load').and.callFake(function(args) {
  return $q.when('resolved');
});
{% endhighlight %}

The app.product state has 'products' resolve which will call productService to get the data. Since, we already spied $ocLazyLoad, we only need to create mock objects for productService and securityService and tell angular to use our mocks.

{% highlight javascript %}
var mockSecurity = {}, mockProductService = {};
module(function($provide) {
  $provide.value('securityService', mockSecurity);  
  $provide.value('productService', mockProductService);
});
{% endhighlight %}

Then, we need to fake the implementation of productService.getAll() which is actually going to call $http to get a list of products and return a promise. This can be done by using jasmine createSpy() and $q to create a promise.

{% highlight javascript %}
var $q;
inject(function(_$q_){
    $q = _$q_;
});

mockProductService.getAll = jasmine.createSpy().and.callFake(function() {
    return $q.when([{productId:1}, {productId:2}]);
}); 
{% endhighlight %}

One last thing to handle is permission and securityService. There are two methods in securityService: isAuthorized and isAuthenticated. We will use 'userRole' variable to keep the role of test user.

{% highlight javascript %}
// Check the user's role against state's permission
var userRole = 'admin'
mockSecurity.isAuthorized = jasmine.createSpy().and.callFake(function(permission) {
  return permission.indexOf(userRole) >= 0 ? true : false;
});

// Check if user is logged in or not
mockSecurity.isAuthenticated = jasmine.createSpy().and.returnValue(true);
{% endhighlight %}

With all the above setup and mocks, we can now test 'app.product' routing and verify config and resolve.

{% highlight javascript %}
it('should go to app.product', function() {
  goTo('/products');

  var currentState = $state.current;
  expect(currentState.name).toEqual('app.product');
  expect(currentState.url).toEqual('^/products');
  expect(currentState.templateUrl).toEqual('products.html');
  expect(currentState.data.permissions).toEqual(permission.user);

  var resolvedSpy = jasmine.createSpy();
  $injector.invoke(currentState.resolve['products']).then(resolvedSpy);
  $rootScope.$digest();
  expect(resolvedSpy).toHaveBeenCalledWith([]);
});
{% endhighlight %}

To verify 'products' resolve is revoled, I create a spy object called 'resolvedSpy' and pass that to the callback of the resolve. Resolve is just a method that return promise so we can access it from $state.current.resolve['products']. By using $injector.invoke to invoke the method, all arguments will be supplied fom [$injector](https://docs.angularjs.org/api/auto/service/$injector). Don't forget to call $digest and then we can verify that our spy object has been called or not.

After putting everything together, you can check out the full test below.

{% highlight javascript %}
describe('Router', function() {
  var $state, $rootScope, $location, $injector, $q;
  var userRole = 'user',
      mockSecurity = {}, mockProductService = {};

  beforeEach(function() {
    module(function($provide) {
      $provide.value('securityService', mockSecurity);
      $provide.value('productService', mockProductService);
    });

    module('app.routing');
    module('test.template');

    inject(function(_$state_, _$rootScope_, _$location_, _$injector_, _$q_, _$ocLazyLoad_) {
      $state = _$state_;
      $location = _$location_;
      $rootScope = _$rootScope_;
      $injector = _$injector_;
      $q = _$q_;
      $ocLazyLoad = _$ocLazyLoad_;
    });
  });

  beforeEach(function() {
    userRole = 'user';
    spyOn($ocLazyLoad, 'load').and.callFake(function(args) {
      return $q.when('resolved');
    });
  });

  function goTo(url) {
    $location.url(url);
    $rootScope.$digest();
  }

  describe('Login', function() {
    it('should go to /login', function() {
      goTo('/login');

      var currentState = $state.current;
      expect(currentState.name).toEqual('login');
      expect(currentState.url).toEqual('/login');
      expect(currentState.templateUrl).toEqual('login.html');
    });
  });

  describe('Product', function() {
    beforeEach(function() {
      mockSecurity.isAuthenticated = jasmine.createSpy().and.returnValue(true);
      mockSecurity.isAuthorized = jasmine.createSpy().and.callFake(function(permission) {
        return permission.indexOf(userRole) >= 0 ? true : false;
      });
      
      mockProductService.getAll = jasmine.createSpy().and.callFake(function() {
        return $q.when([{productId:1}, {productId:2}]);
      });
    });

    it('should go to /products', function() {
      userRole = 'user';
      goTo('/products');

      var currentState = $state.current;
      expect(currentState.name).toEqual('app.product');
      expect(currentState.url).toEqual('^/productss');
      expect(currentState.templateUrl).toEqual('products.html');
      expect(currentState.data.permissions).toEqual(permission.user);

      var resolvedSpy = jasmine.createSpy();
      $injector.invoke(currentState.resolve['products']).then(resolvedSpy);
      $rootScope.$digest();
      expect(resolvedSpy).toHaveBeenCalledWith([{productId:1}, {productId:2}]);
    });

    it('should go to /products/new', function() {
      currentRole = 'admin';
      goTo('/products/new');

      var currentState = $state.current;
      expect(currentState.name).toEqual('app.product.new');
      expect(currentState.url).toEqual('/new');
      expect(currentState.templateUrl).toEqual('createProduct.html');
      expect(currentState.data.permissions).toEqual(permission.admin);
    });

    it('should go to error404', function() {
      currentRole = 'user';
      goTo('/products/new');

      var currentState = $state.current;
      expect(currentState.name).toEqual('error404');
    });
  });
});
{% endhighlight %}

