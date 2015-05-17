---
layout: post
title: "RESTful API With Node.js + MongoDB"
date: 2013-09-12 11:50:00 +0400
---


I am mobile app developer. I need some backend service to manage user data in remote databases quite frequently. Of course, I could use some BaaS ([Parse](https://parse.com/), [Backendless](https://backendless.com/), etc…). But good own solution is always a more convenient and practical choice.

I decided to explore completely unknown technologies, which are now very popular and are positioned as easily assimilated by newcomers and do not require in-depth knowledge and experience to implement large-scale projects.

This article will consider building a REST API for mobile applications using [Node.js](http://nodejs.org/) and [Express.js](http://expressjs.com/) framework with [Mongoose.js](http://mongoosejs.com/) for working with [MongoDB](http://www.mongodb.org/). For access control we'll use OAuth 2.0, with the help of [OAuth2orize](https://github.com/jaredhanson/oauth2orize) and [Passport.js](http://passportjs.org/).

<!-- more -->

##Contents

1. [Node.js + Express.js, simple web-server](#Step1)
2. [Error handling](#Step2)
3. [RESTful API endpoints, CRUD](#Step3)
4. [MongoDB & Mongoose.js](#Step4)
5. [Access control — OAuth 2.0, Passport.js](#Step5)

I am working on OSX. IDE is [JetBrains WebStorm](http://www.jetbrains.com/webstorm/).

You can grab a final project from [GitHub](https://github.com/ealeksandrov/NodeAPI). Run `npm install` in projects folder for installation of all required modules.

##<a name="Step1"></a>1. Node.js + Express.js, simple web-server
Node.js has a non-blocking i/o. That's great for API services which will be accessed by many clients. Express.js is an advanced, lightweight framework that allows us to quickly describe all the needed API endpoints. It also supports many useful modules.

Let's create a new project with a single file `server.js`. Since the application will rely on Express.js, we'll  install it. Installing third-party modules through Node Package Manager is simple: `npm install modulename` in the project folder.

{% highlight bash %}
cd NodeAPI
npm i express
{% endhighlight %}

Express will be installed in `node_modules` folder. Now connect it to the application:

{% highlight js %}
var express = require('express');
var app = express();

app.listen(1337, function(){
    console.log('Express server listening on port 1337');
});
{% endhighlight %}

Run the application through the IDE or console (`node server.js`). This code will create a web server on `localhost:1337`. It now displays a message `Cannot GET /`. This is because we haven't configured any routes yet. Next, let's create some routes and configure basic settings of Express.

{% highlight js %}
var express         = require('express');
var path            = require('path'); // path parsing module
var app = express();

app.use(express.favicon()); // use standard favicon
app.use(express.logger('dev')); // log all requests
app.use(express.bodyParser()); // JSON parsing
app.use(express.methodOverride()); // HTTP PUT and DELETE support
app.use(app.router); // simple route management
app.use(express.static(path.join(__dirname, "public"))); // starting static fileserver, that will watch `public` folder (in our case there will be `index.html`)

app.get('/api', function (req, res) {
    res.send('API is running');
});

app.listen(1337, function(){
    console.log('Express server listening on port 1337');
});
{% endhighlight %}

Now `localhost:1337/api` returns our message (handled by `app.router`). `localhost:1337` displays `index.html` (handled by `express.static()`).

Next step is error handling.

##<a name="Step2"></a>2. Error handling
First connect a cool logging module [Winston](https://github.com/flatiron/winston). We will make a wrapper for it. Run `npm i winston` in project root, then create a folder named `libs/` with `log.js` there.

{% highlight js %}
var winston = require('winston');

function getLogger(module) {
    var path = module.filename.split('/').slice(-2).join('/'); //using filename in log statements
	
    return new winston.Logger({
        transports : [
            new winston.transports.Console({
                colorize:   true,
                level:      'debug',
                label:      path
            })
        ]
    });
}

module.exports = getLogger;
{% endhighlight %}

We created one transport for logging – console. You can separately sort and store logs in different transports, such as a database or file. Connect the logger to `server.js`.


{% highlight js %}
var express         = require('express');
var path            = require('path');
var log             = require('./libs/log')(module);
var app = express();

app.use(express.favicon());
app.use(express.logger('dev'));
app.use(express.bodyParser());
app.use(express.methodOverride());
app.use(app.router);
app.use(express.static(path.join(__dirname, "public")));

app.get('/api', function (req, res) {
    res.send('API is running');
});

app.listen(1337, function(){
    log.info('Express server listening on port 1337');
});
{% endhighlight %}

Info message now passes through Winston to its transport – the console.

Next – 404 and 500 error handling.

{% highlight js %}
app.use(function(req, res, next){
    res.status(404);
    log.debug('Not found URL: %s',req.url);
    res.send({ error: 'Not found' });
    return;
});

app.use(function(err, req, res, next){
    res.status(err.status || 500);
    log.error('Internal error(%d): %s',res.statusCode,err.message);
    res.send({ error: err.message });
    return;
});

app.get('/ErrorExample', function(req, res, next){
    next(new Error('Random error!'));
});
{% endhighlight %}

Now, if there are no suitable routes, Express will return our message. When an internal error occurs, it will be passed to the handler, you can check it on `localhost:1337/ErrorExample`.

##<a name="Step3"></a>3. RESTful API endpoints, CRUD
Let's add a way to handle some "articles". Implementation will be empty for now, it will be fixed in the next step, after connecting to a database.

{% highlight js %}
app.get('/api/articles', function(req, res) {
    res.send('This is not implemented now');
});

app.post('/api/articles', function(req, res) {
    res.send('This is not implemented now');
});

app.get('/api/articles/:id', function(req, res) {
    res.send('This is not implemented now');
});

app.put('/api/articles/:id', function (req, res){
    res.send('This is not implemented now');    
});

app.delete('/api/articles/:id', function (req, res){
    res.send('This is not implemented now');
});
{% endhighlight %}

To test post/put/delete I am advising a wonderful wrapper over cURL - [httpie](https://github.com/jkbr/httpie). I will give examples of requests by using this tool.

##<a name="Step4"></a>4. MongoDB & Mongoose.js
Choosing a database, I was guided by the desire to once again explore something new. [MongoDB](http://www.mongodb.org/) - the most popular NoSQL document-oriented database. [Mongoose.js](http://mongoosejs.com/) - wrapper, allowing to create comfortable and functional schema documents.

Download and install [MongoDB](http://www.mongodb.org/downloads). Than, install Mongoose: `npm i mongoose`. I will put database interaction in separate module: `libs/mongoose.js`.

{% highlight js %}
var mongoose    = require('mongoose');
var log         = require('./log')(module);

mongoose.connect('mongodb://localhost/test1');
var db = mongoose.connection;

db.on('error', function (err) {
    log.error('connection error:', err.message);
});
db.once('open', function callback () {
    log.info("Connected to DB!");
});

var Schema = mongoose.Schema;

// Schemas
var Images = new Schema({
    kind: {
        type: String,
        enum: ['thumbnail', 'detail'],
        required: true
    },
    url: { type: String, required: true }
});

var Article = new Schema({
    title: { type: String, required: true },
    author: { type: String, required: true },
    description: { type: String, required: true },
    images: [Images],
    modified: { type: Date, default: Date.now }
});

// validation
Article.path('title').validate(function (v) {
    return v.length > 5 && v.length < 70;
});

var ArticleModel = mongoose.model('Article', Article);

module.exports.ArticleModel = ArticleModel;
{% endhighlight %}

In this file, connection to the database is implemented and object scheme are declared. Articles will contain picture objects. A variety of complex validation can be implemented here as well.

I will use [nconf](https://github.com/flatiron/nconf) module to store there database path. Also, let's move a server port number there. The module is installed by `npm i nconf`. Custom wrapper will be `libs/config.js`.

{% highlight js %}
var nconf = require('nconf');

nconf.argv()
    .env()
    .file({ file: './config.json' });

module.exports = nconf;
{% endhighlight %}

All the settings will be stored in `config.json` at the project's root.

{% highlight json %}
{
    "port" : 1337,
    "mongoose": {
        "uri": "mongodb://localhost/test1"
    }
}
{% endhighlight %}

`mongoose.js` changes:

{% highlight js %}
var config      = require('./config');

mongoose.connect(config.get('mongoose:uri'));
{% endhighlight %}

`server.js` changes:

{% highlight js %}
var config = require('./libs/config');

app.listen(config.get('port'), function(){
    log.info('Express server listening on port ' + config.get('port'));
});
{% endhighlight %}

Let's add CRUD actions in existing routes.
{% highlight js %}
var ArticleModel    = require('./libs/mongoose').ArticleModel;

app.get('/api/articles', function(req, res) {
    return ArticleModel.find(function (err, articles) {
        if (!err) {
            return res.send(articles);
        } else {
            res.statusCode = 500;
            log.error('Internal error(%d): %s',res.statusCode,err.message);
            return res.send({ error: 'Server error' });
        }
    });
});

app.post('/api/articles', function(req, res) {
    var article
 = new ArticleModel({
        title: req.body.title,
        author: req.body.author,
        description: req.body.description,
        images: req.body.images
    });

    article.save(function (err) {
        if (!err) {
            log.info("article created");
            return res.send({ status: 'OK', article:article });
        } else {
            console.log(err);
            if(err.name == 'ValidationError') {
                res.statusCode = 400;
                res.send({ error: 'Validation error' });
            } else {
                res.statusCode = 500;
                res.send({ error: 'Server error' });
            }
            log.error('Internal error(%d): %s',res.statusCode,err.message);
        }
    });
});

app.get('/api/articles/:id', function(req, res) {
    return ArticleModel.findById(req.params.id, function (err, article) {
        if(!article) {
            res.statusCode = 404;
            return res.send({ error: 'Not found' });
        }
        if (!err) {
            return res.send({ status: 'OK', article:article });
        } else {
            res.statusCode = 500;
            log.error('Internal error(%d): %s',res.statusCode,err.message);
            return res.send({ error: 'Server error' });
        }
    });
});

app.put('/api/articles/:id', function (req, res){
    return ArticleModel.findById(req.params.id, function (err, article) {
        if(!article) {
            res.statusCode = 404;
            return res.send({ error: 'Not found' });
        }

        article.title = req.body.title;
        article.description = req.body.description;
        article.author = req.body.author;
        article.images = req.body.images;
        return article.save(function (err) {
            if (!err) {
                log.info("article updated");
                return res.send({ status: 'OK', article:article });
            } else {
                if(err.name == 'ValidationError') {
                    res.statusCode = 400;
                    res.send({ error: 'Validation error' });
                } else {
                    res.statusCode = 500;
                    res.send({ error: 'Server error' });
                }
                log.error('Internal error(%d): %s',res.statusCode,err.message);
            }
        });
    });
});

app.delete('/api/articles/:id', function (req, res){
    return ArticleModel.findById(req.params.id, function (err, article) {
        if(!article) {
            res.statusCode = 404;
            return res.send({ error: 'Not found' });
        }
        return article.remove(function (err) {
            if (!err) {
                log.info("article removed");
                return res.send({ status: 'OK' });
            } else {
                res.statusCode = 500;
                log.error('Internal error(%d): %s',res.statusCode,err.message);
                return res.send({ error: 'Server error' });
            }
        });
    });
});
{% endhighlight %}

All operations are very clear, thanks to Mongoose and self-explanatory scheme. Now, before running our `node.js`, we need to run MongoDB server: `mongod`. `mongo` - is a client utility for working with the database, the service itself is `mongod`.

Request examples using httpie:

{% highlight bash %}
http POST http://localhost:1337/api/articles title=TestArticle author='John Doe' description='lorem ipsum dolar sit amet' images:='[{"kind":"thumbnail", "url":"http://habrahabr.ru/images/write-topic.png"}, {"kind":"detail", "url":"http://habrahabr.ru/images/write-topic.png"}]'

http http://localhost:1337/api/articles

http http://localhost:1337/api/articles/52306b6a0df1064e9d000003

http PUT http://localhost:1337/api/articles/52306b6a0df1064e9d000003 title=TestArticle2 author='John Doe' description='lorem ipsum dolar sit amet' images:='[{"kind":"thumbnail", "url":"http://habrahabr.ru/images/write-topic.png"}, {"kind":"detail", "url":"http://habrahabr.ru/images/write-topic.png"}]'

http DELETE http://localhost:1337/api/articles/52306b6a0df1064e9d000003
{% endhighlight %}

You can checkout the project at this stage from [Github](https://github.com/ealeksandrov/NodeAPI/tree/e8764a97f9c70fb6eae102fda7237e745d9e99ac).

##<a name="Step5"></a>5. Access control — OAuth 2.0, Passport.js
We will use OAuth 2. Perhaps this is redundant, but in the future, this approach facilitates an integration with other OAuth-providers.

Module [Passport.js](http://passportjs.org/) will be responsible for access control. For OAuth2 server, I will use handy solution from the same author - [OAuth2orize](https://github.com/jaredhanson/oauth2orize). Access tokens will be stored in MongoDB.

First you need to install all the required modules:

* Faker
* oauth2orize
* passport
* passport-http
* passport-http-bearer
* passport-oauth2-client-password

Then, you need to add mongoose.js scheme for users and tokens:

{% highlight js %}
var crypto = require('crypto');

// User
var User = new Schema({
    username: {
        type: String,
        unique: true,
        required: true
    },
    hashedPassword: {
        type: String,
        required: true
    },
    salt: {
        type: String,
        required: true
    },
    created: {
        type: Date,
        default: Date.now
    }
});

User.methods.encryptPassword = function(password) {
    return crypto.createHmac('sha1', this.salt).update(password).digest('hex');
    //more secure – return crypto.pbkdf2Sync(password, this.salt, 10000, 512);
};

User.virtual('userId')
    .get(function () {
        return this.id;
    });

User.virtual('password')
    .set(function(password) {
        this._plainPassword = password;
        this.salt = crypto.randomBytes(32).toString('hex');
        //more secure - this.salt = crypto.randomBytes(128).toString('hex');
        this.hashedPassword = this.encryptPassword(password);
    })
    .get(function() { return this._plainPassword; });


User.methods.checkPassword = function(password) {
    return this.encryptPassword(password) === this.hashedPassword;
};

var UserModel = mongoose.model('User', User);

// Client
var Client = new Schema({
    name: {
        type: String,
        unique: true,
        required: true
    },
    clientId: {
        type: String,
        unique: true,
        required: true
    },
    clientSecret: {
        type: String,
        required: true
    }
});

var ClientModel = mongoose.model('Client', Client);

// AccessToken
var AccessToken = new Schema({
    userId: {
        type: String,
        required: true
    },
    clientId: {
        type: String,
        required: true
    },
    token: {
        type: String,
        unique: true,
        required: true
    },
    created: {
        type: Date,
        default: Date.now
    }
});

var AccessTokenModel = mongoose.model('AccessToken', AccessToken);

// RefreshToken
var RefreshToken = new Schema({
    userId: {
        type: String,
        required: true
    },
    clientId: {
        type: String,
        required: true
    },
    token: {
        type: String,
        unique: true,
        required: true
    },
    created: {
        type: Date,
        default: Date.now
    }
});

var RefreshTokenModel = mongoose.model('RefreshToken', RefreshToken);

module.exports.UserModel = UserModel;
module.exports.ClientModel = ClientModel;
module.exports.AccessTokenModel = AccessTokenModel;
module.exports.RefreshTokenModel = RefreshTokenModel;
{% endhighlight %}

Virtual property `password` is an example of how mongoose model can embed convenient logic. Hashing algorithms and salt is not in this article's scope, so we won't dig into the details of the implementation.

DB objects:

1. User – a user who has a name, password hash and a salt.
2. Client – a client application which requests access on behalf of a user, has a name and a secret code.
3. AccessToken – token (type of bearer), issued to the client application, limited by time.
4. RefreshToken – another type of token allows you to request a new bearer-token without re-request a password from the user.

Add token lifetime to `config.json`:

{% highlight json %}
{
    "port" : 1337,
    "security": {
        "tokenLife" : 3600
    },
    "mongoose": {
        "uri": "mongodb://localhost/testAPI"
    }
}
{% endhighlight %}

I implemented OAuth2 server and authorization logic in separate modules. In `auth.js` passport.js strategies are described. We connect 3 strategies – 2 for OAuth2 username-password flow and one to check the token.

{% highlight js %}
var config                  = require('./config');
var passport                = require('passport');
var BasicStrategy           = require('passport-http').BasicStrategy;
var ClientPasswordStrategy  = require('passport-oauth2-client-password').Strategy;
var BearerStrategy          = require('passport-http-bearer').Strategy;
var UserModel               = require('./mongoose').UserModel;
var ClientModel             = require('./mongoose').ClientModel;
var AccessTokenModel        = require('./mongoose').AccessTokenModel;
var RefreshTokenModel       = require('./mongoose').RefreshTokenModel;

passport.use(new BasicStrategy(
    function(username, password, done) {
        ClientModel.findOne({ clientId: username }, function(err, client) {
            if (err) { return done(err); }
            if (!client) { return done(null, false); }
            if (client.clientSecret != password) { return done(null, false); }

            return done(null, client);
        });
    }
));

passport.use(new ClientPasswordStrategy(
    function(clientId, clientSecret, done) {
        ClientModel.findOne({ clientId: clientId }, function(err, client) {
            if (err) { return done(err); }
            if (!client) { return done(null, false); }
            if (client.clientSecret != clientSecret) { return done(null, false); }

            return done(null, client);
        });
    }
));

passport.use(new
 BearerStrategy(
    function(accessToken, done) {
        AccessTokenModel.findOne({ token: accessToken }, function(err, token) {
            if (err) { return done(err); }
            if (!token) { return done(null, false); }

            if( Math.round((Date.now()-token.created)/1000) > config.get('security:tokenLife') ) {
                AccessTokenModel.remove({ token: accessToken }, function (err) {
                    if (err) return done(err);
                });
                return done(null, false, { message: 'Token expired' });
            }

            UserModel.findById(token.userId, function(err, user) {
                if (err) { return done(err); }
                if (!user) { return done(null, false, { message: 'Unknown user' }); }

                var info = { scope: '*' }
                done(null, user, info);
            });
        });
    }
));
{% endhighlight %}

`oauth2.js` is responsible for the issuance and renewal of the token. One token exchange strategy is for username-password flow, another is to refresh tokens.

{% highlight js %}
var oauth2orize         = require('oauth2orize');
var passport            = require('passport');
var crypto              = require('crypto');
var config              = require('./config');
var UserModel           = require('./mongoose').UserModel;
var ClientModel         = require('./mongoose').ClientModel;
var AccessTokenModel    = require('./mongoose').AccessTokenModel;
var RefreshTokenModel   = require('./mongoose').RefreshTokenModel;

// create OAuth 2.0 server
var server = oauth2orize.createServer();

// Exchange username & password for an access token.
server.exchange(oauth2orize.exchange.password(function(client, username, password, scope, done) {
    UserModel.findOne({ username: username }, function(err, user) {
        if (err) { return done(err); }
        if (!user) { return done(null, false); }
        if (!user.checkPassword(password)) { return done(null, false); }

        RefreshTokenModel.remove({ userId: user.userId, clientId: client.clientId }, function (err) {
            if (err) return done(err);
        });
        AccessTokenModel.remove({ userId: user.userId, clientId: client.clientId }, function (err) {
            if (err) return done(err);
        });

        var tokenValue = crypto.randomBytes(32).toString('hex');
        var refreshTokenValue = crypto.randomBytes(32).toString('hex');
        var token = new AccessTokenModel({ token: tokenValue, clientId: client.clientId, userId: user.userId });
        var refreshToken = new RefreshTokenModel({ token: refreshTokenValue, clientId: client.clientId, userId: user.userId });
        refreshToken.save(function (err) {
            if (err) { return done(err); }
        });
        var info = { scope: '*' }
        token.save(function (err, token) {
            if (err) { return done(err); }
            done(null, tokenValue, refreshTokenValue, { 'expires_in': config.get('security:tokenLife') });
        });
    });
}));

// Exchange refreshToken for an access token.
server.exchange(oauth2orize.exchange.refreshToken(function(client, refreshToken, scope, done) {
    RefreshTokenModel.findOne({ token: refreshToken }, function(err, token) {
        if (err) { return done(err); }
        if (!token) { return done(null, false); }
        if (!token) { return done(null, false); }

        UserModel.findById(token.userId, function(err, user) {
            if (err) { return done(err); }
            if (!user) { return done(null, false); }

            RefreshTokenModel.remove({ userId: user.userId, clientId: client.clientId }, function (err) {
                if (err) return done(err);
            });
            AccessTokenModel.remove({ userId: user.userId, clientId: client.clientId }, function (err) {
                if (err) return done(err);
            });

            var tokenValue = crypto.randomBytes(32).toString('hex');
            var refreshTokenValue = crypto.randomBytes(32).toString('hex');
            var token = new AccessTokenModel({ token: tokenValue, clientId: client.clientId, userId: user.userId });
            var refreshToken = new RefreshTokenModel({ token: refreshTokenValue, clientId: client.clientId, userId: user.userId });
            refreshToken.save(function (err) {
                if (err) { return done(err); }
            });
            var info = { scope: '*' }
            token.save(function (err, token) {
                if (err) { return done(err); }
                done(null, tokenValue, refreshTokenValue, { 'expires_in': config.get('security:tokenLife') });
            });
        });
    });
}));

// token endpoint
exports.token = [
    passport.authenticate(['basic', 'oauth2-client-password'], { session: false }),
    server.token(),
    server.errorHandler()
]
{% endhighlight %}

Connect these modules with server.js:

{% highlight js %}
var oauth2 = require('./libs/oauth2');

app.use(passport.initialize());

require('./libs/auth');

app.post('/oauth/token', oauth2.token);

app.get('/api/userInfo',
    passport.authenticate('bearer', { session: false }),
        function(req, res) {
            // req.authInfo is set using the `info` argument supplied by
            // `BearerStrategy`.  It is typically used to indicate a scope of the token,
            // and used in access control checks.  For illustrative purposes, this
            // example simply returns the scope in the response.
            res.json({ user_id: req.user.userId, name: req.user.username, scope: req.authInfo.scope })
        }
);
{% endhighlight %}

For example, the access is restricted on `localhost:1337/api/userInfo`.

To check the auth logic, we should create a user and a client in our database. Use this node application, which will create the necessary objects and remove redundant from collections. It helps quickly clean the tokens and users for testing.

{% highlight js %}
var log                 = require('./libs/log')(module);
var mongoose            = require('./libs/mongoose').mongoose;
var UserModel           = require('./libs/mongoose').UserModel;
var ClientModel         = require('./libs/mongoose').ClientModel;
var AccessTokenModel    = require('./libs/mongoose').AccessTokenModel;
var RefreshTokenModel   = require('./libs/mongoose').RefreshTokenModel;
var faker               = require('Faker');

UserModel.remove({}, function(err) {
    var user = new UserModel({ username: "andrey", password: "simplepassword" });
    user.save(function(err, user) {
        if(err) return log.error(err);
        else log.info("New user - %s:%s",user.username,user.password);
    });

    for(i=0; i<4; i++) {
        var user = new UserModel({ username: faker.random.first_name().toLowerCase(), password: faker.Lorem.words(1)[0] });
        user.save(function(err, user) {
            if(err) return log.error(err);
            else log.info("New user - %s:%s",user.username,user.password);
        });
    }
});

ClientModel.remove({}, function(err) {
    var client = new ClientModel({ name: "OurService iOS client v1", clientId: "mobileV1", clientSecret:"abc123456" });
    client.save(function(err, client) {
        if(err) return log.error(err);
        else log.info("New client - %s:%s",client.clientId,client.clientSecret);
    });
});
AccessTokenModel.remove({}, function (err) {
    if (err) return log.error(err);
});
RefreshTokenModel.remove({}, function (err) {
    if (err) return log.error(err);
});

setTimeout(function() {
    mongoose.disconnect();
}, 3000);
{% endhighlight %}

If you used `dataGen.js` following commands to test authorization will fit you well. Let me remind you that I am using [httpie](https://github.com/jkbr/httpie).

{% highlight bash %}
http POST http://localhost:1337/oauth/token grant_type=password client_id=mobileV1 client_secret=abc123456 username=andrey password=simplepassword

http POST http://localhost:1337/oauth/token grant_type=refresh_token client_id=mobileV1 client_secret=abc123456 refresh_token=TOKEN

http http://localhost:1337/api/userinfo Authorization:'Bearer TOKEN'
{% endhighlight %}

__Attention!__ On production always use HTTPS, it is implicit in OAuth 2 specification. And do not forget to do correct password hashing.
Let me remind that you can find the working example at the repository on [GitHub](https://github.com/ealeksandrov/NodeAPI).

To start example project, you should run `npm install` in project root, then run `mongod`, `node dataGen.js` (wait for completion), and then `node server.js`.

If any part of the article is worth to be described more clearly, please contact me by email or twitter.

To summarize, I want to say that node.js is a great, convenient server solution. MongoDB document-oriented approach is a very unusual, but certainly a useful tool. It also has a lot of features that I have not used yet. Node.js has a very large community and there are many open-source projects that come along. For example, the creator of the oauth2orize and passport.js, Jared Hanson makes wonderful projects that facilitate the implementation of the most well-protected systems.
