---
layout: post
title: "AngularJs Unit Testing: Controllers"
quote: "Testing controllers and handling promises"
video: false
comments: true
theme_color: 302F2D
tags: [angularjs, jasmine, unit-test, controller, promises, $httpBackend]
---

#Unit Testing Controllers with Jasmine

Angular has a pretty good document about [unit testing](https://docs.angularjs.org/guide/unit-testing). If you haven't read it, give it a try. It covers the basic of unit testing in general and in AngularJS application. It is a very good guideline for understanding what we will need to get started and what features angular provide us to help testing easier. However, the application we want to test is always a lot more complicated than these examples.

Here, I am going to test productCtrl that has dependencies to $scope, $state, and our productService. If you remember from part I, the productSrvice use $http service which return promise to get data from the server. The implementation of productService is not important here. Just keep in mind that it returns promise object from getAll and getDetail.

{% highlight javascript %}
angular.module('app.product')
  .service('productService', function() {
    this.getAll = function() { return $http.get(...); }

    this.getDetail = function(productId) { return $http.get(...); }
  })
  .controller('productCtrl', function($scope, $state, productService){

    productService.getAll().then(function(products) {
      // An array of product (productId, productName)
      $scope.products = products;
    });

    $scope.getDetail = function(productId) {
      productService.getDetail(productId).then(function(product) {
        $state.go('product.detail', {product: product});
      })
    }
  });
{% endhighlight %}


What I would like to explain in this post is how to test a controller which has dependencies to service that return promise (e.g. $http).

### How to mock a promise? ###

First, we create an object for productService with two methods: getAll and getDetail by using jasmine.createSpy.

{% highlight javascript %}
mockProductService = {
    getAll: jasmine.createSpy('getAll'),
    getDetail: jasmine.createSpy('getDetail')
}
{% endhighlight %}

This is not going to return anything when we call so we need to implement a fake function that will return a promise. To create a promise, we need [$q](https://docs.angularjs.org/api/ng/service/$q) a service that helps you run function asynchronously.

{% highlight javascript %}
products = [];
mockProductService = {};
inject(function($q) {
    mockProductService.getAll = jasmine.createSpy('getAll').and.callFake(function() {
        var defer = $q.defer();
        defer.resolve(products);
        return defer.promise;
    });
    
    mockProductService.getDetail = jasmine.createSpy('getDetail').and.callFake(function(id) {
        var defer = $q.defer();
        defer.resolve({productId: id});
        return defer.promise;
    });
});
{% endhighlight %}

If we want to mock failed response, you can replace defer.resolve with

{% highlight javascript %}
defer.reject('Something is wrong!');
{% endhighlight %}

It will go to your error calllback or catch block whichever you prefer.

We can now create a controller with mockProductService.

{% highlight javascript %}
inject(function($rootScope, $controller) {
    $scope = $rootScope.$new();
    controller = $controller('productCtrl', {
      $scope: $scope,
      $state: mockState,
      productService: mockProductService
    });
});
{% endhighlight %}

Now, let's get to the test. It's very easy to test that our $scope is populated with product list during initilization. Since we already created our controller in the beforeEach block, we can go straight to the test and verify that.

{% highlight javascript %}
it('should initialize scope with products', function() {
  expect(mockProductService.getAll).toHaveBeenCalled();
  expect($scope.products).toEqual(products);
});

it('should get product detail', function() {
  // productId = 128
  $scope.getDetail(128);

  expect(mockProductService.getDetail).toHaveBeenCalledWith(128);
  expect(mockState.go).toHaveBeenCalledWith('product.detail', {productId: 128});
});
{% endhighlight %}

These two tests look fine and simple, but if you run this test, you might encounter an error. This is because our mockproductService is just an object that returns promise. It is not a real service so after the promise is resolved we need to manually digest the cycle by calling $scope.$apply() to make sure the code in callback is executed and scope is updated. Normally, we do not need to deal with $digest() or $apply() directly because Angular do this for us.

{% highlight javascript %}
$scope.$apply(function() {
    $scope.getDetail(128)
});
{% endhighlight %}

There are some good articles about $apply() and $digest you can read: [Understanding Angular’s $apply() and $digest()](http://www.sitepoint.com/understanding-angulars-apply-digest/)

Here is the complete test.

{% highlight javascript %}
describe('Product Controller', function() {
  var controller,
    $scope,
    mockState = {},
    mockProductService = {},
    products = [];

  beforeEach(function() {
    module('app.product');

    mockState.go = jasmine.createSpy('go');

    inject(function($q) {
      mockProductService.getAll = jasmine.createSpy('getAll').and.callFake(function() {
        var defer = $q.defer();
        defer.resolve(products);
        return defer.promise;
      });

      mockProductService.getDetail = jasmine.createSpy('getDetail').and.callFake(function(id) {
        var defer = $q.defer();
        defer.resolve({
          productId: id
        });
        return defer.promise;
      });
    });

    inject(function($rootScope, $controller) {
      $scope = $rootScope.$new();
      controller = $controller('productCtrl', {
        $scope: $scope,
        $state: mockState,
        productService: mockProductService
      });
    });

  });

  it('should initialize scope with products', function() {
    $scope.$apply();
    expect(mockProductService.getAll).toHaveBeenCalled();
    expect($scope.products).toEqual(products);
  });

  it('should get product detail', function() {
    $scope.getDetail(128);
    $scope.$apply();

    expect(mockProductService.getDetail).toHaveBeenCalledWith(128);
    expect(mockState.go).toHaveBeenCalledWith('product.detail', {
      productId: 128
    });
  });

});
{% endhighlight %}

