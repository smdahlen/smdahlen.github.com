---
layout: post
title: "4dashes - The Build System"
date: 2014-03-13 13:47:12 -0400
comments: true
keywords: 4dashes,grunt,node.js,angular
description: A post covering the build system for 4dashes using Grunt.
---

This is the first in a series of posts discussing the implementation of the
[4dashes][1] productivity tool. It covers the project structure and build 
system supporting a rapid developer workflow and creation of production assets.
It is a continuation on the groundwork discussed [here][2] representing the
state of the product at launch.

<!-- more -->

4dashes is a web-based product built on [Node.js][3] and [Angular][4]
(deviating from the earlier setup using Backbone). Like many Node.js projects
nowadays, the build system is based on [Grunt][5]. Before discussing the
*Gruntfile* mechanics, its beneficial to understand the project file structure.
Here is the top-level:

{% codeblock %}
4dashes/
	app/
	lib/
	test/
	public/
	Gruntfile.js
	package.json
	server.js
{% endcodeblock %}

The *server.js* file loads configuration, sets up logging, and mounts RESTful
endpoints defined within modules in the *lib/* directory. The details of this
will be discussed in a future article. The *app/* directory contains the
client-side assets that comprise the Angular-based web application and which
are ultimately built to the *public/* directory to serve for production.

{% codeblock %}
4dashes/
	app/
		components/
			timer/
				timer.html
				timer.less
				timer.js
			charts/
			task-list/
			...
		images/
		fonts/
			icons.woff
		index.html
		app.html
		app.less
		app.js
{% endcodeblock %}
		
Unlike other Angular projects, 4dashes was organized around the functional
components that comprise the application (similar to what was recently
[proposed][6] by the Angular team.) This approach aligns well with the
upcoming [web components][7] technology. Each component is contained within
its own directory with a file for template markup, styles, and script. The
_app.*_ files represent the entry point for the application and dynamically
reference files for each component. When a new component is created
(or removed), the build system updates the appropriate references. This is
accomplished with the [grunt-file-blocks][8] plugin. 

The plugin works by updating replacement blocks within a source file with
references based on file matching patterns. *app.html* contains replacement
blocks for third-party dependencies and component scripts: 

{% codeblock lang:html app.html %}
<html>
	<body>
		<header>...</header>
		<div id="content" ng-view></div>
		<footer>...</footer>

    <!-- build:js app.js -->
		<!-- fileblock:js dependencies -->
		<script src="bower_components/angular/angular.min.js"></script>
		<script src="bower_components/moment/moment.js"></script>
		...
		<!-- endfileblock -->

		<!-- fileblock:js components -->
		<script src="components/api/api.js"></script>
		<script src="components/avatar/avatar.js"></script>
		<script src="components/timer/timer.js"></script>
		<script src="components/templates.js"></script>
		...
		<!-- endfileblock -->

		<script src="app.js"></script>
    <!-- endbuild -->
	</body>
</html>
{% endcodeblock %}

*app.less* follows the same pattern replacing a comment block with `@import`
statements for components styles:

{% codeblock lang:css app.less %}
// import base bootstrap styles 
@import 'bower_components/bootstrap/less/bootstrap.less';

// define color palette
@base-color: hsl(45, 15%, 99%);
@base-dark-color: hsl(45, 15%, 89%);

// additional common styles
...

/* fileblock:less components */
@import 'components/avatar/avatar.less';
@import 'components/timer/timer.less';
...
/* endfileblock */
{% endcodeblock %}

Finally, component template markup is pre-processed and compiled into a single
*templates.js* file using the [grunt-angular-templates][9] plugin. While this
wasn't technical required for a development workflow it did simplify references
within unit testing code.

The supporting Grunt configuration for these two plugins is as follows:

{% codeblock lang:javascript Gruntfile.js %}
module.exports = function (grunt) {
	require('matchdep').filterDev('grunt-*').forEach(grunt.loadNpmTasks)

	var dependencies = [ ... ]

	grunt.initConfig({
		fileblocks: {
			options: {
				removeFiles: true,
				templates: {
					less: '@import \'${file}\';'
				}
			},
			js: {
				src: 'app/app.html',
				blocks: {
					dependencies: { cwd: 'app', src: dependencies },
					components: { cwd: 'app', src: 'components/**/*.js' }
				}
			},
			less: {
				src: 'app/app.less',
				blocks: {
					components: { cwd: 'app', src: 'components/**/*.less' }
				}
			}
		},
		ngtemplates: {
			options: {
				prefix: 'template/',
				standalone: true,
				htmlmin: { collapseWhitespace: true }
			},
			templates: {
				cwd: 'app/components',
				src: '**/*.html',
				dest: 'app/components/templates.js'
			}
		}
	})
}
{% endcodeblock %}

This setup eliminated the need to manually update referenced assets and
provided the basis for a rapid development workflow and a streamlined 
production build.

