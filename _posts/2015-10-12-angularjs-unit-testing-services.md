---
layout: post
title: "AngularJs Unit Testing: Services"
quote: "Testing angular's services with Jasmine"
video: false
comments: true
theme_color: 302F2D
tags: [angularjs, jasmine, unit-testing, service]
---

# Unit Testing Services with Jasmine #

In unit testing, we want to make sure that a particular class is working correctly without concerning the behavior of its dependent components (with no external dependencies). 

Service and controller are the two most common pieces of code we want to test. There are a few differences between testing service and controller classes, but idea is basically the same. Jasmine is a simple and powerful framework that helps us to write a unit test, but that alone might not be so easy as it seems. Luckily, angular has provided many tools to make unit testing easier. 

Let's start with how to test angular service. Here we have a simple service called ‘productService’ which responsible to get the data from the server via $http service with getAll() that will return a list of products.

{% highlight javascript %}
angular.module('app.product')
  .service('productService', function($http, modalService) {
    this.getAll = function() {
      return $http.get('api/products')
        .then(function(response) {
          return response.data;
        })
        .catch(function(error) {
          modalService.handleError(error);
        });
    };
});
{% endhighlight %}


There are many steps before we can write an actual test. We need to set up a test, load required modules, spy and mock dependencies.

The productService is registered to the 'app.product' module and we can load it the same way we load the module to register a service or controller.


{% highlight javascript %}
module('app.product');
{% endhighlight %}

### Working with $httpBackend ###

It also requires $http service to get the data from the server and modalService to handle the error. We want to test that our service calls the api with the expected request, but we don't want to actually make a real request to the server. This is when you should get to know $httpBackend a service in module ngMock by angular. 

[$httpBackend](https://docs.angularjs.org/api/ngMock/service/$httpBackend) is a fake HTTP backend implementation for unit testing in application. It is going to help us verify the requests and respond without sending a request to the real server. 

We can inject $httpBackend and our service (productService) to our test in two ways.

The first way is to inject $injector and use $injector to get other services

{% highlight javascript %}
inject(function($injector) {
  $httpBackend = $injector.get('$httpBackend');
  productService = $injector.get('productService');
});
{% endhighlight %}

or we can inject other services directly. Notice the use of _ before and after the name of the service. Angular will ignore this underscore for us and get the services.

{% highlight javascript %}
inject(function(_$httpBackend_, _productService_) {
  $httpBackend = _$httpBackend_;
  productService = _productService_;
});
{% endhighlight %}


### Spy/Mock objects ###

Next, we need to mock and spy modalService. We can easily create a mock like creating any object and use jasmine spy to spy on any object or function. 

{% highlight javascript %}
mockModalService = {
  handleError: jasmine.createSpy('handleError')
}
{% endhighlight %}

Then, we use $provide to tell Angular to use our mock object instead of the actual one.

{% highlight javascript %}
module(function($provide) {
  $provide.value('modalService', mockModalService);
});
{% endhighlight %}

All the above will be placed in the beforeEach() block of the test.

Let's get to the test. We are going to test getAll method if it calls the server and return the response. If there's an error, the modalService is going to handle that. 

From Angular document, there are two ways to specify what test data should be returned as http responses by the mock backend when the code under test makes http requests:
 
* $httpBackend.expect(method, url, [data], [headers], [keys]) - specifies a request expectation and what to return
* $httpBackend.when(method, url, [data], [headers], [keys]) - specifies a backend definition

There are shortcuts for each of these e.g. expectGET, whenGET

### What is the difference between these two? ###

To put it easy, expect is stricter than when. Both can be used to define fake backend but only expect will make assertions for requests and responses. It will fail if it's not as expect or in a wrong order. More detail in [$httpBackend](https://docs.angularjs.org/api/ngMock/service/$httpBackend).

We are going to fake a response for getAll with .when(..) which returns requestHandler, an object with respond method that can be saved for later use. Call our service and assert the response.

{% highlight javascript %}
beforeEach(function() {
  requestHandler = $httpBackend.when('GET', 'api/products').respond([{productId: 1}, {productId: 2}]);
});

it('should return all productService', function() {
  $httpBackend.expectGET('api/products');
  
  var products;
  productService.getAll().then(function(response) {
    products = response;
  });

  $httpBackend.flush();

  expect(products.length()).toEqual(2);
  expect(products[0].productId).toEqual(1);
  expect(products[0].productId).toEqual(1);
  expect(modalService, 'handleError').not.toHaveBeenCalled();
});
{% endhighlight %}

### What if the server thows an error? ###
We can change the respond of requestHandler to return error with 

> requestHandler.respond([status,] data[, headers, statusText])

and verify that the error has been handled as we expected.

{% highlight javascript %}
it('should handle error', function() {
    requestHandler.respond(400, 'Bad Request');

    $httpBackend.expectGET('api/products');

    var result;
    productService.getAll().then(function(response) {
      result = response;
    });

    $httpBackend.flush();
    
    expect(modalService, 'handleError').toHaveBeenCalledWith('Bad Request');
  });
{% endhighlight %}

One of the important thing to note is the use of .flush(). Since $http make asynchronous request, we need a way to flush pending requests to be able to test this synchronously.

Here is the complete test for productService.

{% highlight javascript %}
describe('productService', function() {
  var productService, $httpBackend, mockModalService;

  beforeEach(function() {
    // load 'app' module
    module('app');

    // This is our mock modalService
    mockModalService = {
      handleError: jasmine.createSpy('handleError')
    };

    // Here is how we tell angular to use this mock instead of the real one
    module(function($provide) {
      $provide.value('modalService', mockModalService);
    });

    // Then, we use inject() to get productService and $httpBack to our test
    inject(function(_productService_, _$httpBackend_) {
      productService = _productService_;
      $httpBackend = _$httpBackend_;
    });

    // Default handler for get all products request
    requestHandler = $httpBackend.when('GET', 'api/products')
                                 .respond([{productId: 1}, {productId: 2}]);
  });

  it('should return all productService', function() {
    $httpBackend.expectGET('api/products');
    
    var result;
    productService.getAll().then(function(response) {
      result = response;
    });

    $httpBackend.flush();

    expect(products.length()).toEqual(2);
    expect(products[0].productId).toEqual(1);
    expect(products[0].productId).toEqual(1);
    expect(modalService, 'handleError').not.toHaveBeenCalled();
  });

  it('should handle error', function() {
    requestHandler.respond(400, 'Bad Request');

    $httpBackend.expectGET('api/products');

    var result;
    productService.getAll().then(function(response) {
      result = response;
    });

    $httpBackend.flush();
    
    expect(modalService, 'handleError').toHaveBeenCalledWith('Bad Request');
  });
  
});
{% endhighlight %}

