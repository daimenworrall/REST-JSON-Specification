# REST+JSON Specification
#### Rev 0.1
> By Daimen Worrall

## Introduction

The goal with REST+JSON is to create a standard for REST APIs that respond with JSON documents.

In this specification, we will describe various parts of a RESTful API and give guidelines on how they should operate.

All feedback is welcome, create an issue if you have any suggestions or amendments and they will all be considered. 

## Content Negotiation

### Client Responsibilities
Clients __MUST__ send all data in request documents with the header `accept: application/rest+json` without any media type parameters.

### Server Responsibilities
Servers __MUST__ send all data in response documents with the header `content-type: application/rest+json` without any media type parameters.

Servers __MUST__ respond with a 415 Unsupported Media Type status code if a request does not specify the header `accept: application/rest+json` __or__ `content-type: application/rest+json`.

Servers __MUST__ respond with an `error` if the response code is `4XX`

## Document Structure

### Top Level

A JSON object __MUST__ be at the root of every JSON API request and response containing data. This object defines a document’s “top level”.

#### A document __MUST__ contain at least one of the following top-level members:

- `meta` - Metadata about the request / api
- `data` - The requests primary data
- `errors` - an array of errors

The members `data` and `errors` __MUST NOT__ coexist in the same document.

The `data` object __MUST__ be an array. It should never be a singular object.

Main `data` objects __MUST__ contain an `_id`.

The `data` object __MAY__ contain child objects.

Child objects __MUST__ contain an `_id`. Child object without an `_id` __MUST__ be part of the main object.

#### A document __MAY__ contain any of these top-level members:

- `links` - links related to child objects contained inside of the main data object
- `related` - links to other endpoints related to the main data object, for example: next/previous pages for pagination

#### Here is an example of a basic success response

```javascript
{
  meta: {
    restJson: "0.1"
  },
  data: [
    {
      _id: "abcd1234",
      title: "Example Post",
      content: "This is an example post.",
      tags: ["test", "post"]
    }
  ]
}
```
Notice the child `tags` doesn't have an `_id`. This means that it is contained within the main object.

#### Here is an example of a basic success response with child objects

```javascript
{
  meta: {
    restJson: "0.1"
  },
  data: [
    {
      _id: "abcd1234",
      title: "Example Post",
      content: "This is an example post.",
      tags: ["test", "post"],
      comments: [
          {_id: "qqqq1111", content: "This post is awesome!", post: "abcd1234"},
          {_id: "aaaa5555", content: "Nice post.", post: "abcd1234"}
        ]
      }
    }
  ],
  links: {
    comments: "/api/comments"
  }
}
```
Notice the child `comments` have `_id`. This means they aren't part of the main object and are stored seprately in the database.

We also see a good example how the links object should be used.

## Fetching Data

Data can be fetched by sending a `GET` request to the server. Querying the name of resource will fetch all of that resource type, while adding an `_id` to the query will fetch a single resource.

Requests that use `GET` __MUST__ send the header `accept: application/rest+json` to tell the server that the client accepts JSON documents.

A server __MUST__ respond with one of the following:

- `200` - A successful request
- `400` - Invalid request (ie: the query was invalid)
- `401` - Unauthorized
- `404` - Not Found (If the resource exists but there are 0, the server should return a 200)

### Examples

To get all of the `posts` from an API, the following query would be used:

```javascript
accept: application/rest+json
GET /api/posts
```

To get an individual `post`, the following query would be used: (where `abcd1234` is the `_id` of the `post`)

```javascript
accept: application/rest+json
GET /api/posts/abcd1234
```
### Inclusion of related resources

To get related resources to the query, an `include` flag can be specified.

For example, if we want to return all `comments` related to our `post`, the following query would be used:

```javascript
accept: application/rest+json
GET /api/posts/abcd1234?include=comments
```

Multiple included resources would be separated by commas. For example, if we wanted to return all `likes` too, the following query would be used:

```javascript
accept: application/rest+json
GET /api/posts/abcd1234?include=comments,likes
```

### Sorting of resources

To sort by a specific field on the main resource, a `sort` flag can be specified.

For example, if we wanted to sort our `posts` by its `date`, the following query would be used:

```javascript
accept: application/rest+json
GET /api/posts?sort=date
```

To specify a direction for the sort (ie: ascending or descending) we would use a `direction` flag. For example:

```javascript
accept: application/rest+json
GET /api/posts?sort=date&direction=asc
```

The value of `direction` __MUST__ be either `asc` or `desc`.

### Filtering / Searching

Filtering or Searching is done via parameters. This means that a query parameter should match a main document member.

The filtering/searching __MAY__ be case insensitive and __MAY__ have to match exactly

