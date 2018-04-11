# Example MongoDB Graph Lookup Linking People With Hierarchical Geographical Places Data

## Background

MongoDB demo to re-create the graph example cited in the book [Designing Data-Intensive Applications](https://dataintensive.net/) by _Martin Kleppmann_ (Chapter 2: 'Data Models and Query Languages', section 'Graph Like Data Models'). The book discusses an example graph data structure (hierarchy of geographical places within places, with a set of persons, each having 'born_in' and 'lives_in' attributes). The book provides examples of how this would be modelled in a graph database and in a regular relational database and then provides an example of how to query the data using a specific graph database query language (Cypher) and using SQL. The query example scenario is: __Find People Who Emigrated From US to Europe__. Specifically, this GitHub project is to demo how the same example scenario can be fulfilled using MongoDB's Document model and Aggregation pipeline.

The two main collections to be stored in MongoDB for the demo are:

* __places__  - Contains hierarchical geographical places data with the graph structure of: __SUBDIVISIONS-->COUNTRIES-->SUBREGIONS-->CONTINENTS__ (e.g. England-->United Kingdom of Great Britain and Northern Ireland-->Northern Europe-->Europe)
* __persons__ - Contains ~1 million person records where each person has 'born_in' and 'lives_in' attributes, which each reference a 'starting' place record in the places collection

Similar to the book's example, amongst the many persons records stored in MongoDB for the demo, are the following two records relating to the persons 'Lucy' and 'Alain':

    {fullname: 'Lucy Smith',   born_in: 'Idaho',                   lives_in: 'England'}
    {fullname: 'Alain Chirac', born_in: 'Bourgogne-Franche-Comte', lives_in: 'England'}


## Steps To Run

_Note: It is assumed that you've already installed MongoDB (version 3.6 or greater) on a laptop/workstation, you've already started running a MongoDB database instance and you are familiar with using MongoDB's command line tools._

    # Unzip places.json.zip & persons.json.zip first, then import:
    $ mongoimport -d placedata -c places --type json places.json
    $ mongoimport -d placedata -c persons --type json persons.json

    $ mongo

    use placedata

    // Define index ready for use by the subsequent graph lookups
    // (with the index in place the main scenario takes around 2 
    //  seconds to run versus ~45 seconds if no index is defined)
    db.places.ensureIndex({name: 1})

    // Example: Find birth location line for Lucy Smith
    db.persons.aggregate([
        {$match: {fullname: 'Lucy Smith'}},
        {$graphLookup: {
            from: 'places',
            startWith: '$born_in',
            connectFromField: 'part_of',
            connectToField: 'name',
            depthField: 'depth',
            as: 'birth_location_line'
        }}
    ]).pretty()

    // Example: Find people who emigrated from 'United States of America'
    // (born in) to 'Europe' (lives in)
    var born = 'United States of America', lives = 'Europe'
    db.persons.aggregate([
        {$graphLookup: {
            from: 'places',
            startWith: '$born_in',
            connectFromField: 'part_of',
            connectToField: 'name',
            depthField: 'depth',
            as: 'born_location_line'
        }},
        {$match: {'born_location_line.name': born}},
        {$graphLookup: {
            from: 'places',
            startWith: '$lives_in',
            connectFromField: 'part_of',
            connectToField: 'name',
            depthField: 'depth',
            as: 'lives_location_line'
        }},
        {$match: {'lives_location_line.name': lives}},
        {$project: {
            _id: 0,
            fullname: 1, 
            born_in: 1, 
            lives_in: 1, 
        }}
    ])

Example Output:

    { "lives_in" : "England", "fullname" : "Lucy Smith", "born_in" : "Idaho" }
    { "lives_in" : "Eastern Europe", "fullname" : "Travis Mc243", "born_in" : "West Virginia" }
    { "lives_in" : "Trenciansky kraj", "fullname" : "Simon Mc1093", "born_in" : "South Dakota" }
    { "lives_in" : "Lovrenc na Pohorju", "fullname" : "Sandy Mc1666", "born_in" : "Connecticut" }
    { "lives_in" : "Jonkopings lan", "fullname" : "Gertrude Mc1718", "born_in" : "Alaska" }
    { "lives_in" : "Sofia", "fullname" : "Bobby Mc1810", "born_in" : "California" }
    { "lives_in" : "Ilinden", "fullname" : "Simon Mc1839", "born_in" : "North Carolina" }
    { "lives_in" : "Zebbug Gozo", "fullname" : "Sandy Mc2165", "born_in" : "Texas" }
    { "lives_in" : "Gradsko", "fullname" : "David Mc2409", "born_in" : "Michigan" }
    .......
    .......
