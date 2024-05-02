# Example Project

This guide explains the example application we use to introduce the different Neo4j drivers in detail.

## Example Project Description

To demonstrate connection to and usage of Neo4j in different programming languages we've created an example application.
It is a [simple, one-page webapp](https://my-neo4j-movies-app.herokuapp.com/), that uses Neo4j's movie demo database (movie, actor, director) as data set.
The same front-end web page in all applications consumes 3 REST endpoints provided by backend implemented in the different programming languages and drivers.

* movie search by title
* single movie listing
* graph visualization of the domain

![](https://dist.neo4j.com/wp-content/uploads/movie_application.png)

## GitHub

The source code for all the different language examples is available on [https://github.com/neo4j-examples?query#movies](Repositories) as individual repositories that can be cloned and directly used as starting points.

## Domain Model

We use a simple Movie database domain, which is also built into Neo4j Browser.

If you need the data separately, it is available in this https://github.com/neo4j-graph-examples/movies[GitHub repository].

```cypher
(:Person {name: string})-[:ACTED_IN {roles: [string]}]->(:Movie {title: string, released: number})
```


![](https://dist.neo4j.com/wp-content/uploads/graph-examples-movies-model.svg)


An example subgraph of the data is the neighborhood of "Tom Hanks".

![](https://dist.neo4j.com/wp-content/uploads/graph-examples-movies-example.png)

## Data Setup

### Neo4j Labs Demo Server

Most of the examples should already work with the following settings:

* Connection URI: `neo4j+s://demo.neo4jlabs.com`
* Database Name: `movies`
* Username: `movies`
* Password: `movies`

Other data setup options are explained <<other-setups,below>>.

## Webpage

The webpage is a single, simple Bootstrap page that uses jQuery AJAX calls to access the backend and adds the returned JSON result data to HTML DOM elements in the page.

For the search, that is the `/search?q#query` endpoint whose results are added as table rows in a listing.

If you click on any of the movie titles, the right-side view is populated with the movie details.

For the movie details it uses the `/movie/A%20Movie%20Title` endpoint and the results are rendered as panel title, image source, and unordered list for the crew.

## Graph Visualization

For the Graph Visualization we use [d3.js](http://d3js.org). Our `/graph` endpoint already returns the data in the format of "nodes" and "links"-list that d3 can use directly.
We then apply the force-layout algorithm to render nodes as circles and relationships as lines, and add some minimal styling to the visualization to provide the movie title/person name as title attribute to the node circles which is shown as a tooltip.

## Endpoints

### Search Movies

```
// list of JSON objects for movie search results
curl http://localhost:8080/search?q#matrix

[{"movie": {"released": 1999, "tagline": "Welcome to the Real World", "title": "The Matrix"}},
 {"movie": {"released": 2003, "tagline": "Free your mind", "title": "The Matrix Reloaded"}},
 {"movie": {"released": 2003, "tagline": "Everything that has a beginning has an end", "title": "The Matrix Revolutions"}}]
```

The Cypher query used, with the parameters `{query:"matrix"}`.

The search movies query ``MATCH``es the movies by title with `CONTAINS` and then ``RETURN``s the movie nodes as a list of maps with the `title`, `released` and `tagline` attributes.

```cypher
MATCH (movie:Movie)
 WHERE lower(movie.title) CONTAINS $query
 RETURN movie
```

### Get Movie

```
// JSON object for single movie with cast
curl http://localhost:8080/movie/The%20Matrix

{"title": "The Matrix",
 "cast": [{"job": "acted", "role": ["Emil"], "name": "Emil Eifrem"}, {"job": "acted", "role": ["Agent Smith"], "name": "Hugo Weaving"}, ...
          {"job": "directed", "role": null, "name": "Andy Wachowski"}, {"job": "produced", "role": null, "name": "Joel Silver"}]}
```

The Cypher query is used, with the parameters `{title:"The Matrix"}`

It matches the movie by title, then optionally finds all people working on that movie and returns the movie title and a crew-list consisting of a map with person-name, job identifier derived from the relationship-type and optionally a role for actors.

This is the Cypher statement used, it finds a movie by title and then returns for all people their name, possible roles, and the job (acted, directed, produced) as the first part of the lowercase rel-type `ACTED_IN -> acted`.

```cypher
MATCH (movie:Movie {title:$title})
OPTIONAL MATCH (movie)<-[rel]-(person:Person)
RETURN movie.title as title,
collect({name:person.name, role:rel.roles, job:head(split(toLower(type(rel)),'_'))}) as cast
LIMIT 1
```

### Graph Visualization

```
// JSON object for whole graph viz (nodes, links - arrays)
curl http://localhost:8080/graph[?limit#50]

{"nodes":
  [{"title":"Apollo 13","label":"movie"},{"title":"Kevin Bacon","label":"actor"},
   {"title":"Tom Hanks","label":"actor"},{"title":"Gary Sinise","label":"actor"},
   {"title":"Ed Harris","label":"actor"},{"title":"Bill Paxton","label":"actor"}],
 "links":
  [{"source":1,"target":0},{"source":2,"target":0},{"source":3,"target":0},
   {"source":4,"target":0},{"source":5,"target":0}]}
```

The Cypher query used finds all pairs of movies and actors and returns the movie title and a collection of all actor names as cast.
A separate function then takes this result and converts it into the node- and link-list that d3 expects.

The parameter `{limit:50}` is used to prevent the visualization from becoming a hairball.

```cypher
MATCH (m:Movie)<-[:ACTED_IN]-(a:Person)
 RETURN m.title as movie, collect(a.name) as cast
 LIMIT $limit
```

## Deployment

### Run Locally

Then setup and start the language/stack specific implementation of the backend and open the web-application on `http://localhost:8080`.

You can search for movies by title or click on any result entry to see the details.

### Deployment Options

There are many deployment options, from Docker containers or Kubernetes pods to serverless AWS-lambdas, GCP-cloud-run functions to PaaS environments like Heroku, Cloudfound or DigitialOcean App Platform.

To keep it simple we only show deployment to Heroku here, more involved deployments will be discussed later.

### Deploy to Heroku

Many of the mentioned [GitHub example repositories](https://github.com/neo4j-examples?query#movies) feature a "Deploy to Heroku" button.
You can either use that or follow the manual process outlined below.

We want to install our application to the cloud, for instance the [Heroku](https://heroku.com) PaaS.
You can either use the demo server mentioned above or the Neo4j AuraDB Free, Neo4j Sandbox or Cloud Marketplaces shown below.

Install the [Heroku Toolbelt](https://toolbelt.heroku.com/) and git.

Then run these commands:

```
# Clone the appropriate repository for your language
# initialize the git repository and add files
git init
git add .
git commit -m"my neo4j movies app"

# create heroku application, please change the app-name
export app#my-neo4j-movies-app
heroku apps:create $app

# configure your app to use the connection credentials
# e.g. bolt://user:password@host:port
heroku config:set NEO4J_URL#<bolt-url> --app $app

# or depending on the repository you might have separate username and password environment variables
heroku config:set NEO4J_USER#neo4j --app $app
heroku config:set NEO4J_PASSWORD#neo4j --app $app

# deploy to heroku
git push heroku master

# open application
heroku open --app $app
```

Then your app is ready to roll.

## Other Neo4j Database Setup Options

### Neo4j AuraDB Free

 * Log in to [Neo4j AuraDB Free](https://console.neo4j.io/?ref#developer-pages).
 * Proceed to the database creation page
 * Choose AuraDB Free
 * Confirm and save the credentials
 * Then use the connection URL and credentials for your backend setup (environment variables)
 * Populate the database using the `Movies` guide [or run this script](https://github.com/neo4j-graph-examples/movies/blob/main/scripts/movies.cypher)

### Neo4j Desktop

- [Download & Install Neo4j Desktop](http://neo4j.com/download)
- After installation, there is a "first use" graph available
- To change the password from the auto-generated one, click the three dots to Manage and choose the Admistration tab
- Open the Neo4j Browser with the blue "Open" button to explore

If you want to create a new database, do so within a Desktop Project, then start Neo4j-Browser

- Install the Movies dataset with `:play movies`
- Click the large `CREATE` statement and hit the triangular "Run" button to insert the data.

The connection details will be `bolt://localhost`, username: `neo4j` and your chosen password.

### Neo4j Sandbox

- Log in to [Neo4j Sandbox](https://sandbox.neo4j.com/).
- Create a project with the "Movies" dataset and launch it.
- The Connection Details are in the sandbox UI if you unfold the lower part with the black triangle

### Existing Language / Driver Examples

For our example application we've provided a implementation for the languages and drivers listed below.
You can find them in the [Neo4j Examples GitHub repository](http://github.com/neo4j-examples?query#movie).
Most of them have thankfully been made available by the driver authors.

#### For the Official Drivers

* Java
** [Java Bolt Driver](http://github.com/neo4j-examples/neo4j-movies-java-bolt)
** [Spring Data Neo4j 5](http://github.com/neo4j-examples/spring-data-neo4j-intro-app)
* Python
** [Python Bolt Driver](http://github.com/neo4j-examples/movies-python-bolt)
* Go
** [Golang Bolt Driver](http://github.com/neo4j-examples/movies-golang-bolt)
* Javascript
** [Javascript Bolt Driver](http://github.com/neo4j-examples/movies-javascript-bolt)

#### For Community Drivers

* Elixir: [phoenix](http://github.com/neo4j-examples/movies-elixir-phoenix) by http://twitter.com/florin
* .Net [neo4jclient](http://github.com/neo4j-examples/movies-dotnet-neo4jclient) by http://twitter.com/pierrick22 and https://twitter.com/cskardon
* Ruby [Neo4j-Core](http://github.com/neo4j-examples/movies-ruby-neo4j-core) by http://twitter.com/neo4jrb
* Ruby [Neo4j.rb Rails](http://github.com/neo4j-examples/movies-ruby-neo4jrb) by http://twitter.com/neo4jrb
* Perl: [Neo4p::REST](http://github.com/neo4j-examples/movies-perl-neo4p) by https://twitter.com/thinkinator
* PHP: [Neoclient]([http://github.com/neo4j-examples/movies-php-neoclient](https://github.com/neo4j-examples/movies-php-client)) by http://twitter.com/ikwattro and https://twitter.com/GhlenNagels
* Python Object Mapping: [Neomodel](http://github.com/neo4j-examples/neo4j-movies-python-neomodel)

