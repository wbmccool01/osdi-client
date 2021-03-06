# OSDI Client

Library for interacting with any OSDI compliant server (http://opensupporter.github.io/osdi-docs/).

Will parse the app entry point and returned metadata on subsequent api calls
and create convenience methods for you.

Note: I haven't finished populating the Resource classes with the correct property
fields.

## Usage

Works in the browser and in node. Browersify / webpack friendly.

```
npm install --save osdi-client
```

```
const osdi = require('osdi-client')
osdiClient.client('https://sample-system.com/api/aep')
  .then(client => {
    // ready to use client
  })
  .catch(err => {
  })
```

Relies on `superagent` (https://github.com/visionmedia/superagent) for most of
the heavy lifting, so get familiar with their API first.

### osdi.client

The initialization method returns a promise, which is resolved with an `client`
object or reject with whatever error.

`osdi.client` accepts an the API's app entry point...
```javascript
osdi.client('https://my-osdi-endpoint.com')
.then(client => {
  // do stuff with the client here
})
.catch(err => {
  // either the aep was formatted invalidly or there was a network error probably
})
```

or you can give it the JSON data that lives at the app entry point directly, and
save a network request.

```javascript
osdi.client({
  "motd": "Welcome to the OSDI API Entry Point!",
  "vendor_name" : "Foobar Incorporated",
  "product_name" : "Curly Braces OSDI Server",
  "osdi_version" : "1.0",
  "max_pagesize": 25,
  "namespace": "osdi_sample_system",
  // and on and on...
})
.then(/* yada */)
```

### client.get<Resource>

The `client` object generated by `osdi.client` will have all of the exposed
resources created as get methods.

So, for example, if the AEP (app entry point) has `osdi:people` and `osdi:events`,
`client` will have `client.getPeople()` and `client.getEvents()`.

These methods will return a `superagent` request, which can be called with `.end`
as in
```javascript
client.getPeople()
.end((err, res) => {
  if (err) {
    // handle error
  }

  // do something with res
})
```

or with `.then` and `.catch` as in

```javascript
client.getPeople()
.then(res => {
  // do something with res
})
.catch(err => {
  // handle error
})
```

And since `superagent` supporters promises, you can do
```
async myFunc () {
  const people = await client.getPeople()
  // do something with people
}
```

Since the `get<Resource>` methods return a `superagent` request, you can manually
add headers with `.set`, modify query parameters, etc.

```javascript
client.getPeople()
.set('OSDI-API-TOKEN', secretToken)
.end((err, res) => {
  // yadada
})
```

Helper methods for queries and authentication are planned but not implemented yet.

### Resource Objects

For whatever resources the app entry point exposes, a resource class will be
available from `client.resources`. These classes have `.save`, `.delete`, and
getter / setters for any attribute in the OSDI docs. `.save` and `.delete` also
expose `superagent` requests.

*** Note *** --- I have chosen to make all properties camel case, so `family_name`
becomes `familyName`. This is all done behind the scenes. I did it because most
Javascript uses camelCase, but if you think it's a bad decision (leads to unnecessary confusion)
let me know.

So for example, to create a new event...

```javascript
const party = new client.resources.Event({
  name: 'Pizza Party',
  title: 'Happy Birthday',
  description: 'Celebrating Pizza\'s Birthday'
})

console.log(party.name()) // 'Pizza Party'
party.name('Pizza\'s Birthday Party')
console.log(party.name()) // 'Pizza's Birthday Party'

party.save()
.set('OSDI-API-TOKEN', secretToken)
.end((err, res) => {
  // now the server knows about the party
})
```

### Parsing Responses with `client.parse`

Calling `client.parse` on the response will serialize its metadata into function
calls and wrap its embedded response data inside Resource classes. For example:

```javascript
client.getPeople()
.then(raw => {
  const res = client.parse(raw)
  // Now res has the following methods, if appropriate
  // each returns a superagent request
  res.nextPage().then(/* stuff */)
  res.prevPage().then(/* stuff */)

  // Whatever resource was requested is not available as a property
  res.people.forEach(person => {
    // and each of these are Person object's, with `.save`, `.delete`, and getter / setters
    person.givenName('Greg')
    person.save()
    .end((err, result) => {
      // the guy's named Greg now
    })
  })
})
```

To make everyone in the database named Greg, simply

```javascript
function nameGreg (response) {
  Promise.all(
    response.people.map(person => new Promise((resolve, reject) => {
      person.givenName('Greg')
      return person.save() // save returns a superagent object which is promise compatible
    }))
  ).then(gregs => {
    if (response.nextPage) { // nextPage will be a function if another page exists, undefined otherwise
      response.nextPage()
      .then(nameGreg)
    }
  })
}
```

### Manually Editing a Resource

All of a resources data live at `person.data` or `event.data` or whatever. However,
if you manually modify `.data` and then call `.save`, the Resource will not know
the given field has been modified, and so it will not include it in the body of
the request generated by `resource.save()`. To make sure it gets included, use
`resource.markModified('field')`.

```javascript
// BAD!!!!
person.data.givenName = 'Thomas'
person.save().then(/* stuff */)

// OK!!!
person.data.givenName = 'Thomas'
person.markModified('givenName')
person.save().then(/* stuff */)

// PREFERRED!!!
person.givenName('Thomas')
person.save().then(/* stuff */)
```

## Tests

Written with Mocha, Chai, and node-nock. `npm test` does the trick.
