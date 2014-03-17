---
layout: post
title: "4dashes - the API"
date: 2014-03-17 12:02:55 -0400
comments: true
categories: 
---

This is the second in a series of posts discussing the implementation of the
[4dashes][1] productivity tool. It covers the server-side API implementation
for HTTP web services built on [Node.js][2] and [Express][3]. It is an in-depth 
continuation of the devops discussion for deploying a [load-balanced 
configuration][4].

<!-- more -->

4dashes is implemented as an Express application. The main application, defined
in *server.js*, loads configuration, configures logging, and detects
unsupported user agents. It delegates the remaining functionality to mounted
sub-applications. Specifically, one serves static assets that comprise the
client-side Angular application (SSL and caching is offloaded to a reverse
proxy) while the other handles API requests. Below is stripped down version of
*server.js* highlighting configuration loaded for the API sub-application and
the mounting of the two apps:

{% codeblock lang:javascript server.js %}

var express = require('express')
var app = express()
var Log = require('log')
var api = require('./lib/api')
var assets = require('./lib/assets')

// define setting for log level
app.set('log level', process.env.DASHES_LOG_LEVEL || Log.DEBUG)

// define setting for port to bind application to
app.set('port', process.env.DASHES_PORT || 8080)

// define setting for 256 bit token key (base64 encoding)
var key = '5zgIDUlloyybplQZtkTzbwoZJusp+SLJj8vjBjqiCh8='
app.set('token key', process.env.DASHES_TOKEN_KEY || key)

// define setting for max age of token
app.set('token age', process.env.DASHES_TOKEN_AGE || 60*60*1000)

// define setting for mongodb database name
app.set('db name', process.env.DASHES_DB_NAME || '4dashes')

// define setting for mongodb hosts
app.set('db hosts', process.env.DASHES_DB_HOSTS || 'localhost')

// configure default logger
var logger = new Log(app.get('log level'))
app.use(function (req, res, next) {
	req.log = logger
	next()
})

// log all requests if log level is INFO or higher using log module format
if (app.get('log level') >= Log.INFO) {
	var format = '[:date] INFO :remote-addr - :method :url ' +
		':status :res[content-length] - :response-time ms';
	express.logger.token('date', function () { return new Date() })
	app.use(express.logger(format))
}

// mount assets and api sub-applications
app.use('/api', require('./lib/api'))
app.use('/', require('./lib/assets'))

app.listen(app.get('port'))

{% endcodeblock %}

The file, *api.js*, defines the Express application responsible for handling
api requests. It is itself comprised of sub-applications that handle requests
for the three core resources: users, tasks, and summaries.

{% codeblock lang:javascript api.js %}

var express = require('express')
var app = express()
var db = require('./db')
var auth = require('./auth')
var users = require('./users')
var tasks = require('./tasks')
var summaries = require('./summaries')

// establish database connection
app.use(db)

// parse request body based on content type
app.use(express.bodyParser())

// define setting for token validation
app.set('validate', auth.validate)

// mount resources
app.use('/token', auth)
app.use('/users', users)
app.use('/tasks', tasks)
app.use('/summaries', summaries)

// handle errors
app.use(function (err, req, res, next) {
	req.log.error(err.message)
	res.send(500)
})

module.exports = app

{% endcodeblock %}

In addition to configuring middleware for parsing JSON request payloads and
converting internal errors into a 500 response code, the api application
imports and uses a database middleware function. This function checks if a
database connection (to a replica set) is established, and if not, attempts to
connect for use in the request pipeline. This approach allowed the startup
order of the application and database servers to be decoupled. The
[Mongoose][5] library is used to communicate with [MongoDb][6].

{% codeblock lang:javascript db.js %}

var mongoose = require('mongoose')
var hosts = ''

module.exports = function (req, res, next) {
	if (hosts === '') {
		hosts = req.app.get('db hosts')
		hosts = hosts.replace(/\s+/g, ' ').split(' ')
		hosts = hosts.map(function (host, index) {
			return 'mongodb://' + host + '/' + req.app.get('db name')
		})
	}

	if (mongoose.connection.readyState === 0) {
		mongoose.connection.once('error', function (err) {
			return next(err)
		})
		mongoose.connect(hosts.join(','))
	}
	next()
}

{% endcodeblock %}

