# BloodHound

https://blog.cptjesus.com/posts/introtocypher

## Executing BloodHound

As of v1.5, basic collection of everything:

`Invoke-BloodHound -CollectionMethod All -CompressData -RemoveCSV`

Decrease network load or modify traffic to reduce basic detection:

`Invoke-BloodHound -Throttle 1500 -Jitter 10`

Bloodhound execution is fussy when excuted over a beacon, so Rohan suggests using SharpHound.exe:

`SharpHound.exe -c All`

## Relationships

"Relationships in BloodHound always go in the direction of compromise or further privilege, whether through group membership or user credentials from a session."

### Relationship Functions

`allShortestPaths` function: Finds every shortest path from each node to your target. Note that this results in significantly more data being returned and frequently isn’t something we need.

### Wildcard Matches

Here’s the query that’s run each time you type a letter in the search bar:

`MATCH (n) WHERE n.name =~ “(?i).*searchterm.*” RETURN n LIMIT 10`

Return any nodes of any type that match the search term given.
* `?i` tells Neo4j this is a case insensitive regex
* `.*` on each side indicates that we want to match anything on either side.
* `LIMIT 10` limits the number of items returned to the first ten.

## Advanced Queries

WITH keyword example - The UI calculates the number of sessions for a group being viewed using two separate queries put together:

`MATCH p=shortestPath((m:User)-[r:MemberOf*1..]->(n:Group {name: {name}})) WITH m MATCH q=((m)<-[:HasSession]-(o:Computer)) RETURN count(o)`
￼
### Saving Custom Queries

As of BloodHound v1.2, a JSON file is saved in the Electron user directory associated with BloodHound. You can edit the file by clicking the crayon next to Custom Queries on the UI, which will ask your OS to open the file in whatever your default is. On Windows, the file falls under AppData\Roaming\bloodhound. Anything in this file will be loaded into the BloodHound user interface whenever it initializes.

Saving a simple query that doesn’t require any user input, like Find All Domain Admins:

```
{
    "queries": [
        {
            "name": "Find all Domain Admins",
            "requireNodeSelect": false,
            "query": "MATCH (n:Group) WHERE n.name =~ {name} WITH n MATCH (n)<-[r:MemberOf*1..]-(m) RETURN n,r,m",
            "allowCollapse": false,
            "props": {"name": "(?i).*DOMAIN ADMINS.*"}
        }
    ]
}
```

The intro tag for the JSON is “queries,” which corresponds to a list of the queries you’re interested in. For the simple queries, only a few pieces of data are required:

* name - The name of the query to display on the UI
* requireNodeSelect - Setting this to “false” designates this as a query that does not require user input
* allowCollapse - Setting this to “false” will prevent the UI from collapsing nodes when running this query, logic which is used to minimize the number of nodes drawn on screen
* query - The actual query to run.
* props - Properties to pass into the query

It’s important to note that properties, such as names, should not be passed directly into the query, but as properties. In this example, we add {name} into the query. This corresponds to the appropriately named parameter in the props object. Using properties this way ensures that escaping of strings is done properly when sending the property data to the database.
The second, more complicated query is a query that needs user input of some sort. The pre-built query is structured with two different queries this time. An example is Shortest Paths to Domain Admins: query

```
{
    "queries": [
        {
            "name": "Find Shortest Paths to Domain Admins",
            "requireNodeSelect": true,
            "nodeSelectQuery":  {
                "query":"MATCH (n:Group) WHERE n.name =~ {name} RETURN n.name",
                "queryProps": {"name":"(?i).*DOMAIN ADMINS.*"},
                "onFinish": "MATCH (n:User),(m:Group {name:{result}}),p=shortestPath((n)-[*1..]->(m)) RETURN p",
                "start":"",
                "end": "{}",
                "allowCollapse": true,
                "boxTitle": "Select domain to map..."
            }
        }
    ]
}
```

This query has more properties to it:

* name - The name to display on the UI
* requireNodeSelect - We set this to “true”, this time to indicate we’re doing a more complicated query with user input
* nodeSelectQuery - This JSON object contains all the actual query data
* query - The query to be run to get user input
* queryProps - Properties to pass to the user input query
    * onFinish - The query to run with your user input. Note that we use the keyword result in order to slot in our user input. This is important, as this is the name of the property passed into the onFinish query
    * start - The name of the start node of the query. By passing in {} the result of user input will be substituted there.
    * end - The name of the end node of the query. By passing in {} the result of user input will be substituted there.
    * allowCollapse - Setting this to “false” will prevent the UI from collapsing nodes when running this query, logic which is used to minimize the number of nodes drawn on screen
    * boxTitle - The title of the user input box that will be displayed
￼
Whichever one of these options is selected by the user will be passed into the onFinish query and the result will be displayed on the user interface. These two options let you make simple or complicated queries that can be persisted in the user interface. Because the queries are saved in your customqueries file, even when BloodHound updates, the queries will still be available to you for use.

## Crunching Numbers

Count Local Admins where Domain Admins have sessions:

```
MATCH p = (u1:User)-[r:MemberOf|AdminTo*1..]->(c:Computer)-[r2:HasSession]->(u2:User)-[r3:MemberOf*1..]->(g:Group {name:’DOMAIN ADMINS@EXTERNAL.LOCAL’})
RETURN COUNT(DISTINCT(u1)) AS adminCount,c.name as computerName
ORDER BY adminCount DESC
```

Calculate the percentage of users with a path to DA:

```
MATCH (totalUsers:User {domain:'EXTERNAL.LOCAL'})
MATCH p = shortestPath((pathToDAUsers:User {domain:'EXTERNAL.LOCAL'})-[r*1..]->(g:Group {name:'DOMAIN ADMINS@EXTERNAL.LOCAL'}))
WITH COUNT(DISTINCT(totalUsers)) as totalUsers, COUNT(DISTINCT(pathToDAUsers)) as pathToDAUsers
RETURN 100.0 * pathToDAUsers / totalUsers AS percentUsersToDA
```

Note: This really needs to be done via Python and the Neo4j Bolt driver. It'll probably run forever in the Neo4j console.

Calculate the percentage of computers with a path to DA:

```
MATCH (totalComputers:Computer {domain:'EXTERNAL.LOCAL'})
MATCH p = shortestPath((pathToDAComputers:Computer {domain:'EXTERNAL.LOCAL'})-[r*1..]->(g:Group {name:'DOMAIN ADMINS@EXTERNAL.LOCAL'}))
WITH COUNT(DISTINCT(totalComputers)) as totalComputers, COUNT(DISTINCT(pathToDAComputers)) as pathToDAComputers
RETURN 100.0 * pathToDAComputers / totalComputers AS percentComputersToDA
```

Calculate the average attack path length:

```
MATCH p = shortestPath((n {domain:'EXTERNAL.LOCAL'})-[r*1..]->(g:Group {name:'DOMAIN ADMINS@EXTERNAL.LOCAL'}))
RETURN toInt(AVG(LENGTH(p))) as avgPathLength
```

Calcuate average group membership **without** recursion (only direct group membership):

`MATCH (u:User {domain: "EXTERNAL.LOCAL"})-[r:MemberOf]->(g:Group) WITH u.name as userName,COUNT(r) as relCount RETURN AVG(relCount)`

Calcuate average group membership **with** recursion (unrolled group membership):

`MATCH (u:User {domain: "EXTERNAL.LOCAL"})-[r:MemberOf*1..]->(g:Group) WITH u.name as userName,COUNT(r) as relCount RETURN AVG(relCount)`

Note: Remove the `{domain: "EXTERNAL.LOCAL"}` filter if you want to query all domains in the data together.