For example, if we wanted to search the `title` member of our `posts`, the following query would be used:

```javascript
accept: application/rest+json
GET /api/posts?title=example
```

Case sensitivity and exact matching may be toggles with `sensitive` and `exact` queries respectively. To do this, the following query would be used:

```javascript
accept: application/rest+json
GET /api/posts?title=example&sensitive=false&exact=false
```

The default state for the `sensitive` and `exact` queries __MUST__ be `false`.

### Pagination

A server __MAY__ choose to limit the number of resources returned. In this case, we would use pagination.

If no pagination is specified but the server wants to limit the number of resources, the server __MAY__ return a default pagination value, starting from the first page.

If pagination is used, the following `links` __MUST__ be included in the response:

- `first` - the first possible page of the pagination
- `last` - the last possible page of the pagination
- `previous` - the previous page (__MUST__ be `null` if on the first page)
- `next` - the next page (__MUST__ be `null` if on the last page)

If pagination is used, the following `meta` members __MUST__ be included in the response:

- `currentPage` - the current page
- `totalPages` - the total number of pages

The client __MAY__ specify a `page` and an `amount` in their request. To do this, the following query would be used:

```javascript
accept: application/rest+json
GET /api/posts?page=0&amount=10
```

If `amount` is not specified, the server __MAY__ use a default amount (in most cases, it is recommended to pick `10`).

The `page` __MUST__ start at 0.

## Creating Resources

Resources can be created on the server using a `POST` request with the data for the resource in the request body.

Requests that use `POST` __MUST__ send the header `content-type: application/rest+json` to tell the server that the data is formatted as JSON.

A server __MUST__ respond with one of the following:

- `201` - The resource was created successfully
- `400` - Invalid request (ie: the data was invalid)
- `401` - Unauthorized

A query __MUST__ not contain an `_id` __but__ the server __MAY__ still create this resource, ignoring the `_id` property and generating a new one.

The server __MUST__ respond with the created documents `_id`.

### Linked resources

If the resource being created is linked as a child to another resource. It __MUST__ contain a descriptive member with the parents `_id` as its value.

For example, if we want to add a comment to the post with `_id: abcd1234`, the request would look like:

```javascript
content-type: application/rest+json
POST /api/posts
{
  content: "This post is awesome!",
  post: "abcd1234"
}
```

## Updating Resources

Resources can be updated on the server using a `PUT` request with the data for the resource in the request body.

Requests __MUST__ contain an `_id` and this __MUST__ be in the URL (similar to `GET` of a single resource). If this `_id` is missing, the server __MUST__ respond with a `400` error.

Requests __MUST__ only contain data to be updated and this data __MUST__ be in the request body.

Requests that use `PUT` __MUST__ send the header `content-type: application/rest+json` to tell the server that the data is formatted as JSON.

A server __MUST__ respond with one of the following:

- `202` - The resource was created successfully
- `204` - The data was valid but the server didn't update the resource
- `400` - Invalid request (ie: the data was invalid)
- `401` - Unauthorized

An example update request would look like:

```javascript
content-type: application/rest+json
PUT /api/posts/qwer1234
{
  content: "This is updated!"
}
```

## Deleting Resources

Resources can be deleted on the server using a `DELETE` request.

Requests __MUST__ contain an `_id` and this __MUST__ be in the URL (similar to `GET` of a single resource). If this `_id` is missing, the server __MUST__ respond with a `400` error.

A server __MUST__ respond with one of the following:

- `200` - The resource was deleted successfully
- `400` - Invalid request (ie: the `_id` was invalid)
- `401` - Unauthorized

An example `DELETE` request would look like:

```javascript
content-type: application/rest+json
DELETE /api/posts/qwer1234
```

## Errors

A server __MAY__ respond with an error. Errors __MUST__ have a code of `4XX`.

An error response __MUST__ contain an `errors` attribute. This `errors` attribute __MUST__ be an array.

Each error __MUST__ contain a `description` member and __MAY__ contain `title` and `link` members. An example `link` can be to a useful support article.

An example of an error response would look like:

```javascript
content-type: application/rest+json
{
  errors: [
    {name: "Error!", description: "Something went wrong.", link: "http://google.com"}
  ]
}
```

Generally there would only be a single error, but allowing for multiple leaves scope for errors such as a user registration form. ie: `invalid username` and `invalid password` errors in the same response.

## Meta

The server __MUST__ include a `meta` attribute with every response.

The `meta` attribute in a request __MUST__ contain an `restJson` member containing the version of this specification used.

The server __MAY__ send back other non-standard information in the `meta` attribute. Examples of which include: `copyright` and `authors` members.