To authenticate and authorize requests, a simple token-based approach was
implemented. Upon successful authentication against the `/api/token` endpoint,
a signed token is returned in a response header containing the user's id and
expiration timestamp. Subsequent API requests are authenticated by the token
passed within a request header and authorized by confirming the user's
ownership of a resource. A new token is returned in each API response extending
the expiration window. To mitigate against session hijacking, all communication
is encrypted with SSL.

The file, *auth.js*, implements the `/api/token` endpoint. Additionally, it
exports (as a property of the Express application) a `validate` middleware
function that is leveraged by the other API sub-applications. 

{% codeblock lang:javascript auth.js %}

var app = require('express')()
var crypto = require('crypto')
var User = require('./models/user')
var TOKEN_HEADER = app.TOKEN_HEADER = 'x-access-token'

// authenticate user credentials in exchange for an access token
app.post('/', function (req, res, next) {
	var key = req.app.get('token key')
	var age = req.app.get('token age')

	User.authenticate(req.body.email, req.body.password, function (err, id) {
		if (err) { return next(err) }
		if (id) {
			var token = generateToken(id, Date.now() + age, key)
			req.log.info('token generated for %s', req.body.email)
			res.set(TOKEN_HEADER, token)
			res.send(204)
		} else {
			res.send(401)
		}
	})
})

// validate signed access token included in request header
// TODO improve logging for fail2ban processing
app.validate = function (req, res, next) {
	var token = req.get(TOKEN_HEADER)
	var key = req.app.get('token key')
	var age = req.app.get('token age')
	var contents

	if (token) {
		contents = token.split(':')
		if (contents.length !== 3) { return res.send(401) }
		User.findById(contents[0], function (err, user) {
			if (err) { return next(err) }
			if (!user) {
				req.log.warning('token received with invalid user id: %s', contents[0])
				res.send(401)
			} else if (!validTimestamp(contents[1])) {
				req.log.debug('token received with expired timestamp')
				res.send(401)
			} else if (!validSignature(contents)) {
				req.log.warning('token received with invalid signature: %s', token)
				res.send(401)
			} else {
				req.user = user
				res.set(TOKEN_HEADER, generateToken(user.id, Date.now() + age, key))
				next()
			}
		})
	} else {
		req.log.warning('token not received for %s', req.originalUrl)
		res.send(401)
	}

	function validTimestamp(timestamp) {
		return timestamp - Date.now() >= 0
	}

	function validSignature(contents) {
		return generateToken(contents[0], contents[1], key)  === contents.join(':')
	}
}

// generate signed access token with user id and timestamp
function generateToken(id, timestamp, key) {
	var content = id + ':' + timestamp
	return content + ':' + crypto
	.createHmac('sha256', new Buffer(key, 'base64'))
	.update(content)
	.digest('base64')
}

module.exports = app
{% endcodeblock %}

The file, *user.js*, implements the `User` model that defines the
`authenticate` method used by the `/api/token` endpoint. A user's email address
and plaintext password submitted to the endpoint is checked against the salted
hash stored within the database. A partial implementation of the `User` model
is shown below highlighting the salted password hashing:


{% codeblock lang:javascript user.js %}

var mongoose = require('mongoose')
var crypto = require('crypto')

var SALTLEN = 32
var KEYLEN = 512
var ITERATIONS = 10000

var schema = new mongoose.Schema({
	hash: { type: String, required: true },
	salt: { type: String, required: true },
	...
})

schema.methods.setPassword = function (plaintext, callback) {
	var self = this

	crypto.randomBytes(SALTLEN, function (err, buf) {
		if (err) callback(err)
		var salt = buf.toString('base64')
		crypto.pbkdf2(plaintext, salt, ITERATIONS, KEYLEN, function (err, key) {
			if (err) callback(err)
			self.salt = salt
			self.hash = key.toString('base64')
			callback()
		})
	})
}

schema.statics.authenticate = function (email, plaintext, callback) {
	this.findByEmail(email, function (err, user) {
		if (err) { return callback(err, null) }
		if (user) {
			crypto.pbkdf2(plaintext, user.salt, ITERATIONS, KEYLEN, onComplete)
		} else {
			callback(null, null)
		}

		function onComplete(err, key) {
			if (err) callback(err, null)
			if (user.hash === key.toString('base64')) {
				callback(null, user.id)
			} else {
				callback(null, null)
			}
		}
	})
}

module.exports = mongoose.model('User', schema)

{% endcodeblock %}

