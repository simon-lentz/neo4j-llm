# Vector Indexes

## Creating Vector Embeddings of Text Content in an Existing Neo4j Database

### Vectorizing Movie Plots

The Movie Recommendation Sandbox was created by Neo4j for learning about gdbs
This database consists of over 9000 movies, 15000 actors, and just over
100000 user ratings. Additional resources and data are available at
[Neo4j LLM Fundamentals]("https://github.com/neo4j-graphacademy/llm-fundamentals/tree/main")

Each movie has a .plot property.

```cypher
MATCH (m:Movie {title: "Toy Story"})
RETURN m.title AS title, m.plot AS plot;

>>> "A cowboy doll is profoundly threatened and jealous when a new spaceman figure supplants him as top toy in a boy's room."
```

## Creating the Vector Index

The embeddings are assigned to an .embedding property on the (:Movie) node.
To search across the embeddings they are assigned to a vector index, which can
be created directly in cypher using external libraries. In this case, the
db.index.vector.createNodeIndex() procedure is used to create the index.

```cypher
CALL db.index.vector.createNodeIndex(
    indexName :: STRING,
    label :: STRING,
    propertyKey :: STRING,
    vectorDimension :: INTEGER,
    vectorSimilarityFunction :: STRING)
```

db.index.vector.createNodeIndex() expects the following parameters:

indexName - The name of the index

label - The node label on which to index

propertyKey - The property key on which to index

vectorDimension - The dimension of the embedding e.g. OpenAI embeddings consist of 1536 dimensions.

vectorSimilarityFunction - The similarity function to use when comparing values in this index, this can be euclidean or cosine.

```cypher
CALL db.index.vector.createNodeIndex(
    'moviePlots',
    'Movie',
    'embedding',
    1536,
    'cosine'
)
```

Note that the index is called moviePlots, it is against the Movie label, and it is on the .embedding property. The vectorDimension is 1536 (as used by OpenAI) and the similarity function is cosine. For additional information on the supported vector similarity functions, visit [Neo4j supported vector similarity functions]("https://neo4j.com/docs/cypher-manual/current/indexes-for-vector-search/#indexes-vector-similarity")

Generally, cosine will perform best for text embeddings, but you may want to experiment
with other functions. Typically, you will choose a similarity function closest to the
loss function used when training the embedding model. You should refer to the model’s documentation for more information.

Index creation can be verified by running

```cypher
SHOW INDEXES 
YIELD id, name, type, state, populationPercent 
WHERE type = "VECTOR";
```

The result should resemble

|  id |     name     |     type     |   state  | populationPercent |
| --- | ------------ | ------------ | -------- | ----------------- |
|  1  | "moviePlots" |   "VECTOR"   | "ONLINE" |       100.0       |

Once the state is listed as online, the index will be ready to query.
The populationPercentage field indicates the proportion of node and
property pairing.

## Loading CSV Embeddings

The LOAD CSV command can be used to load the embeddings into
the Neo4j Sandbox instance. The following Cypher loads the
embeddings CSV file, performs a MATCH query to find the (:Movie)
node with the corresponding movieId property, and then sets the
.embedding property on that node.

```cypher
LOAD CSV WITH HEADERS
FROM 'https://data.neo4j.com/llm-fundamentals/openai-embeddings.csv'
AS row
MATCH (m:Movie {movieId: row.movieId})
CALL db.create.setNodeVectorProperty(m, 'embedding', apoc.convert.fromJsonList(row.embedding))
RETURN count(*);
```

The above loads the CSV file, matches the (:Movie) node with the
corresponding movieId property, calls db.create.setNodeVectorProperty()
procedure to set the embedding property, the procedure also validates
that the property is a valid vector.

When data is loaded using LOAD CSV, it is treated as a string unless
specifically cast using a specific function, for example, toInteger()
or toFloat(). In this case, the embedding is a string representing
a JSON list, and needs to be coerced into a Cypher List.
[See APOC documentation for more details]("https://neo4j.com/docs/apoc/current/overview/apoc.convert/apoc.convert.fromJsonList/")

The index will be updated asynchronously. You can check the status of
the index population using the SHOW INDEXES statement, when the
populationPercent is 100.0, all the movie embeddings have been indexed.

## Querying Vector Indexes

You can query the index using the db.index.vector.queryNodes() procedure.
The procedure returns the requested number of approximate nearest neighbor
nodes and their similarity score, ordered by the score.

```cypher
CALL db.index.vector.queryNodes(
    indexName :: STRING,
    numberOfNearestNeighbours :: INTEGER,
    query :: LIST<FLOAT>
) YIELD node, score
```

