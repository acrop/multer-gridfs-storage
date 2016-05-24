# Multer GridFS storage engine

[GridFS](https://docs.mongodb.com/manual/core/gridfs) storage engine for [Multer](https://github.com/expressjs/multer) to store uploaded files directly to MongoDb

## Installation

Using npm

```sh
$ npm install multer-gridfs-storage --save
```

Basic usage example:

```javascript
var express = require('express');
var multer  = require('multer');
// Create a storage object with a given configuration
var storage = require('multer-gridfs-storage')({
   url: 'mongodb://localhost:27017/database'
});
// Set multer storage engine to the newly created object
var upload = multer({ storage: storage });

var app = express()

// Upload your files as usual
var sUpload = upload.single('avatar');
app.post('/profile', sUpload, function (req, res, next) { 
    /*....*/ 
})

var arrUpload = upload.array('photos', 12);
app.post('/photos/upload', arrUpload, function (req, res, next) {
    /*....*/ 
})

var fUpload = upload.fields([{ name: 'avatar', maxCount: 1 }, { name: 'gallery', maxCount: 8 }])
app.post('/cool-profile', fUpload, function (req, res, next) {
    /*....*/ 
})
```

## API

### module(options) : function

The module returns a function that can be invoked with options to create a Multer storage engine.

The options parameter is an object with the following properties.

#### gfs

Type: **Object**

Required if `url` option is not present

The [gridfs-stream](https://github.com/aheckmann/gridfs-stream/) instance to use.

If this option is provided files are stored using this stream. The connection should be opened or
the module might fail to store incoming files since no connection test is made. 
This option is useful when you have an existing GridFS object and want to reuse it to upload your files.

Example:

```javascript
var Grid = require('gridfs-stream');
var mongo = require('mongodb');
var GridFsStorage = require('multer-gridfs-storage');

var db = new mongo.Db('database', new mongo.Server("127.0.0.1", 27017));

db.open(function (err) {
  if (err) return handleError(err);
  var gfs = Grid(db, mongo);

  var storage = GridFsStorage({
     gfs: gfs
  });
  var upload = multer({ storage: storage });
})
```

#### url

Type: **String**

Required if `gfs` option is not present

The mongodb connection uri. 

A string pointing to the database used to store the incoming files. This must be a standard mongodb [connection string](https://docs.mongodb.com/manual/reference/connection-string).

With this option the module will create a GridFS stream instance for you instead. 

Note: If the `gfs` option is specified this setting is ignored.

Example:

```javascript
var storage = require('multer-gridfs-storage')({
    url: 'mongodb://localhost:27017/database'
});
var upload = multer({ storage: storage });
```

Creating the connection string from a host, port and database object

```javascript
var url = require('url');

var settings = {
    host: '127.0.0.1',
    port: 27017,
    database: 'database'
};

var connectionString = url.format({
    protocol: 'mongodb',
    slashes: true,
    hostname: settings.host,
    port: settings.port,
    pathname: settings.database
});

var storage = require('multer-gridfs-storage')({
    url: connectionString
});
var upload = multer({ storage: storage });
```

#### filename

Type: **Function**

Not required

A function to control the file naming in the database. Is invoked with
the parameters `req`, `file` and `callback`, in that order, like all the Multer configuration
functions.

By default this module behaves exactly like the default Multer disk storage does.
It generates a 16 bytes long name in hexadecimal format with no extension for the file
to guarantee that there are very low probabilities of naming collisions. You can override this 
by passing your own function.

Example:

```javascript
var storage = require('multer-gridfs-storage')({
   url: 'mongodb://localhost:27017/database',
   filename: function(req, file, cb) {
      cb(null, file.originalname);
   }
});
var upload = multer({ storage: storage });
```

In this example the original filename and extension in the user's computer are used 
to name each of the uploaded files. Please note that this will not guarantee that file
names are unique and you might have files with duplicate names or overwritten in your database.

```javascript
var crypto = require('crypto');
var path = require('path');

var storage = require('multer-gridfs-storage')({
   url: 'mongodb://localhost:27017/database',
   filename: function(req, file, cb) {
       crypto.randomBytes(16, function (err, raw) {
           cb(err, err ? undefined : raw.toString('hex') + path.extname(file.originalname));
       });
   }
});
var upload = multer({ storage: storage });
```

To ensure that names are unique a random name is used and the file extension is preserved as well. 
You could also use the user's file name plus a timestamp to generate unique names.

#### identifier

Type: **Function**

Not required

A function to control the unique identifier of the file. 

This function is invoked as all the others with the `req`, `file` and `callback` 
parameters and can be used to change the default identifier ( the `_id` property)
created by MongoDb. Please note that you must garantee that this value is unique 
otherwise you will get an error.

To use the default generated identifier invoke the callback with a [falsey](http://james.padolsey.com/javascript/truthy-falsey/) value like `null` or `undefined`.  

Example:

```javascript
var path = require('path');
var storage = require('multer-gridfs-storage')({
   url: 'mongodb://localhost:27017/database',
   identifier: function(req, file, cb) {
      var id = '';
      for (var i = 0; i < 24; i++) {
          id = id + Math.floor(Math.random()*16777216).toString(16);
      }
      cb(null, id);
   }
});
var upload = multer({ storage: storage });
```

In this example a random hex value between 0 and ffffff is used for the file identifier. Normaly you shouldn't use this function
unless you want granular control of your file ids because autogenerated identifiers are garanteed to be unique.

#### metadata

Type: **Function**

Not required

A function to control the metadata associated to the file. 

This function is called with the `req`, `file` and `callback` parameters and is used
to store metadata with the file. 

By default the stored metadata value for uploaded files is `null`.

Example:

```javascript
var storage = require('multer-gridfs-storage')({
   url: 'mongodb://localhost:27017/database',
   metadata: function(req, file, cb) {
      cb(null, req.body);
   }
});
var upload = multer({ storage: storage });
```

In this example the contents of the request body are stored with the file. Please note that
this is only for illustrative purposes. If your users send passwords or other sensitive data in the request 
those will be stored unencrypted in the database as well, inside the metadata of the file.

#### log

Type: **Boolean**
Default: `false`

Not required

Enable or disable logging.

By default the module will not output anything. Set this option
to true to log when the connection is opened, files are stored or an error occurs.
The console is used to log information to `stdout` or `stderr`

#### logLevel

Not required

The events to be logged out. Only applies if logging is enabled.

Type: **string**
Default: `'file'`
Possible values: `'all'` and `'file'`

If set to `'all'` and the connection is established using the `url` option 
some events are attached to the mongoDb connection to output to `stdout` and `stderr`
when the connection is established and files are uploaded.

If set to `'file'` only successful file uploads will be registered. This setting applies to
both the `gfs` and the `url` configuration.

This option is useful when you also want to log when the connection is opened
or an error has occurs. Setting it to `'all'` and using the `gfs` option
has no effect and behaves like if it were set to `'file'`.

### File information

Each file in `req.file` and `req.files` contain the following properties in addition
to the ones that Multer create by default.

Key | Description
--- | --
`filename` | The name of the file within the database
`metadata` | The stored metadata of the file
`id` | The id of the stored file
`grid` | The GridFS information of the stored file

To see all the other properties of the file object check the Multer's [documentation](https://github.com/expressjs/multer#file-information).

## Events

Each storage object is also a standard Node.js Event Emitter. This is done to ensure that some 
objects are made available after they are instantiated since some options
generate asyncronous database connections that needs to be established before those objects
are referenced.

#### Event: `'connection'`

Only available when the storage is created with the `url` option.

This event is emited when the MongoDb connection is opened. This is useful 
if you want access to the internal GridFS instance. This instance is exposed 
in the `storage.gfs` property but until the connection is made the value of 
this property is `null`.

The handler for this event is a function with two parameters `gfs` and `db`. Those parameters are
the newly created GridFS instance and the native MongoDb instance respectively.

Example:

```javascript
var url = 'mongodb://localhost:27017/database';
var storage = require('multer-gridfs-storage')({
   url: url
});
var upload = multer({ storage: storage });

storage.once('connection', function(gfs, db) {
   // Do something with the GridFS or the MongoDb instance
   console.log('MongoDb connected in url ' + url);
});
```

This event is only triggered once. Note that if you only want to log events there is an api option for that

#### Event: `'file'`

This event is emited everytime a new file is stored in the db. This is useful when you have
a custom logging mechanism and want to record every uploaded file.

```javascript
var storage = require('multer-gridfs-storage')({
   url: 'mongodb://localhost:27017/database'
});
var upload = multer({ storage: storage });

storage.on('file', function(file) {
   // Do something with the file
   console.log('New file uploaded ' + file.originalname);
});
```

## Test

To run the test suite, first install the dependencies, then run `npm test`:

```bash
$ npm install
$ npm test
```

Tests are written with [mocha](https://mochajs.org/) and [chai](http://chaijs.com/). You can also run the tests with:

In case you don't have mocha installed globally run

```bash
$ npm install mocha -g
```

and then run

```bash
$ mocha
```

## License

MIT