# express-validator

[![Build Status](https://secure.travis-ci.org/ctavan/express-validator.png)](http://travis-ci.org/ctavan/express-validator)

An [express.js]( https://github.com/visionmedia/express ) middleware for
[node-validator]( https://github.com/chriso/validator.js ).

## Installation

```
npm install express-validator
```

## Usage

```javascript
var util = require('util'),
    express = require('express'),
    expressValidator = require('express-validator'),
    app = express.createServer();

app.use(express.bodyParser());
app.use(expressValidator([options])); // this line must be immediately after express.bodyParser()!

app.post('/:urlparam', function(req, res) {

  // VALIDATION
  // checkBody only checks req.body; none of the other req parameters
  // Similarly checkParams only checks in req.params (URL params) and
  // checkQuery only checks req.query (GET params).
  req.checkBody('postparam', 'Invalid postparam').notEmpty().isInt();
  req.checkParams('urlparam', 'Invalid urlparam').isAlpha();
  req.checkQuery('getparam', 'Invalid getparam').isInt();

  // OR assert can be used to check on all 3 types of params.
  // req.assert('postparam', 'Invalid postparam').notEmpty().isInt();
  // req.assert('urlparam', 'Invalid urlparam').isAlpha();
  // req.assert('getparam', 'Invalid getparam').isInt();

  // SANITIZATION
  // as with validation these will only validate the corresponding
  // request object
  req.sanitizeBody('postparam').toBoolean();
  req.sanitizeParams('urlparam').toBoolean();
  req.sanitizeQuery('getparam').toBoolean();

  // OR find the relevent param in all areas
  req.sanitize('postparam').toBoolean();

  var errors = req.validationErrors();
  if (errors) {
    res.send('There have been validation errors: ' + util.inspect(errors), 400);
    return;
  }
  res.json({
    urlparam: req.params.urlparam,
    getparam: req.params.getparam,
    postparam: req.params.postparam
  });
});

app.listen(8888);
```

Which will result in:

```
$ curl -d 'postparam=1' http://localhost:8888/test?getparam=1
{"urlparam":"test","getparam":"1","postparam":true}

$ curl -d 'postparam=1' http://localhost:8888/t1est?getparam=1
There have been validation errors: [
  { param: 'urlparam', msg: 'Invalid urlparam', value: 't1est' } ]

$ curl -d 'postparam=1' http://localhost:8888/t1est?getparam=1ab
There have been validation errors: [
  { param: 'getparam', msg: 'Invalid getparam', value: '1ab' },
  { param: 'urlparam', msg: 'Invalid urlparam', value: 't1est' } ]

$ curl http://localhost:8888/test?getparam=1&postparam=1
There have been validation errors: [
  { param: 'postparam', msg: 'Invalid postparam', value: undefined} ]
```

### Middleware Options
####`errorFormatter`
_function(param,msg,value)_

The `errorFormatter` option can be used to specify a function that can be used to format the objects that populate the error array that is returned in `req.validationErrors()`. It should return an `Object` that has `param`, `msg`, and `value` keys defined.

```javascript
// In this example, the formParam value is going to get morphed into form body format useful for printing.
app.use(expressValidator({
  errorFormatter: function(param, msg, value) {
      var namespace = param.split('.')
      , root    = namespace.shift()
      , formParam = root;

    while(namespace.length) {
      formParam += '[' + namespace.shift() + ']';
    }
    return {
      param : formParam,
      msg   : msg,
      value : value
    };
  }
}));
```

####`customValidators`
_{ "validatorName": function(value, [additional arguments]), ... }_


The `customValidators` option can be used to add additional validation methods as needed. This option should be an `Object` defining the validator names and associated validation functions.

Define your custom validators:

```javascript
app.use(expressValidator({
 customValidators: {
    isArray: function(value) {
        return Array.isArray(value);
    },
    gte: function(param, num) {
        return param >= num;
    }
 }
}));
```
Use them with their validator name:
```javascript
req.checkBody('users', 'Users must be an array').isArray();
req.checkQuery('time', 'Time must be an integer great than or equal to 5').isInt().gte(5)
```
####`customSanitizers`
_{ "sanitizerName": function(value, [additional arguments]), ... }_

The `customSanitizers` option can be used to add additional sanitizers methods as needed. This option should be an `Object` defining the sanitizer names and associated functions.

Define your custom sanitizers:

```javascript
app.use(expressValidator({
 customSanitizers: {
    toSanitizeSomehow: function(value) {
        var newValue = value;//some operations
        return newValue;
    },
 }
}));
```
Use them with their sanitizer name:
```javascript
req.sanitize('address').toSanitizeSomehow();
```

## Validation

#### req.check();
```javascript
   req.check('testparam', 'Error Message').notEmpty().isInt();
   req.check('testparam.child', 'Error Message').isInt(); // find nested params
   req.check(['testparam', 'child'], 'Error Message').isInt(); // find nested params
```

Starts the validation of the specifed parameter, will look for the parameter in `req` in the order `params`, `query`, `body`, then validate, you can use 'dot-notation' or an array to access nested values.

Validators are appended and can be chained. See [chriso/validator.js](https://github.com/chriso/validator.js) for available validators, or [add your own](#customvalidators).

#### req.assert();
Alias for [req.check()](#reqcheck).

#### req.validate();
Alias for [req.check()](#reqcheck).

#### req.checkBody();
Same as [req.check()](#reqcheck), but only looks in `req.body`.

#### req.checkQuery();
Same as [req.check()](#reqcheck), but only looks in `req.query`.

#### req.checkParams();
Same as [req.check()](#reqcheck), but only looks in `req.params`.

## Validation errors

You have two choices on how to get the validation errors:

```javascript
req.assert('email', 'required').notEmpty();
req.assert('email', 'valid email required').isEmail();
req.assert('password', '6 to 20 characters required').len(6, 20);

var errors = req.validationErrors();
var mappedErrors = req.validationErrors(true);
```

errors:

```javascript
[
  {param: "email", msg: "required", value: "<received input>"},
  {param: "email", msg: "valid email required", value: "<received input>"},
  {param: "password", msg: "6 to 20 characters required", value: "<received input>"}
]
```

mappedErrors:

```javascript
{
  email: {
    param: "email",
    msg: "valid email required",
    value: "<received input>"
  },
  password: {
    param: "password",
    msg: "6 to 20 characters required",
    value: "<received input>"
  }
}
```

### Optional input

You can use the `optional()` method to check an input only when the input exists.

```javascript
req.checkBody('email').optional().isEmail();
//if there is no error, req.body.email is either undefined or a valid mail.
```

## Sanitizer

#### req.sanitize();
```javascript

req.body.comment = 'a <span>comment</span>';
req.body.comment.username = '    user    ';

req.sanitize('comment').escape(); // returns 'a &lt;span&gt;comment&lt;/span&gt;'
req.sanitize('comment.user').trim(); // returns 'a user'

console.log(req.body.comment); // 'a &lt;span&gt;comment&lt;/span&gt;'
console.log(req.body.comment.user); // 'a user'

```

Sanitizes the specified parameter (using 'dot-notation' or array), the parameter will be updated to the sanitized result. Cannot be chained, and will return the result. See [chriso/validator.js](https://github.com/chriso/validator.js) for available sanitizers, or [add your own](#customsanitizers).

If the parameter is present in multiple places with the same name e.g. `req.params.comment` & `req.query.comment`, they will all be sanitized.

#### req.filter();
Alias for [req.sanitize()](#reqsanitize).

#### req.sanitizeBody();
Same as [req.sanitize()](#reqsanitize), but only looks in `req.body`.

#### req.sanitizeQuery();
Same as [req.sanitize()](#reqsanitize), but only looks in `req.query`.

#### req.sanitizeParams();
Same as [req.sanitize()](#reqsanitize), but only looks in `req.params`.

### Regex routes

Express allows you to define regex routes like:

```javascript
app.get(/\/test(\d+)/, function() {});
```

You can validate the extracted matches like this:

```javascript
req.assert(0, 'Not a three-digit integer.').len(3, 3).isInt();
```

## Changelog

See [CHANGELOG.md](CHANGELOG.md)

## Contributors

- Christoph Tavan <dev@tavan.de> - Wrap the gist in an npm package
- @orfaust - Add `validationErrors()` and nested field support
- @zero21xxx - Added `checkBody` function

## License

Copyright (c) 2010 Chris O'Hara <cohara87@gmail.com>, MIT License