The procedure accepts three parameters:

indexName - The name of the vector index
numberOfNearestNeighbours - The number of results to return
query - A list of floats that represent an embedding

The procedure yields two arguments; a node and a similarity score
ranging from 0.0 to 1.0. You can use this procedure to find the
closest embedding value to a given embedding.

For example, find movies with a similar plot to another:

```cypher
MATCH (m:Movie {title: 'Toy Story'})
WITH m LIMIT 1

CALL db.index.vector.queryNodes('moviePlots', 6, m.embedding)
YIELD node, score

RETURN node.title AS title, node.plot AS plot, score;
```

The query finds the Toy Story Movie node and uses the .embedding
property to find the most similar plots. The db.index.vector.queryNodes()
procedure uses the moviePlots vector index to find similar embeddings.
The procedure returns the requested number of approximate nearest neighbor
nodes and their similarity score, ordered by the score.

|  title  |     plot     |     score     |  
| ------- | ------------ | ------------- |
|  "Toy Story"      | "A cowboy doll is profoundly threatened and jealous when a new spaceman figure supplants him as top toy in a boy’s room." |   1.0    |
| "Little Rascals, The" | "Alfalfa is wooing Darla and his He-Man-Woman-Hating friends attempt to sabotage the relationship." | 0.9214372634887695 |
|"NeverEnding Story III, The"| "A young boy must restore order when a group of bullies steal the magical book that acts as a portal between Earth and the imaginary world of Fantasia." | 0.9206198453903198 |
|"Drop Dead Fred"| "A young woman finds her already unstable life rocked by the presence of a rambunctious imaginary friend from childhood." | 0.9199690818786621 |
|"E.T. the Extra-Terrestrial"| "A troubled child summons the courage to help a friendly alien escape Earth and return to his home-world." | 0.919100284576416 |
|"Gumby: The Movie"| "In this offshoot of the 1950s claymation cartoon series, the crazy Blockheads threaten to ruin Gumby’s benefit concert by replacing the entire city of Clokeytown with robots." | 0.9180967211723328 |

## Summary

As you can see, this approach is relatively straightforward and can quickly yield results. The downside to this approach is that it relies heavily on the embeddings and similarity function to produce valid results. This approach is also a black box - with 1536 dimensions, it would be impossible to determine how the vectors are structured and what factors were considered when calculating the similarity. The movies returned look similar, but you would have no way of verifying that the results are correct.

## Improving Semantic Search

When using vector-backed semantic search, you are placing a lot of emphasis on the underlying model to provide a good similarity. The context from which the similarity scores are provided is an important factor here. The embeddings were trained on a generic dataset, and as such, may identify two movies with a main character called Jack who likes fruit as movies with a close similarity. This is correct in one sense; they are both movies and very similar when compared to a cricket bat or a vanilla yoghurt. On the other hand, if you are recommending a movie to pass away a rainy afternoon, a fairy tale for children is very different to a horror film.

## Incorporating Graph Features

Graph features can enrich vector-backed semantic search by providing structural and relational context to data. By leveraging the connections in graphs, semantic search can draw upon relationships and hierarchies between entities, enhancing the depth and relevance of search results. While vectors capture semantic nuances, adding graph structures allows for a more holistic understanding of data, considering its inherent meaning and position in a broader knowledge network. for more information, listen to [Dr Jesus Barrasa and Alexander Erdl explore the differences between Vector-based Semantic Search and Graph-based Semantic Search.]("https://www.youtube.com/watch?v=bRD09ndyJNs")

## Using Feedback

A more straightforward approach may be to use user feedback to gradually re-rank and curate the content returned by a vector index search. This is the approach the GraphAcademy chatbot uses. To celebrate the launch of the Vector Index, Neo4j published an [article]("https://medium.com/neo4j/building-an-educational-chatbot-for-graphacademy-with-neo4j-f707c4ce311b") detailing how we used vector index to identify Neo4j content to improve LLM responses.
The article demonstrates how we ingested Neo4j Documentation and GraphAcademy lessons, using embeddings of the content to populate a vector index.

When a user asks a question, the server generates an embedding of it and uses the vector index to identify relevant content. This approach works well in most cases, but now and again, the index fails to return relevant content, in which case the server instructs the LLM to respond to the user with an apology. When a chatbot is unable to answer a question, or the user provides feedback, the result is stored in the database. Over time, a better picture of which documents do or do not answer specific queries emerges. When a new user question comes in, the chatbot can compare it to previous questions and exclude any suggested documents where the response was previously unhelpful.
