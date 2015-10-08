---
layout: post
title: "AngularJS HttpInterceptor"
quote: "Intercepts error from server in one place"
image: false
video: false
comments: true
---


One of the interesting things I have been working with this week is httpInterceptor. I have heard about it before, but never really understand why I would need it. Here are a few scenarios why you might need to implement your own interceptor.

- Every time you make a http request to get some data from the server, you need to set the header with token or some other stuffs for authentication on the server side.
- When the server return 401 (Unauthorized) or 403 (Forbidden), you want to redirect the user to your beautiful error page.
- Generic error handling for any kind of errors.

These are the things I need for my project, but the idea is generic enough to apply to any application. You can do all three of these without interceptor, but you will finally consider using it after a few months when you start to copy and paste the same part of your code to other files. Then, you realize how nice it would be if I can do this everytime I get errors from the server.

So what is httpInterceptor and how does it work?
------------------------------------------------

You can go and read from angularJS document about [interceptor](https://docs.angularjs.org/api/ng/service/$http#interceptors) for detail. In short, interceptors are services that get called before and/or after the request is sent/returned to/from the server.  

First, you can create an interceptors just the same as creating any service or factory. In our custom interceptor service, we can implement any of these four kinds of interceptor and return from the service

    angular.module('interceptor-module', [])
    	.service('myHttpInterceptor', function( ) {
    		return { 
	    		request: function(config) {...},
	    		requestError: function(config) {...},
    			response: function(response) {...},
    			responseError: function(response) {...}
    		}
    	});

That is our custom interceptor registered to interceptor module. I keep in it separate module for the purpose of unit testing. Next we need to register this interceptor to our application. We can do that with this line of code.

    $httpProvider.interceptors.push(‘myHttpInterceptor’);

But where does this line go. The only time we can register an interceptor is during the config. If you do not separate it to a new module like me, but to the main module. You can just register it in the config block of your main module.

    angular.module('app')
    	.config( function($httpProvider, ...) {
		   $httpProvider
			   .interceptors
			   .push('myHttpInterceptor');
    	});

In my case, we just need a few more lines. Add config block to interceptor or module

    angular.module('interceptor-module', [])
		   .service('myHttpInterceptor', function( ) {
			   ...
		   })
		   .config(function($httpProvider, ...) {
			   $httpProvider
				   .interceptors
				    .push('myHttpInterceptor');
	       });

 and include this module as a dependency for the main module.
 
    angular.module('app', ['interceptor-module']);

Make sure to load the js file for the interceptor module before the app is bootstrapped.

Let’s go to the implementation detail. It’s simple and based on your need. 

{% highlight javascript %}
    angular.module('interceptor', [])
      .factory('httpInterceptor', function($q, $location, $injector) {
        return {
          request: function(config) {
            if ( !config.cache ) {
              var token = $injector.get('securityService').securityToken();
              if ( token ) {
                config.headers['X-AUTH-TOKEN'] = token;
              }
            }
            return config;
          },
    
          responseError: function(response) {
            // 401 - Unauthorized, 403 - Forbidden
            if (response.status == 401 || response.status == 403) {
              // Redirect to 404 error page
              return $location.path('/error404');
            }
            else if (response.status == 500) {
              // Redirect to 500 error page
              return $location.path('/error500');
            }
    else if ( response.config && !response.config.defaultErrorHandler ) {
              // Do not perform any error handling here
              return $q.reject(response);
            }
            else {
              // Show error modal
              var modalService = $injector.get('modalService');
              var errorStatus = response.status;
              var error = response.data.error;
              return modalService.showErrorModal(errorStatus, error);
            }
          }
        };
      })
      .config(
        function($httpProvider) {
          $httpProvider.interceptors.push('httpInterceptor');
      });
{% endhighlight %}

It is straightforward in the code. You need to intercept the request before it goes to the server to append the token. So we need to implement ‘response’ interceptors. It takes config as an argument. We can get the headers from config and manipulate it. The only thing worth mentioning is you need to check whether it’s the http call to your api not http call from angular itself for the template. Here I check ‘cached’ property in the config. This is for template request only.

Then, we intercept responseError which takes response as an argument. Then, check for a status and redirect to our error page. For all other cases, we are going to show a modal with error message. Sometimes, we don’t want these default (global) error handler. You want to handle this request specifically. Then, we need to tell our custom interceptor to ignore this. What I did is to add config to a request I want to handle other ways and then check for that flag in the interceptor. 

There are a few tips you may want to know. If you need to use other services and inject it to interceptor service, you might get an errors like circulate dependencies, and etc. One way to do is to inject $injector and use $injector.get(‘...’) to get what you want.