# Jamit Labs Simple API standard (JLS:API)

This API specification intends to standardize requests and responses for JSON based APIs with the goal to keep things simple and yet flexible and automatable. It is inspired by [JSON:API](http://jsonapi.org) but the structure is much simpler and it has more features when it comes to sorting & filtering.

## Global Stuff

A client must always send the locale it's running in with the HTTP Header field `Accept-Language`. For example:

`Accept-Language: en-US` or `Accept-Language: en`

A client must send requests with the value `application/vnd.simple-api+json` in the `Accept` header.
If there is a body the client must set the value `application/vnd.simple-api+json` also to the `Content-Type` header.

A server must respond with the `application/vnd.simple-api+json` in the `Content-Type` header.

## Basic Response Structure

There are two different cases for response objects:

1. Success: HTTP Codes 200-299
2. Bad Request: HTTP Codes 400-499

For Success the one-object structure looks like the following:

``` json
{
    "typeName": "User",
    "identifier": "783",
    "firstName": "John",
    "lastName": "Appleseed",
    "createdChallengesCount": 4,
    "department": {
        "typeName": "CompanyDepartment",
        "identifier": "3"
    }
}
```

The multiple-object structure looks the same, except that the top level object is an array including multiple hashes of the above structure:

``` json
[
    {
        "typeName": "User",
        "identifier": "783",
        "firstName": "John",
        "lastName": "Appleseed",
        "createdChallengesCount": 4,
        "department": {
            "typeName": "CompanyDepartment",
            "identifier": "3"
        }
    },
    { "typeName": "User", "identifier": "784", "firstName": "..." },
    { "typeName": "User", "identifier": "785", "firstName": "..." },
]
```

Specifically note the following:

- The `typeName` and `identifier` are required fields amongst all responses.
- The `typeName` must be titlecased like class names usually are.
- Attributes and relationships are presented at the same level.

For Bad Request it has always this structure:

``` json
[
    {
        "message": "can't be empty",
        "pointer": "firstName"
    }
]
```

Specifically note the following:

- The response is always an array, even if there's only one error.
- The `message` is localized and human readable.
- The `pointer` is not localized and refers to the problematic field in the request.

## Request Options

### Include Relationships

If you want to include detailed fields of relationships, then simply add an `include` to your request and list all relationship names to include separated by a comma. For example:

Request: `/user/783?include=createdChallenges,department`

Response:

``` json
{
    "typeName": "User",
    "identifier": "783",
    "firstName": "John",
    "lastName": "Appleseed",
    "createdChallenges": [
        { "typeName": "Challenge", "identifier": "72" },
        { "typeName": "Challenge", "identifier": "73" },
        { "typeName": "Challenge", "identifier": "74" },            
    ],
    "department": {
        "typeName": "CompanyDepartment",
        "identifier": "3",
        "name": "iOS Development",
        "company": { "typeName": "Company", "identifier": "75" },
        "membersCount": 3
    }
}
```

Note how the `department` now has its detailed properties included as well and the `createdChallenges` (because it's an array) includes the `identifier`s. You can see though that in the `department` there's also a `company` and some `members` defined without their detailed fields. How do you get them on those?

Just specify them separated by a dot like so in your request:

Request: `/user/783?include=department.company,department.members`

Note that you can not chain them once more, so the maximum level of depth is 3 including the top level object. Also beware that there's no pagination available for included relationships.

### Pagination

All top level arrays of objects will be paginated by default. To turn pagination off simply add `pagination=false` to your request. The API documentation can also specify that a specific endpoint has pagination always turned off.

By default servers are required to paginate with a default page size. This number must be documented globally in the API docs, if this is not the case, a page size of 25 is to be used.

A client can specify the requested page by two ways:

1. Provide a `page[num]` and optionally a `page[size]`
2. Provide a `page[offset]` and optionally a `page[limit]`

A server can simply convert one way to the other to fulfil both requirements. Alternatively it can choose one strategy explicitly.

Note that pagination doesn't work on related objects with the `include` instruction.

### Sorting

The API documentation can declare specific attributes of a model as `sortable`. If this is the case, then calls can be made as follows:

Request: `/users?sort=firstName`

It is also possible to specify multiple attributes with decreasing priority by separating them by a comma like so:

Request: `/users?sort=firstName,lastName`

You can even sort by attributes of relationships with the maximum level of depth being 3 like so:

Request: `/users?sort=department.company.name,lastName`

### Filtering

The API documentation can declare specific attributes of a model as `filterable`. If this is the case, then depending on the type different filter options are available.

Note that you can combine any different filters with each other in a single request, the filters will work as a conjunction (logical AND). By default, for all attribute types the `equal` filter option will be used if you don't specify one. For example:

Request: `/users?filter[firstName]=John&filter[lastName]=Appleseed`

This will filter all users with the exact name "John Appleseed". You can also filter by attributes of relationships with a maximum depth level of 3. For example:

Request: `/users?filter[department.company.name]=JamitLabs`

This will list all users working in a company with the exact name match "JamitLabs".

#### Filtering by Strings

For filterable attributes of type String you have the following options (replace `attribute` by your String attribute name):

- `filter[attribute,equal]`: Filters with exact, case-sensitive matches of the full string.
- `filter[attribute,pattern]`: Filters with a pattern, an [SQL pattern](https://www.w3schools.com/sql/sql_like.asp) if not documented otherwise.

*SQL Tip:* Using the `pattern` option you can filter by Strings starting with a substring using `%substring`, ending with a substring using `substring%` or containing a substring anywhere using `%substring%`.

#### Filtering by Numerals & Dates

For filterable attributes of numerical type (Integer, Float, Double) or of a date type (Date, DateTime) you have the following options (replace `attribute` by your attribute name):

- `filter[attribute,equal]`: Filters with an exact match.
- `filter[attribute,gt]`: Filters all that are greater than the specified.
- `filter[attribute,gte]`: Filters all that are greater than or equals the specified.
- `filter[attribute,lt]`: Filters all that are lower than the specified.
- `filter[attribute,lte]`: Filters all that are lower than or equals the specified.

## Request Body

When sending POST, PUT or PATCH requests to create or update data on the server, clients must use the following structure:

``` json
{
    "firstName": "John",
    "lastName": "Appleseed",
    "department": { "typeName": "Department", "identifier": "72" }
}
```

Specifically note the following:

- The response is always a hash and directly contains the attributes.
- Editable to-one relations are referenced by their minimum properties object type (other properties will be ignored).
- Editable to-many relations are not listed at all. They must be placed under their own endpoint.

The response of the server for POST, PUT or PATCH requests must include the entire object as if a GET request on the specific object was made.
