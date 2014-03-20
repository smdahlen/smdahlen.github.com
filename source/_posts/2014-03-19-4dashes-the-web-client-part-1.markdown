---
layout: post
title: "4dashes - the web client, part 1"
date: 2014-03-19 14:28:37 -0400
comments: true
keywords: 4dashes,angular,api,authentication,web
description: A post covering the web services communication and implementation of business logic for 4dashes using Angular.js.
---

This is the third in a series of posts discussing the implementation of the
[4dashes][1] productivity tool. It covers the web services communication and
implementation of business logic within the [Angular-based][2] web application.
The post assumes the reader has a basic understanding of the Angular framework.

<!-- more -->

## Understanding Angular 

After a few months of using Angular to build 4dashes, it became
clear that Angular was a library for building a tailor-made framework
for your application. Consequently, there was more than
one way to implement a solution and guidance was marginal at best. At first
this was a struggle, but as I better understood the core concepts behind
Angular I crafted a set of guidelines that shaped my implementation and should
prove useful for new developers.

The big realization was that Angular, at its core, is a DOM compiler.
What I mean by that is it provides the mechanics to extend the HTML language
into your domain through custom elements and attributes. The net result is
declarative markup that represents a higher level abstraction of the interfaces
and their associated behavior in your application. This is paired with a
two-way data binding mechanism in which changes to your data model are
reflected in the DOM and vice-versa. In tandem with the DOM compiler and
two-way data binding, Angular provides a simple mechanism for dependency
injection. This enables us to structure our code into testable units that
depend upon one another with dependencies instantiated and resolved at runtime.
A set of primitives common to web applications is built on top of these
concepts and provided by the library.

## Communicating with the API

Lets begin with the implementation responsible for communicating with the
server-side API discussed in the [previous post][3]. To keep it simple, I
started with Angular's `$http` service. This quickly grew into a custom
`resource` service resembling the optional `$resource` service provided by the
Angular team. I treaded down this path for two reasons: 1) support for PATCH
requests with automatic change tracking (similar to [Backbone][4]), and 2)
support for offline use with data synchronization.

Unfortunately, time did not permit a clean implementation of either requirement
but I proceeded with my custom service anyway (if there is interest, I can
expand on the issues I ran into). As you will see below, the implementation was
small and straightforward while providing me a great understanding of Angular's
`$http` service and *promises* implementation.

The implementation aligned with my creation and update semantics (e.g.
conditional updates, creation with `PUT`), cached results for subsequent
fetches, transformed RFC8601 strings into dates, and provided an initialization
hook to modify the resource instance on creation.

{% codeblock lang:javascript app/components/api/api.js %}

angular.module('api', [])

.factory('resource', function ($http, $q, $cacheFactory) {
	return function (template, defaultParams) {
		var cache

		template = urltemplate.parse(template)
		cache = $cacheFactory(template.expand({}))

		function Resource(data) {
			var self = this
			var regex = /^(\d{4}|\+\d{6})(?:-(\d{2})(?:-(\d{2})(?:T(\d{2}):(\d{2}):(\d{2})\.(\d{1,})(Z|([\-+])(\d{2}):(\d{2}))?)?)?)?$/
			var match

			if (angular.isFunction(this.initialize)) {
				this.initialize()
			}

			angular.extend(self, data)

			angular.forEach(self, function (value, key) {
				// transform iso 8601 strings into date objects
				if (angular.isString(value)) {
					match = regex.exec(value)
					if (match) {
						self[key] = new Date(Date.parse(match[0]))
					}
				}
			})
		}

		Resource.create = function (resource, params) {
			var url = Resource.url(params, resource)

			return $http({
				method: template.expand({}) === url ? 'POST' : 'PUT',
				url: url,
				data: resource
			})
			.then(function (response) {
				resource.modified = new Date(response.headers('last-modified'))
				return resource
			})
		}

		Resource.fetch = function (params, fresh) {
			var url = Resource.url(params)
			var result

			if (!fresh) {
				result = cache.get(url)
				if (result) { return $q.when(result) }
			}

			return $http.get(url).then(function (response) {
				if (Array.isArray(response.data)) {
					result = []
					response.data.forEach(function (value, key) {
						result.push(new Resource(value))
					})
				} else {
					result = new Resource(response.data)
				}

				cache.put(url, result)

				return result
			})
		}

		Resource.url = function (requestParams, resource) {
			var params = angular.extend({}, defaultParams, requestParams)

			angular.forEach(params, function (value, key) {
				if (angular.isFunction(value)) {
					params[key] = value(resource)
				} else if (angular.isString(value) && value.charAt(0) === '@') {
					params[key] = resource ? resource[value.slice(1)] : null
				}
			})

			return template.expand(params)
		}

		Resource.prototype.save = function (params) {
			var self = this
			var url = Resource.url(params, self)

			return $http({
				method: 'PUT',
				url: url,
				headers: {
					'content-type': 'application/json',
					'if-unmodified-since': self.modified.toString()
				},
				data: self
			})
			.then(function (response) {
				self.modified = new Date(response.headers('last-modified'))
				return self
			})
		}

		return Resource
	}
})

