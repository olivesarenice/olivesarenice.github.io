---
title: Apache Cassandra RAG
tag: software
category: learning
---

# Summary

In this post I cover my learnings from attending the recent AI Camp workshop where amongst other sharings, I learned how to utilise Apache Cassandra via [Datastax](), a cloud platform, to run a chatbot powered by Retrieval Augmented Generation (RAG).

It follows the tutorial from [Retrieval Augmented Generation for AI Chatbots](https://colab.research.google.com/github/awesome-astra/docs/blob/main/docs/pages/tools/notebooks/Retrieval_Augmented_Generation_(for_AI_Chatbots).ipynb) in the [Datastax Docs](https://docs.datastax.com/en/astra-serverless/docs/vector-search/examples.html)

The presenter was [Erick Ramirez](https://au.linkedin.com/in/erickramirez) who is a community partner and open-source contributor to the Apache Cassandra project. [AI Camp](https://www.aicamp.ai/) is a community of people interested in learning AI and has many charters around the world. The Singapore charter is headed by [Matthias Baetens](https://sg.linkedin.com/in/matthiasbaetens) who is an open-source developer for Apache Beam.

The components covered in the talk were new to me so I will need to break each of them into thier own sections and record all my notes on each item.

Components:

- Overall functionality RAG chatbot

    - What is RAG? 
    - Advantages of RAG over regular LLMs?
    - How a basic RAG works

- Apache Cassandra & Datastax

    - What is a vector search database?
    - What is a NoSQL wide-column store vs. other types of databases?
    - What type of database is Cassandra?

## RAG Chatbot

### What is RAG?

[Retrieval Augmented Generation](https://research.ibm.com/blog/retrieval-augmented-generation-RAG) (RAG) is a way to pass external information to an LLM that will help it generate more accurate replies to prompts. In simple terms, it is the difference between an LLM relying on 'closed-book' vs. 'open-book' answers.

LLMs rely on its billions of parameters to encode 'external' information about the world - it has no other reference other than the data it was trained with. For applications where an LLM needs specific context information, RAG provides a way to *retrieve* this information for the model.

[Unlike fine-tuning](https://www.rungalileo.io/blog/optimizing-llm-performance-rag-vs-finetune-vs-both) which makes the model better a performing specific tasks, RAG improves the model through intrinsically grounding the model with accurate information. This information is also stored in a vector database which is constantly updated. Hence, we do not have to worry about re-training the model to include more up-to-date information.

RAG is best suited for a Q&A type of application where questions have defined answers. The same concept can be used for other forms of semantic object retrieval, such as images, audio waveforms, etc.

### Data flow in a RAG

Like the name suggests, RAG operates first by **retrieval** and then **generation**.

0. A vector database is first prepared with question (context) and answer (knowledge) pairs.

    The context is encoded using a pre-trained transformer to produce vector embeddings. These are just an array of numbers of N dimensions (where N varies by the LLM) that describe the text in the context. Thus, we can do a vector search to find the most similar vectors (context) and their associated knowledge. Since vector search is searching for semantic similarity, this means the knowledge returned by the search should also be semantically relevant to the original input.

1. Encode the user input into vector embeddings

    Using the same pre-trained transformer as the vector database, we convert the input text into a vector embedding. 

        embedding = openai.Embedding.create(input=customer_input, model=model_id)['data'][0]['embedding']

2. Do a vector search to **retrieve** semantically relevant data

    Cassandra uses an Approximate Nearest Neighbour search (ANN) to compare the similarity of the input vector to all vectors in the database. There are various options for similarity metrics such as euclidian distance, cosine similarity, and dot product. Euclidian distance is just the absolute distance between 2 vectors. Cosine similarity looks at the angle between 2 vectors. Dot product looks at the magnitude and angle between 2 vectors. If the vectors are already normalised by the transformer, then cosine and dot product are the same.

    The vector search will return the top N nearest vectors and the 'answers' associated with these vectors.

    In Cassandra, you can simply get the ANN scores of each row and then order them to keep the top N rows:

            SELECT *
            FROM squad
            ORDER BY title_context_embedding ANN OF {embedding} LIMIT 3;

3. Pass this knowledge to another LLM (ChatGPT) to process and generate an output in the context of our user input.

    Now instead of the ChatGPT replying to the user's input directly, we tell it to answer the user based on the additional knowledge we have provided. 

    We need to first define the roles so that ChatGPT knows how to use the additional info we give it.

        {"role":"system",
            "content":"You're a chatbot helping customers with questions."}

    Then give the user input

        {"role":"user",
            "content": customer_input})

    Then give the extra information, and repeat for however many additional rows of knowledge we want to feed it

        {'role': "assistant", 
            "content": f"{row.context}"}

    Then prompt it to reply with the added info.

        {"role": "assistant", 
            "content":"Here's my answer to your question."})


As an example, I asked the RAG bot and ChatGPT the same question:


    ChatGPT:

    >> Beyoncé's fourth solo album, titled "4," was released in 2011. She drew inspiration from various sources for this album, including a range of musical genres such as R&B, funk, and Afrobeat. Beyoncé has mentioned that she was influenced by a diverse array of artists and styles while creating "4," including Fela Kuti, Lauryn Hill, Michael Jackson, Earth, Wind & Fire, and Prince. The album showcases a departure from her previous sound and explores a more eclectic mix of musical elements.

    RAG modified ChatGPT:

    >> Beyoncé has cited various influences for her fourth solo album, "4". In particular, she has mentioned that the album was inspired by iconic musicians such as Fela Kuti, Earth, Wind & Fire, DeBarge, Lionel Richie, Teena Marie, The Jackson 5, New Edition, Adele, Florence and the Machine, and Prince. Additionally, she drew inspiration from various genres such as 1990s R&B and 1980s pop.

The RAG modified version is more precise and manages to get 'Florence and the Machine' which regular ChatGPT misses no matter how much you repeat the question. If you ask it about 'Florence and the Machine' it is not able to definitively tell you that they were an influence, although it is directly cited in the Wiki article.

# Vector Databases (Cassandra & Datastax)

    - What type of database is Cassandra?
    - What is a NoSQL wide-column store vs. other types of databases? https://scaleyourapp.com/wide-column-and-column-oriented-databases/
    https://dandkim.com/wide-column-databases/
    - What is a vector search database?
    - What is Datastax?

Cassandra is a NoSQL database that is used by many internet companies. It supports SQL query syntax (CQL) and the added option of vector searching. Datastax is a cloud platform that hosts instances of Cassandra databases for you. They have a generous free tier of "up to 80GB storage and 20 million read/write operations (using free $25/mo credit)". 

Setting up a database can be done in a few clicks:

{% include img-wrap name="datastax-create.png" caption="Datastax DB Setup" %}

From here I dived into a rabbit-hole of database terminologies and the differences between each type of database.

## SQL vs. NoSQL databases

First, SQL vs. NoSQL, is technically a misnomer. The debate is between traditional RDBMS and non-RDMBS architectures. 

After reading multiple articles, I can say that the main difference between SQL and NoSQL databases is the that SQL data structures are pre-defined, and hence rigid. The main appeal is that this fixed structure (schemas, table relations, column types, etc) allows SQL databases to be better suited for applications where incoming data is well-defined, and will not change over time. Additionaly, the rigidity of SQL databases ensure that all transactions are consistent, which is very important in Online Transaction Processing (OLTP) where the goal is data capture and retrieval. Examples include bank transactions, health records, etc. Thus, data integrity as enforced by Atomiticity, Consistency, Isolation, Durability (ACID) principle is a defining feature of SQL databases.

On the other hand, NoSQL databases are much more flexible in how data is stored. Table structures can change, and columns need not be enforced to fit 1 type of data. Examples of NoSQL database structures include ([from AWS](https://docs.aws.amazon.com/whitepapers/latest/choosing-an-aws-nosql-database/types-of-nosql-databases.html)):

- Key-value pair (Amazon DynamoDB, Redis, Apache Cassandra)

    {% include img-wrap name="key-value-pairs.png" caption="KeyVal" %}

    Value can be anything - string, object, another data structure, etc. Basically a hashmap.

- Document-oriented (MongoDB, Google Firestore)

    {% include img-wrap name="document-data-model.png" caption="Document" %}

    Typically stores a 'document' as lists, arrays, or nested elements, with the document tagged to a key.

- Column-oriented (Apache Cassandra, Apache HBase, Google BigTable, Azure Cosmos DB)

    {% include img-wrap name="wide-column-data-store.png" caption="Wide Column" %}

    Flexibility that allows for a column family to contain different types of columns, depending on the row. Hence, moves away from rigid SQL table structure and allows a wide variety of data to be stored as columns.

- Graph-based (Neo4j, RedisGraph)

    {% include img-wrap name="social-network-graph.png" caption="Graph" %}

    Used when there is highly connected data. Such as many-to-many relationships in a user profile database.

- Time series (InfluxDB, TimescaleDB)

    {% include img-wrap name="series-data-model.png" caption="Time Series" %}

    Contains a table with multiple series, within which contains multiple records indexed by timestamp.



Some databases can handle a combination of data structures - e.g. Apache Cassandra is a 2-dimensional key-value store, and also a wide-column database.

Typically, SQL databases are meant to be run on a single node, hence scaling is done vertically. NoSQL databases are meant to be scaled horizontally across multiple nodes. Thus, for cloud applications across multiple regions, NoSQL databases are usually preferred.

## Wide-column store Databases

There are 3 types of databases covered in [this article](https://scaleyourapp.com/wide-column-and-column-oriented-databases/):

- **row-based (MySQL and PostgreSQL)**

    All columns in a row are partitioned together in the same block on a disk. Hence, retrieving all columns from a row is fast as the entire block can be read without switching. These are best suited for OLTP operations where the entire row is usually read or written to the database.

- **column-based (Google BigQuery and Amazon Redshift)**

    All rows in a column are partitioned together in the same block on a disk. This is the opposite of the row-based structure. These are best suited for OLAP operations where aggregations/ calculations are done on specific columns, and aggregates over all the rows of the column. If an OLAP aggregation was done on a row-based database, the query would go into each row and then scan across the columns of that row to retrieve the data, and then repeat this scan for all rows in the table before it is able to aggregate over the retrieved values. Conversely, writing a new record means that each column must be accessed and updated, leading to slower writes.

- **column-family (or wide-column) based**

    {% include img-wrap name="wide-column" caption="Wide Column Stores" %}

    Rows can have column families which contain any number of columns. Rows do not have to have the same columns. If a column is queried on a row which does not have that column, it simply returns `NULL` rather than throwing an error.

**Cassandra**

The main advantage of Cassandra is not in the data structures it allows, but that it is built for cloud applications. Each Cassandra instance runs on a separate server node which replicates changes to the database across the entire peer2peer node network. Thus, there is no single-point of failure and nodes can be scaled up or down depending on query traffic.

## Closing

This was my first meetup in a long time and I am glad that I signed up. Back in Israel we had the habit of signing up for meetups once a week to get to know the experts and enthusiasts in the various fields of tech and business. I am trying to get back into that habit by finding similar meetups here in Singapore.

*That's all for tonight, ciao.*