# Atlas Workshop
[Session slides -> WIP](SearchWorkshopSlides.pdf)

## Prerequisites
Workshop attendees MUST bring their own personal laptop in order to actively participate in the workshop.

Please create a user account on MongoDB Atlas. 

Accounts MUST be created using your personal email address, NOT your J.P.Morgan email.

Download and Install MongoDB Compass, this is verified MongoDB software. Please only use this official link.﻿

Attendees must connect to the JPMC-GUEST-NETWORK on their personal laptop for the workshop.

Ability to install binaries on your personal laptop - NodeJS, Python pip3 etc.

## Steps
### 1. Add sample data to your MongoDB Atlas cluster

If you haven't already sone so, [create an Atlas cluster](https://cloud.mongodb.com/).

Make sure at least 1 user with read-write access has been added.

Make sure that IP Access List allows access (add `Allow ACCESS FROM ANYWHERE`(`0.0.0.0\0`) to be sure).

Add sample data:
- Click the `...` next to your cluster in the Atlas UI
- Select `Load Sample Dataset`

It will take a few minutes for the sample data to be added.

## MongoDB Queries
### 1. Compass

Navigate to the Search bar in your collection, we will be using the ```sample_airbnb``` collection. If you do not have Compass, we can use the Data Explorer UI for the remainder of this session. Alternatively, we can also use the command line if you have ```mongosh``` installed. 

#### a. Query 1

Find all properties in the database with text filter:
Market in Hong Kong: {"address.market":"Hong Kong" } and is an Apartment : {"property_type" :"Apartment"}
To specify equality conditions, use : expressions in the query filter document.
```json
{"address.market":"Hong Kong", "property_type" : "Apartment"}
```

#### b. Query 2

Find all properties in the database with a range based filter:
have at least three bedrooms: {"bedrooms": {"gte" : 3}}  and price range from 1,300 to 1,500: "price": {"gte" : 3}}  and price range from 1,300 to 1,500: "price": {"gte": 1300, "$lte": 1500}
```json
{ "bedrooms": {"$gte": 3}, "price": {"$gte": 1400, "$lte": 1500}}
```

#### c. Query 3

Find all properties in the database with a field that is inside an array:
Has both Wifi and Kitchen: { "amenities": { "$all": ["Wifi", "Kitchen"] }}
When specifying compound conditions on array elements, you can specify the query such that either a single array element meets these condition or any combination of array elements meets the conditions.
```json
{ "amenities": { "$all": ["Wifi", "Kitchen"]}}
```


#### d. Query 4

Find all properties in the database with combination of elements:
Market in Hong Kong: {"address.market":"Hong Kong" } and is an Apartment : {"property_type" :"'Apartment"}
Has at least three bedrooms: {"bedrooms": {"gte" : 3}} and price range from 1,300 to 1,500: "price": {"gte" : 3}} and price range from 1,300 to 1,500: "price": {"gte": 1300, "$lte": 1500}
Has both Wifi and Kitchen: { "amenities": { "$all": ["Wifi", "Kitchen"] }}
```json
{"address.market":"Hong Kong", "property_type" : "Apartment", "bedrooms": {"$gte": 3}, "price": {"$gte": 1400, "$lte": 1500}, "amenities": { "$all": ["Wifi", "Kitchen"]}}
```

#### e. Query 5 (Applicable for Compass or CMD Line)

In this example, run .explain() to returns the queryPlanner information for the evaluated method. Navigate to the Explain tab in compass to see query execution. Alternatively, use the command line. 
```js
db.listingsAndReviews.find({"address.market":"Hong Kong", "property_type" : "Apartment", "bedrooms": {"$gte": 3}, "price": {"$gte": 1400, "$lte": 1500}, "amenities": { "$all": ["Wifi", "Kitchen"]}}).explain()
```


## Aggregation Framework
### 1. Compass

Navigate to the Aggregation tab. in your collection, we will be using the ```sample_airbnb``` collection. 

#### a. Query 1

Group the listing views and group by real estate type using the $group command. Then sort by most popular. Sorting is done via a "1" or "-1" for denoting order of ascending/descending, respectively.

`$group`
```
{
    '_id': '$_id.property_type',
    'avgViews': {
        '$avg': '$viewsCount'
     }
 }
    
```
`$sort`
```
{
    'totalViews': -1
}
```

#### b. Query 2

Let's calculate now popularity by property type, but restricting it to an area of 100 Km around New York City. MongoDB Aggregation Framework has a pipeline philosophy, which may remind some people of the way Unix Pipelines Work: there is a list of ordered stages and the input of every stage is the input of the next one. This way processing is modular and easier to reason about, particularly with the aid of Compass Aggregation Builder.

`$match` - STAGE 1: FILTER TO PROPERTIES NEAR NEW YORK
```
{
    '_id.address.location':  { 
        '$geoWithin': { 
            '$centerSphere': [ [ -73.9667, 40.78], 100/6378.137 ]
         }
     }
}
```

`$group` - STAGE 2: GROUP BY PROPERTY TYPE AND AGGREGATE
```
{
    '_id': '$_id.property_type',
    'avgViews': {
        '$avg': '$viewsCount'
    }
}
```

`$sort` -  STAGE 3: SORT IN DESCENDING ORDER
```
{
    'totalViews': -1
}
```