{% endcodeblock %}

Similar to `$resource`, a concrete resource is defined by supplying a base URI
and template parameters (see [RFC 6570: URI Templates][5]). Here is how I
defined the concrete `Task` and `Summary` resources that communicates with
endpoints defined in the previous post:

{% codeblock lang:javascript app/components/api/api.js %}

.factory('Task', function (resource) {
	var Task = resource ('/api/tasks{/id}', { id: '@id' })

	...

	return Task
})

.factory('Summary', function (resource) {
	var Summary = resource('/api/summaries{/year}{/month}{/day}', {
		year: function (summary) { return format(summary, 'YYYY') },
		month: function (summary) { return format(summary, 'MM') },
		day: function (summary) { return format(summary, 'DD') }
	})

	...

	return Summary

	function format(summary, token) {
		return summary ? moment(summary.day).format(token) : null
	}
})

{% endcodeblock %}

This `Task` resource could now be used in the following manner:

{% codeblock lang:javascript %}

// retrieves an array of incomplete tasks (GET /api/tasks)
var tasks = Task.fetch()

// modifies and saves the first task in the array (PUT /api/tasks/{id})
tasks[0].title = 'new title'
tasks[0].save()

// creates a new task and prints out modified date from server
Task.create({ id: 'guid', title: 'new task' }).then(function (task) {
	console.log(task.modified)
})

{% endcodeblock %}

## Organizing business logic

Before discussing authentication with the API, let me build upon the `Task` and
`Summary` services defined above. Angular is the least prescriptive when it
comes to modelling and organizing your business (and persistence) logic. Most
beginner examples implement an application's logic directly within a
`Controller` and that may even include `$http` calls. However, as an
application grows, a controller's implementation will become unwieldy,
duplicative, and difficult to test. As such, the better approach is to move the
application logic into a set of services that your controllers can depend on.
This yields thin controllers with the single responsibilty of coordinating
between the view (directives, templates) and model (services).

To implement my business logic, I followed an active record like approach
defining methods on the `Task`, `Summary`, and `User` prototypes. Here is a
brief look at how behavior was added to `Task` for use by one of the
controllers:

{% codeblock lang:javascript %}

.factory('Task', function (resource) {
	var Task = resource ('/api/tasks{/id}', { id: '@id' })

	Task.prototype.applyLabels = function (labels) {
		var self = this
		var added = false

		labels.forEach(function (newLabel) {
			var contains = self.labels.some(function (label) {
				return label === newLabel
			})

			if (!contains) {
				added = true
				self.labels.push(newLabel)
			}
		})

		return added
	}

	return Task
})

.controller('InventoryController', function ($scope) {
	$scope.applyLabels = function (labels) {
		$scope.selectedTasks.forEach(function (task) {
			if (task.applyLabels(labels)) {
				task.save()
			}
		})
		$scope.selectedTasks = []
	}
})

{% endcodeblock %}

In addition to implementing methods on the prototype object of the services, I
used virtual properties to derive values from the primary data within a
resource's document. This kept the payload as small as possible. To mitigate
the calculation cost, virtual properties were memozied (cached to return the
same value on subsequent calls). Here is a brief look at the `Summary`'s
`initialize()` method that creates the virtual properties for an instance:

{% codeblock lang:javascript %}

Summary.prototype.initialize = function () {
	var self = this

	self.plannedTasks = []
	self.completedTasks = []
	self.dashes = []
	self.sets = 0
	self.internalInterruptions = 0
	self.externalInterruptions = 0

	Object.defineProperties(self, {
		plannedDashes: {
			get: memoize(function () {
				return self.dashes.reduce(function (sum, dash) {
					return self.isPlannedTask({ id: dash.id }) ? ++sum : sum
				}, 0)
			}, self.hashKey)
		},
		...
	})
}

{% endcodeblock %}

## Handling authentication

In the previous [post][3], I discussed the token-based authentication mechanism
used by the API. To submit a request, a client must supply a valid signed token
within a HTTP header. A valid token can be retrieved by `POST`ing a request to
`/api/token` with an email address and plaintext password. To support this, I
implemented a `token` service responsible for authentication and persistence of
the token and a an `$http` [interceptor][6] to inject the token into API
requests.