All three resource types follow a similar implementation pattern. Endpoints are
defined and exported as an Express application. The `validate` middleware is
looked up and used to authenticate requests (in the context of the task
resource shown below, all endpoints are authenticated). In support for the
offline mobile use case, new resources are created with a `PUT` request based
on a globally unique identifer created by a client. Additionally, the
`If-Unmodified-Since` conditional header is used to detect potential conflicts
that would arise from use by multiple clients (e.g. mobile and web). Finally,
the `PATCH` method is implemented using JSON Patch ([RPC 6902][6]) to optimize
client requests to modify a resource.

The *tasks.js* file below is representative of the implementation approach for
all resource endpoints:

{% codeblock lang:javascript tasks.js %}

var app = require('express')()
var jsonpatch = require('json-patch')
var Task = require('./models/task')

// validate all requests have an access token
app.all('*', function (req, res, next) {
	var validate = req.app.get('validate')
	validate(req, res, next)
})

// return all incomplete tasks for a user
app.get('/', function (req, res, next) {
	var modifiedSince

	if (req.query['modified-since']) {
		modifiedSince = new Date(req.query['modified-since'])
	}

	// return incomplete tasks modified since the specified date
	Task.findByIncomplete(req.user, modifiedSince, function (err, tasks) {
		if (err) { return next(err) }
		res.json(tasks)
	})
})

// lookup a task based on the id within the uri
app.param('id', function (req, res, next, id) {
	Task.findById(id, function (err, task) {
		if (err) { return next(err) }
		if (!task) {
			if (req.method === 'PUT') {
				next()
			} else {
				res.send(404)
			}
		} else {
			// check that the requesting user is authorized
			if (task.user.equals(req.user.id)) {
				req.task = task
				next()
			} else {
				res.send(403)
			}
		}
	})
})

// return the task with the specified id
app.get('/:id', function (req, res) {
	res.json(req.task)
})

// create or replace a task with the specified id
app.put('/:id', function (req, res, next) {
	var unmodifiedSince = req.get('if-unmodified-since')
	
	// create new task
	if (!req.task) {
		if (unmodifiedSince) { return res.send(404) }

		req.body._id = req.params.id
		req.body.user = req.user.id
		req.body.modified = new Date() 
		Task.create(req.body, function (err) {
			if (err) { return res.send(400) }
			res.status(201)
			.set('last-modified', req.body.modified)
			.send()
		})
	
	// update existing task if conditional is provided
	} else {
		if (!unmodifiedSince) { return res.send(428) }
		if (isModified(req.task, unmodifiedSince)) { return res.send(412) }

		var task = new Task(req.body)
		task.validate(function (err) {
			if (err) { return res.send(400) }
			delete req.body._id
			delete req.body.user
			req.body.modified = new Date()
			Task.update({ _id: req.task.id }, req.body, function (err) {
				if (err) { return next(err) }
				res.status(204)
				.set('last-modified', req.body.modified)
				.send()
			})
		})
	}
})

// update a task with partial JSON data based on RFC 6902
app.patch('/:id', function (req, res, next) {
	var unmodifiedSince = req.get('if-unmodified-since')

	if (!req.is('application/json-patch+json')) {
		return res.send(415)
	}
 
	if (!unmodifiedSince) { return res.send(428) }
	if (isModified(req.task, unmodifiedSince)) { return res.send(412) }

	try {
		jsonpatch.apply(req.task, req.body)
		req.task.modified = new Date()
		req.task.save(function (err) {
			if (err) { return next(err) }
			res.status(204)
			.set('last-modified', req.task.modified)
			.send()
		})
	} catch (e) {
		res.send(400)
	}
})

// compare dates stripping milliseconds 
function isModified(task, unmodifiedSince) {
	var modified = new Date(task.modified).setMilliseconds(0)
	return modified > new Date(unmodifiedSince)
}

module.exports = app

{% endcodeblock %}

After the implementation of all resource endpoints, it was clear that much of
the logic could be abstracted if time had permitted. Ideally, I would have been
interested in exploring a declarative approach to endpoints with a more robust framework for supporting data syncronization across clients.

[1]: https://4dashes.com
[2]: http://nodejs.org
[3]: http://expressjs.com
[4]: http://shawn.dahlen.me/blog/2013/03/18/setup-node-dot-js-servers-within-a-load-balanced-configuration/
[5]: http://mongoosejs.com
[6]: http://mongodb.org
[7]: https://tools.ietf.org/html/rfc6902
