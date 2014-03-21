---
layout: post
title: "4dashes - the web client, part 2"
date: 2014-03-21 10:18:34 -0400
comments: true
keywords: 4dashes,angular,directive,ui,web components
description: A post covering the user interface implmentation for 4dashes using Angular.js.
---

This is the fourth in a series of posts discussing the implementation of the
[4dashes][1] productivity tool. Continuing from the last [post][2], it covers
the user interface implementation using [Angular.js][3]. The post assumes the
reader has a basic understanding of the Angular framework.

<!-- more -->

## Organizing a basic interface

With a [well-organized model][2] in place, let's shift our attention towards
the user interface implementation that interacts with it. As I previously
mentioned, Angular is a DOM compiler. An interface is contructed with standard
HTML markup coupled with additional elements and attributes to extend behavior
relevant to web applications. On startup, Angular *compiles* this markup, or
template, to construct an object model the browser can render. 

For the most basic applications, an interface can be declared wholly within a
single template using primitives provided by Angular. Additional routes or
views can be easily added with the `$route` service with each route mapping to
a single template (the declarative interface) and an associated controller (the
logic handling user interaction). As I discussed in my [build system post][4],
I organized around functional components so for each route, I had a
corresponding component. Each component has an *html* file representing the
template, a *less* file representing the style, and a *javascript* file
containing the controller. 

{% codeblock %}

4dashes/
	app/
		components/
			today/
				today.js
				today.html
				today.less
			inventory/
			statistics/
			login/
			signup/

{% endcodeblock %}

## Decomposing complex interfaces

As an interface grows more complex, you are forced to consider decomposition of
a view's template. With Angular, two basic approaches exist. The first approach
is typical of most web frameworks -- logical segments of a template are broken
out into *partial* templates and included into the main one using `ng-include`.
It follows that user interaction logic related to that partial template would
be refactored into its own sub-controller. While this appproach handles the
growing complexity of an interface, there are downsides. Partial templates
establish an **implicit** fragile interface with the main template.
Specifically, the partial will likely depend upon scope defined within it's
parent making it difficult to test in isolation and increasing the liklihood
for regression. The second approach resolves this.

As I have continued to point out, Angular is a DOM compiler providing you with
the mechanism to extend the HTML language with our own domain-specific elements
and attributes. The second approach takes advantage of this by composing a
complex template with custom elements encapsulating markup and presentation
logic. The custom element and its attributes provide an **explicit** interface
to the template and therefore supports testing in isolation and forms the basis
for reusability. As a bonus, it aligns well with future standards regarding
[web components][5].

Many choose the former approach due to its familiarity and simplicity and only
consider custom elements when reusability is required. However, once you fully
understand the concept of a [directive][6], the mechanism to create custom
elements, it becomes clear that the marginal cost is extremely small. It should
be adopted as the preferred approach for its testability and readibility. Below
is a code snippet for creating the most basic custom element that encapsulates
its own template:

{% codeblock lang:javascript %}

.directive('avatar', function () {
	return {
		replace: true,
		templateUrl: 'template/components/avatar/avatar.html'
	}
})

{% endcodeblock %}

## Handling user interaction

Once you start following the guideline articulated above, a question regarding
the handling of user interaction arises. Should your directive communicate
directly with your services (domain model)? Its certainly easy to do so. A
service is injected into a directive and when a DOM event is triggered by user
action, the directive responds by delegating directly to a service method. But
what if other parts (components) of the interface are also interested in the
event? You could broadcast it on an event bus (`$rootScope`) but that has the
disadvantage of spreading out user interaction logic across the application.

Instead, we can follow the approach of primitive HTML elements &mdash; provide
an attribute whose value is a delegate function that is called when a user
action occurs. This establishes an explicit interface while ensuring that the
handling of presentation logic is the *single reponsiblity* of a directive.
Below is a snippet from the inventory view's template (the sidebar)
highlighting delegate functions for custom elements:

{% codeblock lang:html app/components/inventory/inventory.html %}

<div id="inventory">
	...
	<div>
		<statistic
			title="working days"
			value="etc()"
			enabled="{{ isWorkingDaysEnabled() }}">
			estimate to complete flagged tasks
		</statistic>
		<label-list selected="selectedLabels" oncreate="addLabel(label)">
			<label-list-item
				ng-repeat="label in user.labels | orderBy:'name'"
				onchange="renameLabel(label, value)"
				onremove="removeLabel(label)">
			</label-list-item>
		</label-list>
	</div>
</div>

{% endcodeblock %}

As you can see from this example, the inventory's sidebar is declared at a
higher level of abstraction than primitive HTML elements. The `<label-list>`
element exposes an interface allowing the `InventoryController` to handle its
single responsibility of coordinating user actions with application services
(domain model) and other directives (components).

As a consequence of following the guidelines above, we are left with a single
template per view and a controller paired with it to handle user interaction.
For duplicative controller logic across views, I preferred a
[mixin strategy][7] wherein common behavior was added using `angular.extend`.

## Handling complex presentation logic

At this point you may be wondering how to handle complex presentation logic. Is
it enough to simply implement private functions for use within your directive's
`link` function? For simple cases, yes, but as the logic becomes more complex
so does unit testing the directive itself. When the situation arose, I
refactored the complex presentation logic into one or more services 'private'
to the directive. Think of them as support functions that can be tested in
isolation. 

Below is a snippet of my `chart` directive responsible for rendering a stacked
bar chart with a trend line. The implementation responsible for drawing on the
canvas is delegated to a service within the component's module. 

{% codeblock lang:javascript app/components/chart/chart.js %}

angular.module('chart', [])

.directive('chart', function (chart) {
	return {
		restrict: 'E',
		replace: true,
		templateUrl: 'template/chart/chart.html',
		scope: {
			data: '=',
			options: '='
		},
		link: function (scope, element, attrs) {
			var percent = scope.options && scope.options.percentageValue ? '%' : ''
			var canvas
			var cb

			// setup legends
			scope.legends = []
			scope.data.forEach(function (dataset) {
				scope.legends.push({
					title: dataset.legend,
					color: dataset.color
				})
			})

			// render chart on canvas element
			canvas = element.find('canvas')
			cb = chart(canvas, scope.data, scope.options)

			// setup mousemove callback to display data points
			canvas.on('mousemove', function (event) {
				result = cb(event)
				...
			})

			// clean up event handlers on destroy
			scope.$on('$destroy', function () {
				canvas.off()
			})
		}
	}
})

.factory('chart', function ($window) {
	return function (element, datasets, options) {
		var ctx = element[0].getContext('2d')

		// complex canvas rendering here

		// return function to convert mousemove event into data points
		return function (event) {
		}
	}
})

{% endcodeblock %}

I hope you have found this two-part series on Angular.js helpful in
constructing larger web applications. If there is interest and time permits, I
may write an article discussing my unit test strategy. In either case, it
should be reasonably clear that following the guidelines I discussed will yield
an application comprised of testable web components. Ultimately, that is what
makes Angular.js special and a great candidate for tackling client-side
applications.

[1]: https://4dashes.com
[2]: http://shawn.dahlen.me/blog/2014/03/19/4dashes-the-web-client-part-1/
[3]: http://angularjs.org
[4]: http://shawn.dahlen.me/blog/2014/03/13/4dashes-the-build-system/
[5]: https://developers.google.com/events/io/sessions/318907648
[6]: http://docs.angularjs.org/guide/directive
[7]: http://digital-drive.com/?p=188