Below is the implementation of the `token` service. There is a few things to
note: 1) the service provides a method to `authenticate` a user's credentials
used by the login form, 2) the token is set and retrieved via a virtual
property and persisted to `localStorage`, and 3) a timeout is scheduled to fire
two minutes before the token expires and when it does expire &mdash; both
broadcasting an event for interested listeners. 

{% codeblock lang:javascript app/components/api/api.js %}

.factory('token', function ($window, $timeout, $injector, $rootScope) {
	var token = $window.localStorage.getItem('token')
	var expire

	if (token) { startTokenTimeout() }

	return Object.create({
		isValid: function () {
			return token ? true : false
		},

		authenticate: function (email, password) {
			return $injector.get('$http').post('/api/token', {
				email: email,
				password: password
			})
		}
	},
	{
		value: {
			get: function () { return token },
			set: function (newToken) {
				token = newToken
				if (token) {
					$window.localStorage.setItem('token', token)
					startTokenTimeout()
				} else {
					$window.localStorage.removeItem('token')
				}
			}
		}
	})

	function startTokenTimeout() {
		// clear old timeout
		if (expire) {
			$timeout.cancel(expire)
		}

		// parse time remaining minus two minute buffer
		var timeRemaining = token.split(':')[1] - Date.now() - (2*60*1000)

		if (timeRemaining <= 0) {
			token = null
			$rootScope.$broadcast('tokenExpired')
		} else {
			expire = $timeout(function ()	{
				// alert listeners that token will expire in one minute
				$rootScope.$broadcast('tokenWillExpire')
				expire = $timeout(function () {
					token = null
					$rootScope.$broadcast('tokenExpired')
				}, 60*1000)
			}, timeRemaining)
		}
	}
})

{% endcodeblock %}

The `tokenInterceptor` service intercepts `$http` requests and for API calls
injects a header with a token retrieved from the `token` service. It also
intercepts the response to retrieve the new token and persist it with the
`token` service for later use. Below is the implementation of the
`tokenInterceptor` service and the associated code to configure it:

{% codeblock lang:javascript app/components/api/api.js %}

.factory('tokenInterceptor', function ($q, token) {
	var TOKEN_HEADER = 'x-access-token'

	return {
		request: function (config) {
			if (/api/.test(config.url)) {
				config.headers[TOKEN_HEADER] = token.value
			}
			return config
		},
		response: function (response) {
			if (/api/.test(response.config.url)) {
				token.value = response.headers(TOKEN_HEADER)
			}
			return response
		},
		responseError: function (response) {
			if (response.status === 401) {
				token.value = null
			}
			return $q.reject(response)
		}
	}
})

.config(function ($httpProvider) {
	$httpProvider.interceptors.push('tokenInterceptor')
})

{% endcodeblock %}

On startup, the application is configured to listen for the `$routeChangeStart`
event. When a user navigates to a route for the first time or changes to a new
one, the application checks if the route is restricted and if the token is
invalid. If both are true, the user is redirected to the login view. Below is a 
snippet of code highlighting the configuration of a restricted route (note the
`restricted` property) and the associated logic for login redirection:

{% codeblock lang:javascript app/app.js %}

// configure app routes
.config(function ($routeProvider, $locationProvider) {
	$routeProvider
	.when('/today', {
		templateUrl: 'template/today/today.html',
		controller: 'TodayController',
		restricted: true,
		resolve: {
			user: function (User) { return User.fetch() },
			tasks: function (Task) { return Task.fetch() },
			summaries: function (Summary) { return Summary.fetch() }
		}
	})
	.when('/login', {
		templateUrl: 'template/login/login.html',
		controller: 'LoginController',
		restricted: false
	})
	...
})

// initialize the app on startup
.run(function ($rootScope, $location, token) {
	// listen for a route change and redirect to /login if a valid token
	// is not present
	$rootScope.$on('$routeChangeStart', function (event, next) {
		if (next.restricted && !token.isValid()) {
			$location.path('/login')
		}
	})

	// listen for token expiration and redirect to /login
	$rootScope.$on('tokenExpired', function (event) {
		$location.path('/login')
	})
})

{% endcodeblock %}

With the discussion regarding API communication and business logic complete,
the next post will cover the user interface implmentation.

[1]: https://4dashes.com
[2]: http://angularjs.org
[3]: http://shawn.dahlen.me/blog/2014/03/17/4dashes-the-api/
[4]: http://backbonejs.org/#Model-save
[5]: http://tools.ietf.org/html/rfc6570
[6]: http://docs.angularjs.org/api/ng/service/$http
