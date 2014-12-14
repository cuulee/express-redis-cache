express-redis-cache
===================

Easily cache pages of your app using Express and Redis. Could be used without Express. 

# Install

    npm install express-redis-cache
    
`express-redis-cache` ships with a CLI utility you can invoke from the console. In order to use it, install `express-redis-cache` globally (might require super user privileges):

    npm install -g express-redis-cache
    
# Upgrade

Read [this](https://github.com/co2-git/express-redis-cache/blob/master/CHANGELOG.md) if you are upgrading from 0.0.8 to 0.1.0,

# Usage

Just use it as a middleware in the stack of the route you want to cache.

```js
var app = express();
var cache = require('express-redis-cache')();

// replace
app.get('/',
    function (req, res)  { ... });

// by
app.get('/',
    cache.route(),
    function (req, res)  { ... });
```
    
This will check if there is a cache entry for this route. If not. it will cache it and serve the cache next time route is called.

# Redis connexion info

By default, redis-express-cache connects to Redis using localhost as host and nothing as port (using Redis default port). To use different port or host, declare them when you require express-redis-cache.

```js
var cache = require('express-redis-cache')({
    host: String, port: Number
    });
```
        
You can pass a Redis client as well:

```js
require('express-redis-cache')({ client: require('redis').createClient() })
```

# Events

The module emits the following events:

## error

You can catch errors by adding a listener:

```js
cache.on('error', function (error) {
    throw new Error('Cache error!');
});
```

## message

`express-redis-cache` logs some information at runtime. You can access it like this:

```js
cache.on('message', function (message) {
    // ...
});
```

## connected

Emitted when the client is connected to Redis server

```js
cache.on('connected', function () {
    console.log('Ready to party!');
});
```
    
# Models

## The Entry Model

A cache entry.

```js
new Entry ({
    body:    String // the content of the cache
    touched: Number // last time cache was set (created or updated) as a Unix timestamp
    expire:  Number // the seconds cache entry lives (-1 if does not expire)
})
```
    
## The Options Object

This is the object passed as an argument to the constructor.

|       | host    | port  | prefix  | expire  | client |
| ------------- |----------|-------|----------|--------|------|
| type          | String    | Number | String | Number | RedisClient |
| required      | false     |   false | false | false  | false |
| default       | undefined  |    undefined | require('express-redis-cache/package.json').config.prefix | undefined | require('redis').createClient({ host: cache.host, port: cache.port }) |
| description   | Redis server host  |    Redis server port | Default prefix to append to entry names | Default life time of entries in seconds | A Redis client |

```js
new Schema({
  "host": {
    type: String,
    required: false,
    default: undefined,
    description: "Redis server host",
    example: function () {
      /** Setter */
      cache({ host: 'my-host.com' });
      
      /** Getter */
      console.log('host', cache.host);
      
      // Host is not changable at runtime
      // Create different clients instead:
      
      var cache = require('express-redis-cache');
      var client1 = cache({ host: 'my.app.com', port: 6537 });
      var client2 = cache({ host: 'cloud.my.app.com', port: 7433 });
      
      app.get('/1', client1.route(), function (req, res) { res.send('1'); });
      app.get('/2', client2.route(), function (req, res) { res.send('2'); });
      }
    },
      
  "port": {
    type: Number,
    required: false,
    default: undefined,
    description: "Redis server port",
    example: function () {
      /** Setter */
      cache({ port: 18000 });
      
      /** Getter */
      console.log('port', cache.port);
      }
    },
      
  "prefix": {
    type: String,
    required: false,
    default: require("express-redis-cache/package.json").config.prefix,
    description: "Default prefix to prepend to each entry name",
    example: function () {
      /** Setter */
      cache({ prefix: "my-prefix" }); // initiate new client and set "my-prefix" as default prefix
      
      cache.prefix = "my-cool-prefix"; // client's default prefix changed at runtime
      
      /** Getter */
      console.log('prefix', cache.prefix);
  
      // Using Express
      app.get('/my-page',
        // Cache will be saved as "my-cool-prefix:my-page"
        cache.route(),
        function (req, res, next) {
          res.send('Hey! I am a cool page!');
        });
        
      // Using API
      // The entry will be saved under the name "my-cool-prefix:test-entry"
      cache.add('test-entry', 'The quick brown fox jumps over the lazy dog', 'text/plain', 6,
        function (error, added) {});
        
      // You can overwrite default prefix using an object
      
      // With Express
      cache.route({ prefix: 'another-prefix' });
      
      // With API
      cache.add({ name: 'test-entry', prefix: 'prefix-2' });
      
      // You can also choose not to use prefix. The snippet above could also be written as such:
      cache.add({ name: 'prefix-2:test-entry', prefix: false });
    },
  }
);
```

    Object ConstructorOPtions {
        host:   String?     // Redis Host
        port:   Number?     // Redis port
        prefix: String?     // Cache entry name prefix,
        expire: Number?     // Default expiration time in seconds
        client: RedisClient // A Redis client of npm/redis
    }

# Commands

## Constructor

    cache( Object ConstructorOPtions? )

## Route
    
    cache.route( String name?, Number expire? )
        
    
If `name` is a string, it is a cache entry name. If it is null, the route's URI (`req.originalUrl`) will be used as the entry name.

```js
app.get('/', cache.route('home'), require('./routes/'))
// the name of the enrty will be 'home'

app.get('/about', cache.route(), require('./routes/'))
// the name of the enrty will be '/about'
```

Optionally, you can gain more naming control on defining `res.expressRedisCacheName`:

```js
// Example with using parameters
app.get('/user/:userid',
    function (req, res, next) {
        res.expressRedisCacheName = '/user/' + req.params.userid; // name of the entry
        next();
    },
    cache.route(),
    require('./routes/user')
);

// Example with using payload
app.post('/search',
    function (req, res, next) {
        res.expressRedisCacheName = '/search/' + req.body.tag; // name of the entry
        next();
    },
    cache.route(),
    require('./routes/user')
);
```

### Conditional caching

You can also introduce a logic to decide if to use the cache:

```js
app.get('/',
  
  /** Pre function to decide if to use the cache */
  
  function (req, res, next) {
    /** Dummy story: don't use the cache if user has cookie */
    res.express_redis_cache_skip = !! req.signedCookies.user;
    
    /** Continue in stack */
    next();
    },
    
  /** Express-Redis-Cache middleware */
  /** This will be skipped if user has cookie */
  
  cache.route(),
  
  /** The view middleware */
  
  function (req, res) {
    res.render('index');
    }
    
  );
```

### Set an expiration date for the cache entry

The number of seconds the cache entry will live

```js
cache.route('home', ( 60 * 5 ));
// cache will expire in 5 minutes
```
    
If you don't define an expiration date in your route but have set a default one in your constructor, the latter will be used. If you want your cache entry not to expire even though you have set a default expiration date in your constructor, do like this:

```js
cache.route('my-page', cache.FOREVER); // This entry will never expire
```


## Get the list of all cache entries

    cache.ls( Function ( Error, [Entry] ) )
    
Feed a callback with an array of the cache entry names.
    
## Get a single cache entry by name
    
    cache.get( String name, Function( Error, Entry ) )
    
## Add a new cache entry
    
    cache.add( String name, String body, Number expire?, Function( Error, Entry ) )
    
Example:

```js
cache.add('user:info', JSON.stringify({ id: 1, email: 'john@doe.com' }), 60,
    function (error, added) {
    });
```

## Delete a cache entry
    
    cache.del( String name, Function ( Error, Number deletedEntries ) )

## Get cache size for all entries
    
    cache.size( Function ( Error, Number bytes ) )
    
# Command line

We ship with a CLI. You can invoke it like this: `express-redis-cache`

# Test

    mocha --host <redis-host> --port <redis-port>

    # or

    npm test --host=<redis-host> --port=<redis-port>

Enjoy!
