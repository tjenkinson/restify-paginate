# restify-paginate
--------------------------

resitfy-paginate is a middleware that helps the navigation between pages.

When you have a very large set of results to display, you might want to divide it into pages. So when you ask for your resource, you get a portion of the result set aka a `page` and the possibility to get the other pages.

Now the question is: "I have my first page, how can I get to ther other ones ?".
The idea is to not send only the page itself but also the links to the others.

Let's say you get your resource from this URL: `http://api.mysite.com/api/myresource`, you will be able to get the different pages adding the `page` and `per_page` param e.g. `http://api.mysite.com/api/myresource?page=3&per_page=20`

This module processes the request params and generates the links to the first, previous, next and last pages of the result set.

By default this module will generate links like the github API c.f.  [Github Documentation](https://developer.github.com/guides/traversing-with-pagination/), the page count will start at 1 and the page size will be 50.

## Get the module

This module is registered on npm so you just need to get it doing:

```shell
npm install restify-paginate
```

## Using the module

First you require the module and then add it as one of your server middlewares.

*You have to give the restify server object to the paginate module*

```js
var restify = require('restify'),
    paginate = require('restify-paginate');

var server = restify.createServer({
        name: 'My API'
    });

server.use(restify.queryParser());
server.use(paginate(server));

```

This will process the `page` and `per_page` request params and add them in the request object under the `paginate` key.

Your request object will have something like this:

```json
    {
        params: {...},
        paginate: {
            page: 2,
            per_page: 30
        },
        ...
    }
```

It will also add a `paginate` object to the response object, which contains methods for creating paginated responses.

###Paginated responses

with this information, restify-paginate can generate paginated reponses for you. There basically are just two methods you need, depending on what you're working with.

####Generating a response from a full dataset

If you working with a full dataset you want to paginate, you can use the `getPaginatedResponse(data)` method. This will take a whole dataset and create a response object which contains `data` and `pages` linking to the other pages. E.g.:

```js
// example data
var data = {
    first: 1,
    second: 2,
    third: 3,

    // and so on ...

    fortytwothousand: 42000
}

// Let's say page is set to 2 and per_page to 4. Calling
var paginatedResponse = res.paginate.getPaginatedResponse(data);

console.log(paginatedResponse);
// will now print:
{
    data: {
        five: 5,
        six: 6,
        seven: 7,
        eight: 8
    },
    pages: {
        prev: 'http://api.myurl.com/myresource?page=1&per_page=4',
        next: 'http://api.myurl.com/myresource?page=3&per_page=4',
        first: 'http://api.myurl.com/myresource?page=1&per_page=4',
        last: 'http://api.myurl.com/myresource?page=8399&per_page=4',
    }
}

// now send the paginated response
res.send(paginatedResponse);

```

since you usually will send() a paginated response after generating it, there is a shortcut method for that:

```js
// generates a paginated response and calls send(response)
res.paginate.sendPaginatedResponse(data);
```

####Generating a response with a already paginated dataset

If you want to paginate data from a database, you probably want to paginate your data by using `offset` and `limit`, instead of retrieving the whole dataset and using `sendPaginatedResponse()`. This is where the `req.paginate.page` and `req.paginate.per_page` come in handy.

```js
// SQL like query
var query = 'SELECT * FROM my_table'
            + 'OFFSET ' + (req.paginate.page * req.paginate.per_page)
            + ' LIMIT ' + req.params.per_page;
// Mongoose like query
Potatoes.find()
        .offset(req.paginate.page * req.paginate.per_page)
        .limit(req.params.per_page);
```

let's you return a paginated dataset from your database. Now you can generate a response containing this paginated data and links to the other pages:

```js
// generate a response using the data retrieved from the database
var response = res.paginate.getResponse(data);

// send the response
res.paginate.send(paginate);

```

There is a shortcut for this as well

```js
// generate a response and call send(response)
res.paginate.send(data);
```

The response generated by this, will obviously not contain a link to the last page, since there is no information on how big the whole dataset is. If you need a link to the last page, you can also generate a response providing the size of the whole dataset.

```js
// lets say there are 1000 items in your databse...
res.paginate.send(data, 1000);
```

###Creating custom paginated responses

If this doesn't quite fit your needs, you can create your own responses, [using paginate-restifys methods](#functions-reference)

e.g. using getLinks()

```js
// SQL like query
var query = 'SELECT * FROM my_table'
            + 'OFFSET ' + (req.paginate.page * req.paginate.per_page)
            + ' LIMIT ' + req.params.per_page;
// Mongoose like query
Potatoes.find()
        .offset(req.paginate.page * req.paginate.per_page)
        .limit(req.params.per_page);
```

You can now get your links based on the total count of your result set

```js
// Let's say your query would have returned 54356 results without the offset and limit clauses
var links = res.paginate.getLinks(54356);
```

This will give you:

```js
{
    next: 'http://api.myurl.com/myresource?page=2',
    last: 'http://api.myurl.com/myresource?page=1088',
}
```

Here the next page is the second one and the last is the 1088th one because you need to get to the 1088th page to get the last results.

The `first` and `prev` keys are not added since the page asked is the first one.

Depending on how you are retrieving your data, it may be the case, that you'll need to do another query to get total count of items. For huge data sets, this could have an significant performance impact. If that is the case, and you don't necessarily need the `last` page, you can call the `getLinks()` method without a param. This will only generate the first, next and previous links.

## Options

You can change this module behavior through some options by giving them to the paginate module.

```js
    var options = {...};
    server.use(paginate(server, options));
```

By default these options are:

```js
{
    paramsNames: {
        page: 'page',           // Page number param name
        per_page: 'per_page'    // Page size param name
    },
    defaults: {                 // Default values
        page: 1,
        per_page: 50
    },
    numbersOnly: false,         // Generates the full links or not
    hostname: true              // Adds the hostname in the links or not
}
```

### paramsName

You can choose the names of the request params for the page number and the page size.

So you can have URLs like this `http://api.mysite.com/resource?pageCount=2&pageSize=35` by setting the defaults options to:

```js
{
    page: 'pageCount',
    per_page: 'pageSize'
}
```

#### defaults

You can change the dafult values for the page number and the page size. If these informations are not provided in the request, these values will be used.

### numbersOnly

Indicates if the urls should be generated or if you only need the pages numbers. If set to true, the links will look like this:

```json
{
    first: 1,
    prev: 2,
    next: 4,
    last: 57
}
```

### hostname

indicates if the hostname should be in the generated URLs. If set to false, the URLs will look like this `/myresource?page=2`

## Functions reference

### paginate(server, options)

Initialize the paginate module.

**params**:
- server: The restify server object
- options: Your custom options


### res.paginate.getLinks([count])

**params**:
- count (optional): The total number of results. If count is not provided, the link to the last page won't be generated

Returns an object containing the pages.

### res.paginate.addLinks([count])

**params**:
- count: The total number of results

Add a `Link` header to the response that contains the pages.

### res.paginate.getResponse(data, [count])

generates a reponse, containing the given data and a link to the other pages.

**params**:
- data: An Array of Objects
- count: The total count of Items. If count is not provided, the link to the last page won't be generated

Returns an object containing the data and pages

### res.paginate.sendPaginated(data)

sends the data generated by `getResponse`

**params**:
- data: An Array of Objects
- count: The total count of Items. If count is not provided, the link to the last page won't be generated

### res.paginate.getPaginatedResponse(data)

generates a paginated response

**params**:
- data: An Array of Objects, which you want to paginate

Returns an object containing the data and pages

### res.paginate.sendPaginated(data)

sends the data generated by `getPaginatedResponse

**params**:
- data: An Array of Objects, which you want to paginate