HOMEWORK: We encourage you to checkout this [book](https://www.practical-mongodb-aggregations.com/) to learn more about aggregations. This is a free resource available to the general public. We would love to hear your feedback and use cases on how you use Aggregation Framework in the future. 


## Search
### 1. Navigate to the Search Index configuration page on the Atlas UI 

### 2. Add default Search index
- Keep the name as `default`
- Databse: `sample_mflix`
- Collection: `movies`
- Leave `Dynamic Mapping` turned on

### 3. Add autocomplete index
- Set the name to `autocomplete`
- Turn off `Dynamic Mapping`
- Click `Refine Your Index`
- Add a new field mapping for the `title` field with a `Data Type` of `Autocomplete`

### 4. (OPTIONAL) Connect MongoDB Compass
You can get the URL from the Atlas UI - click the `Connect` button next to your cluster.

Paste the URL into the Compass connection box, replacing the username and password.

### 5. Create aggregation pipeline to search for movies
From Compass (or the Atlas UI), navigate to the `Aggregation` pane for `sample_mflix.movies`.

Create these stages in sequence:

#### a. `$search`
```js
{
  index: 'default',
  text: {
    query: 'Indiana Ones',
    path: ['plot', 'fullplot', 'title'],
    fuzzy: {}
  },
  highlight: {path: 'fullplot'}
}
```

#### b. `$limit`
```
12
```

#### c. `$project`
```js
{
  title: 1,
  year: 1,
  poster: 1,
  plot: 1, 
  imdb: 1,
  score: {$meta: 'searchScore'},
  highlights: {
    '$meta': 'searchHighlights'
  }
}
```

- Click `EXPORT TO LANGUAGE`
- Choose `NODE` as the export language
- Copy the code and keep it safe
- Save the pipeline if using Compass

### 6. Create and test a HTTPS endpoint to search for movies
- Create a new Atlas App Service and name it `Movies`.
- Select `Functions` and then `Create New Function`.
- Set `Authentication` to `System`, and then `Save Draft`.
- Use this template for the function code:
```js
exports = async function({ query, headers, body}, response) {
    
    // GET A HANDLE TO THE MOVIES COLLECTION
    const moviesCollection = context.services.get("mongodb-atlas").db("sample_mflix").collection("movies");
    
    // GET SEARCHTERM FROM QUERY PARAMETER. IF NONE, RETURN EMPTY ARRAY
    let searchTerm = query.searchTerm;
    if (!query.searchTerm || searchTerm === ""){
         return [];
    }
  
    const searchAggregation = [];  
    return await moviesCollection.aggregate(searchAggregation).toArray();
};
```
- Replace the `[]` for `searchAggregation` with your aggregation pipeline
- Replace `Indiana Ones` with `searchTerm`
- `Save Draft`
- Select `HTTPS Endpoints`
- `Add An Endpoint`
- `Route`: `/movies'
- `HTTP Method`: `GET`
- Enable `Respond With Result`
- `Function`: the function you just created
- `Save Draft`
- `REVIEW DRAFT & DEPLOY`
- Copy the URL -> `<baseURL>`
- From a browser, paste the URL into the location bar (`<baseURL>?searchTerm=blues`). This will search for movies that mention `blues` - replace it with your own string (using `%20` for spaces - e.g., `blues%20brothers`)

### 7. Create aggregation pipeline to autocomplete movie titles
From Compass (or the Atlas UI), navigate to the `Aggregation` pane for `sample_mflix.movies`.

Create these stages in sequence:

#### a. `$search`
```js
{
  index: 'autocomplete',
  autocomplete: {
    query: 'Indiana J',
    path: 'title'
  }
}
```
#### b. `$limit`
```js
10
```
#### c. `$project`
```js
{
  _id: 0,
  title: 1
}
```

### 8. Create and test a HTTPS endpoint to autocomplete movie titles
- Open your `Movies` Atlas App Service.
- Select `Functions` and then `Create New Function`.
- Set `Authentication` to `System`, and then `Save Draft`.
- Use this template for the function code:
```js
exports = async function({ query, headers, body}, response) {
  
  // GET A HANDLE TO THE MOVIES COLLECTION
  const moviesCollection = context.services.get("mongodb-atlas").db("sample_mflix").collection("movies");
  
  // GET SEARCHTERM FROM QUERY PARAMETER. IF NONE, RETURN EMPTY ARRAY
  let searchTerm = query.searchTerm;

  if (!query.searchTerm || searchTerm ===""){
    return [];
  }
  
  const searchAggregation =[];

  return await moviesCollection.aggregate(searchAggregation).toArray();
};
```
- Replace the `[]` for `searchAggregation` with your aggregation pipeline
- Replace `Indiana Ones` with `searchTerm`
- `Save Draft`
- Select `HTTPS Endpoints`
- `Add An Endpoint`
- `Route`: `/titles'
- `HTTP Method`: `GET`
- Enable `Respond With Result`
- `Function`: the function you just created
- `Save Draft`
- `REVIEW DRAFT & DEPLOY`
- Copy the URL -> `<baseURL>`
- From a browser, paste the URL into the location bar (`<baseURL>?searchTerm=blues`). This will search for movies that mention `blues` - replace it with your own string (using `%20` for spaces - e.g., `blues%20brothers`)

## OPTIONAL: Use your new API from a REACT web app

This repo includes a REACT movie-search web app which is able to use the two HTTP endpoints you've just created.

To download and start the app:
```bash
git clone git@github.com:ClusterDB/AtlasSearchWorkshop.git
cd AtlasSearchWorkshop
npm install
npm start
```
The web app should open in a browser window. If you try searching for anything, you should get a message that you need to set up the HTTPS endpoints.

To make the app work, you need to add your endpoints to `src/components/Home.js` (line 22) and `src/components/SearchBar/SearchBar.js` (line 15).

When you save the files, the app should rebuild - you may need to refresh the browser window.

## Final demo

A quick demo of some of the features of Atlas Search (including the queries) using [Restaurant Finder](https://www.atlassearchrestaurants.com/). Code can be found [here](https://github.com/mongodb-developer/WhatsCooking).