During development, assets are served from the *app/* directory. A custom
Grunt task was registered to start the server and reload on both client- and
server-side changes. For client-side changes, less files and angular templates
are processed and the browser is refreshed using [LiveReload][10]. For
server-side changes, the Node.js server process is restarted. During 
development, the developer simply runs `grunt dev` on the command line, points
a browser at `http://localhost:8080`, and edits/saves code to view the changes.

The following Grunt configuration supports this workflow. It leverages 
[grunt-contrib-watch][11] and [grunt-nodemon][12] to watch and reload client-
and server-side resources respectively. [grunt-concurrent][13] allows these
two plugins to run concurrently.

{% codeblock lang:javascript Gruntfile.js %}
module.exports = function (grunt) {
	require('matchdep').filterDev('grunt-*').forEach(grunt.loadNpmTasks)

	var dependencies = [ ... ]

	grunt.initConfig({
		concurrent: {
			dev: {
				tasks: ['nodemon', 'watch'],
				options: {
					logConcurrentOutput: true
				}
			}
		},
		nodemon: {
			dev: {
				options: {
					file: 'server.js',
					ignoredFiles: ['app', 'test', 'node_modules'],
					watchedExtensions: ['js']
				}
			}
		},
		watch: {
			app: {
				options: { livereload: true },
				files: [
					'app/app.*',
					'app/components/**/*.js',
					'app/images/*'
				]
			},
			templates: {
				tasks: ['ngtemplates'],
				files: ['app/components/**/*.html']
			},
			less: {
				tasks: ['less'],
				files: ['app/app.less', 'app/components/**/*.less']
			}
		},
		less: {
			build: {
				files: { 'app/app.css': 'app/app.less' }
			}
		}
	})

	grunt.registerTask('dev', [
		'ngtemplates',
		'fileblocks:js',
		'fileblocks:less',
		'less',
		'concurrent'
	])
{% endcodeblock %}

For production, the build system uses the excellent [grunt-usemin][14] plugin
to replace non-optimized scripts and stylesheets in *app.html* with a 
reference to a single optimized *app.js* and *app.css* file. This is 
combined with the [grunt-rev][15] plugin for asset revisioning to support cache
busting when changes are made. Ultimately, the application consists of three
core files, *app.html*, *app.js*, and *app.css* with the later two served with
an explicit cache header. 

To round out the production build, [grunt-ngmin][16] is used to pre-minify
Angular scripts so that minifers appropriately handle the dependency
injection style used by the framework.

{% codeblock lang:javascript Gruntfile.js %}
module.exports = function (grunt) {
	require('matchdep').filterDev('grunt-*').forEach(grunt.loadNpmTasks)

	var dependencies = [ ... ]

	grunt.initConfig({
		clean: {
			build: ['public'],
			templates: ['app/templates/**/*.js'],
			less: ['app/styles/*.css'],
			test: ['test/**/*.log']
		},
		copy: {
			fonts: {
				expand: true,
				cwd: 'app/fonts',
				src: '**',
				dest: 'public/fonts/',
				flatten: true,
				filter: 'isFile'
			},
			sounds: {
				expand: true,
				cwd: 'app/sounds',
				src: '**',
				dest: 'public/sounds/',
				flatten: true,
				filter: 'isFile'
			}
		},
		useminPrepare: {
			options: { dest: 'public' },
			html: 'app/*.html'
		},
		usemin: {
			html: 'public/*.html',
			css: ['public/*.css', 'public/index.html'],
			js: ['public/*.js'],
			options: {
				patterns: {
					js: [[
						/["']([^:"']+\.(?:png|gif|jpe?g|css|js))["']/img,
						'Update JavaScript with assets in strings'
					]]
				}
			}
		},
		htmlmin: {
			build: {
				files: [{
					expand: true,
					cwd: 'app',
					src: '*.html',
					dest: 'public'
				}]
			}
		},
		imagemin: {
			build: {
				files: [{
					expand: true,
					cwd: 'app/images',
					src: '*.{jpg,png}',
					dest: 'public/images'
				}]
			}
		},
		ngmin: {
			build: {
				files: { 'public/app.js': 'public/app.js' }
			}
		},
		rev: {
			build: {
				src: ['public/**/*.{js,css,png,jpg,woff}']
			}
		}
	})

	grunt.registerTask('build', [
		'clean',
		'fileblocks:js',
		'fileblocks:less',
		'less',
		'copy',
		'useminPrepare',
		'ngtemplates',
		'concat',
		'htmlmin',
		'imagemin',
		'cssmin',
		'ngmin',
		'uglify',
		'rev',
		'usemin'
	])
{% endcodeblock %}

To conclude, a new production server can be run with the following commands:

{% codeblock %}
$ git clone 4dashes.git
$ cd 4dashes
$ npm install --production
$ bower install --production
$ grunt build
$ NODE_ENV=production node server.js 
{% endcodeblock %}

[1]: https://4dashes.com
[2]: http://shawn.dahlen.me/blog/2013/03/18/setup-node-dot-js-servers-within-a-load-balanced-configuration/
[3]: http://nodejs.org
[4]: http://angularjs.org
[5]: http://gruntjs.com
[6]: https://docs.google.com/document/d/1XXMvReO8-Awi1EZXAXS4PzDzdNvV6pGcuaF4Q9821Es/pub 
[7]: https://developers.google.com/events/io/sessions/318907648
[8]: https://github.com/rrharvey/grunt-file-blocks 
[9]: https://github.com/ericclemmons/grunt-angular-templates
[10]: http://livereload.com
[11]: https://github.com/gruntjs/grunt-contrib-watch
[12]: https://github.com/ChrisWren/grunt-nodemon
[13]: https://github.com/sindresorhus/grunt-concurrent
[14]: https://github.com/yeoman/grunt-usemin
[15]: https://github.com/cbas/grunt-rev
[16]: https://github.com/btford/grunt-ngmin 
