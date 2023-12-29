---
title: DE Interview Notes
tag: others
category: learning
---

Preparing for DE interview from scratch.

According to Chris Garzon who wrote the e-book [Ace the Data Engineer Interview](https://dataengineerinterview.com/), the interview can be broken into these components:

1. Technical understanding of SQL/ Python

    This tests technical coding skills - see Leetcode and Stratascratch.

2. Understanding of database concepts

    May be technology specific as well.

3. CI/CD

    Understands Git and how to write good unit tests

4. Data Models

    The ability to design an appropriate schema given the business requirements. This requires understanding the foundational data engineering concepts like ELT/ETL and processing

5. System Design

    Extension of data model for a business process.

6. Cloud platforms

    Familiarity with the various technologies - compute, database, storage, scaling, network load balancers, gateways, etc.

7. Behavioural

    Sharing about past experiences.


## Application

Interview questions will always go back to the company and its business context/ processes. Therefore, do recon on the tech stack involved and their business models to anticipate what kind of requirements they have.

In this example, I use NetEase Games, which I have an interview for soon. I try to be as **methodical and deliberate** as I can in learning about them and what to prepare for.

## Company Recon

### About

NetEase Games is the online games division of NetEase, Inc. developing and operating some of the most popular mobile and PC games in markets including China and Japan. Starting 2015, they started to map out distribution to overseas markets.

### Job Description

The JD lists the following items:

- Responsible for the **operation** and maintenance of the big **data platform on public clouds**, including deployment, management, troubleshooting, and optimization of open-source components and in-house platforms to ensure platform stability.

- **Understand the architecture and business processes** of the big data platform, resolve issues encountered during business operations, and identify and resolve performance bottlenecks.

- Develop scripts and tools for automation in operations, including monitoring, alerting, and fault handling.

- Familiarity with the Linux operating system, TCP/IP fundamentals, and experience in system tuning are preferred.

- Proficiency in Shell and Python scripting languages.

- Familiarity with the architecture of public cloud platforms such as AWS, GCP, and possess skills in **cloud-based business architecture** design and operations.

- Familiarity with operations systems, skilled in problem-solving, independent resolution of common issues, and experience in **managing large-scale clusters** are preferred.

- Proactive, responsible, meticulous, with strong learning and communication abilities.

- Familiarity with the architecture and basic principles of **big data components** such as Hadoop, Kafka, Flink, Elasticsearch are preferred.

### Role Information

I also got additional info from a screening call with HR:

- The SG group (newly created and 1 pax) is responsible for data warehousing, support functions, cloud infrastructure

- Engineer will work with established team in China (> 10y experience) to transfer knowledge on database maintenance to SG

This suggests that all-rounder competency, especially in cloud is preferred, so I will focus my preparation on that.

### Recon

A quick search of 'Netease Games Cloud Architecture' returns this: [AWS Case Study: NetEase Games](https://aws.amazon.com/solutions/case-studies/netease-games-case-study/)

The article describes Netease Games as using multiple VPCs to segregate their platform services from their game servers, and also to connect their on-prem data servers in China (which are now facing privacy regulations) together.

The main benefit is geographical scalability outside of China. 

The diagram shows that each region has its own:

- platform services VPCs - virtual networking, database, chat/ audio servers

- game server VPC - each with multiple zones

We might benefit from keeping this business architecture in mind when talking about the business requirements.

## Technical understanding of SQL/ Python

N/A. Have to rely on past experience to perform on the day.

## Understanding of database concepts

1. SQL vs. NoSQL
    
    i.e. RDMBS vs. non-RDBMS. Main point is to remember that NoSQL are used for horizontal scaling across multiple nodes/ typically across regions for availability.

    SQL is better for well-defined data models, where data integrity is essential.

2. Sharding vs. Partitions

    Sharding and partitioning are both strategies used in database design to **improve performance and manageability**, but they operate at different levels and have some distinct characteristics.

    Shards are smaller **independent databases across multiple nodes**. Typical example is sharding the database by region, so that requests coming from a certain region are fulfilled by the same-region shard to decrease latency.

    Partitions break the database **on a single node**. For example, part of a table (older rows) can be partitioned off to a slower server while newer rows remain on the high powered server, as they are accessed more frequently.

3. Primary Keys vs. Clustering

    PKs are unique identifiers for the row. Typically, a PK is indexed and thus can be used for sorting and aggregation.

    Clustering is a way to organise the data on the physical storage media. Rows on the same cluster are faster to read than rows on another cluster.

4. Processing patterns

    Know the difference between the different types of data loading patterns. 

    Batch: Processes data at regular intervals. Loads large volumes of data. 
    
    Stream: Event-driven, updates close to real-time. Small chunks or rows at a time.

    Delta: only load new rows by an identifying column (usually time). Useful for saving processing power. Can be run as scheduled batch or real-time.

    Balance between:

    Data integrity + simplicity (Batch) vs. Speed + Availability (Stream)

5. ETL vs. ELT

**ETL:** Data is transformed as it is moving to the storage location.

Example: Data is downloaded from Salesforce using a local server. The local server does all the transformation after it has gotten the data. It then dumps the cleaned data into a data warehouse. This relies on the compute of the local server rather than the data warehouse.

**ELT:** Data is transformed only after is has been loaded into the storage location

Example: Data streamed from various APIs (bottlenecks) should be first dumped as-is into a dAatalake which then has the capability and bandwidth to perform high volume transformation. We do not want to be modifying the data as it is travelling from the API to our storage.

Another example: Data is downloaded in bulk from Salesforce and dumped into S3 as-is. Spark is then used to parallel process and transform the data into a warehouse.

The main difference is **where the transformation is taking place** - either in a dedicated (ETL) transformation server, or inside the (ELT) loaded database.

Most of the time, the process is a mix of EL-ETL where data is dumped into intermediate data lakes and then transformed as they move to another destination data warehouses.

## CI/CD

Other than basic Git, also need to know what kind of unit tests and what to test for. In data processing, some examples of things to check for :

- for batch loads, total rows processed written vs. total rows in data source
- no. of unique rows
- distribution of IDs
- NULL values

## Data Models

Most important part.

1. Dimensional Models

    Similar to Activity schema. Where a business can be broken down into 'an **object** does an **activity**'
    
    Fact table = Activity table, Object table = Dimension table.

    So a **customer** makes an **order** on the **vendor's shop** leading to a **sale**. Gives `dim.customers`, `fact.orders` and `dim.vendor_shop`, `fact.vendor_sales`.

    For Netease as a gaming company, ChatGPT gives us a general idea of the types of data we will be dealing with.

    In general, **players** start **sessions** for specific **games** hosted on various **servers** in **AWS VPCs** which handle **network requests**, they may also **purchase** items through the game **platforms**. So we have a skeleton for our data model already. 

    ```ChatGPT

    PROMPT >> 
    
    Provide a general data model for a gaming company with multiple game titles and operates worldwide. Focus on the IT infrastructure such as game server and network traffic

    REPLY >>

    [dim.game_title]
    GameID (Primary Key)
    Title
    Genre
    Release Date
    Description
    Developer

    [dim.game_server]
    ServerID (Primary Key)
    GameID (Foreign Key)
    VPCID (Foreign Key)
    Server Name
    Location
    IP Address
    Status (Online, Offline)
    Capacity (Maximum Players)
    Current Players
    24-hour Latency

    [dim.aws]
    VPCID (Primary Key)
    Region
    Zones
    IP Range
    ... repeat for storage, compute instances, IAM accounts etc.

    [dim.player]
    PlayerID (Primary Key)
    Username
    Email
    Password
    Date of Birth
    Country
    Language
    Avatar

    [dim.region]:
    RegionID (Primary Key)
    Region Name
    Countries

    [dim.localization]:
    LocalizationID (Primary Key)
    GameID (Foreign Key)
    Language
    Subtitle
    Audio

    [fact.sessions]:
    SessionID (Primary Key)
    PlayerID (Foreign Key)
    GameID (Foreign Key)
    ServerID (Foreign Key, Nullable)
    Playtime
    Achievements
    High Scores

    [fact.microtransaction]:
    TransactionID (Primary Key)
    PlayerID (Foreign Key)
    GameID (Foreign Key)
    ServerID (Foreign Key, Nullable)
    Purchase Date
    Transaction Amount
    Currency
    Payment Method

    [fact.network_logs]:
    LogID (Primary Key)
    Timestamp
    PlayerID (Foreign Key)
    GameID (Foreign Key)
    ServerID (Foreign Key)
    IP Address (Foreign Key)
    Data Sent
    Data Received
    Latency

    [dim.server_logs]:
    LogID (Primary Key)
    ServerID (Foreign Key)
    Timestamp
    Status (Online, Offline)
    Packet Loss

    This data model incorporates elements related to game titles, player information, game servers, network traffic, transactions, and server status logs. It also considers regional information for game platforms and localization for multiple languages.

The considerations provided by ChatGPT summarises the main concepts to mention:

    - The relationship between players and game servers allows for tracking player activities on specific servers.

    - The transaction table captures details of in-game purchases, including the server involved in the transaction.

    - Network traffic logs can be useful for monitoring and optimizing the gaming experience, including latency measurements.

    - Server status logs provide a history of server availability.

## System Design

See this example of a [system design interview for Tinder](https://youtu.be/i7twT3x5yv8?si=ps-lbTGn1R1O6If_)

My notes from the video:

1. Start with main features for the app 

    - list out each feature, these are the features we will explore individually later

    - mention monitoring and logs

    - mention additional functionality for subscriptions or premium/ paid users

2. List out usage and traffic estimations to plan the scale needed

    - users, requests, storage, latency, regions

3. Walk through the feature and high level components needed, including request flow

    - clarify requirements for feature

    - data structure needed for each part and their dtypes

    - tie in any algorithms to the feature requirements 
    
    - matching, closest proximity, lowest cost, etc.


4. For databases:

    - typically we have a user db / content map db (SQL) and blob storage (noSQL) 

    - consider sharding databases based on a useful metadata (location, usage)

    - recommend DB services/ clients fit for purpose 

        - key-value (Redis cache)
        - document (MongoDB)
        - time-series (TimescaleDB)
        - columnar (most popular for scalability, many options Google BigTable, AWS Redshift, Apache Cassandra)
        - RDBMS 

    - when deciding between 2 product types (eg. SQL vs. NoSQL), define what they are and their differences are first

## Cloud platforms

    Familiarity with the various technologies - compute, database, storage, scaling, network load balancers, gateways, etc.

## Behavioural

***Share about a data pipeline that you built***

A pipeline always consists of 4 parts. Construct the story based on this template:

Business requirement/ need --> Pipeline:

[ Data Source > Business Logic > Destination ] -  Scheduler

This ties in with the ETL/ ELT terminology, where ETL/ELT describes what happens in the Business Logic step that converts source data into useable data at the destination.

My personal example (1): <br> ETL of time-series sensor data and batch quality data for control tower dashboard.

1. Business requirement

    Need for a dashboard which aggregated data from various parts of the business, including SAP, manufacturing equipment, lab testing software.

2. Data sources:

    - Tabular product quality/ transaction data from Azure Cloud Database.

    - Time-series sensor streams stored in InfluxDB, along with timestamp manufacturing metadata in on-prem SQL Server tables. 80+ sensors over 10 years (1000s of manufacturing batches)

    - Experimental testing data from on-prem Oracle database of our Lab Information System (LIMS) (~20GB of data spread across multiple tables)

3. Business logic:

    All data belongs to individual `production_batch`, which are categorised by their `product_type`. We want to analyse and display the product quality, sensor data, and testing data, by `production_batch`.

    - Tabular product quality data from Azure:
    

    - Time-series sensor streams:

        Used the associated metadata in SQL Server to get the start and end times for each `production_batch`, then query Influx with these time-ranges to get relevant time-series.

        Downsampled time-series from millisecond precision to 1 min for dashboard display.

    - Experimental testing data:

        Oracle DB is used for system operation OLTP, **not** meant for OLAP.

        Reverse engineered the data model based on available tables and business info needed for OLAP.

        Custom query to join various tables while bypassing the nested views that were being used by other analysts (query duration from 1 day to < 1 hour)

        Delta load based on record's timestamp. Loaded via `INSERT FROM TEMP` table to allow insert in batches.

    - Transaction data from Azure: 

        Source had no PK, index, or timestamp for deltaload. IDs are non-consecutive.

        No transformation required.

        Create proxy PK consisting of (transaction_id, transaction_type) and deltaload by looping over range of transaction_id. 
        
        Range is dynamically determined to ensure load sizes are always ~ 5000 rows to prevent Azure timeout.

4. Destination:

    All transformed data was stored in an on-prem SQL Server database meant for warehousing purposes. 
    
    Within the warehouse, tables were stored with appropriate indexing for their queries - e.g. indexed on `product_name` for quick filtering.

    Views created on common table aggregations, allows users to extend their own queries more easily.

5. Scheduling:

    All jobs ran daily on in-house parallel task orchestrator (used k8s). Scheduler is `cron` based.
    
    Used Grafana to monitor health of tasks and data consistency (e.g. number of rows present vs. written)

***How would you approach the design of a datawarehouse?***

Assuming this question is open-ended with no business context, first clarify if there is a specific business requirement. Otherwise, we can ask the follow these general steps:

1. Business Objectives:

    - What is the overall strategic purpose of this data warehouse?
    - What business decisions will this data be used for?
    - What metrics are needed to make these decisions?

2. Data Requirements:

    - What data sources are available/ required? e.g. game servers, players, network traffic, game titles

    - What is the nature of these data sources? 
    
        Type: structure/ unstructured

        Content: profiles, logs, chats, metrics

3. Data Modeling:

    - What schema best captures the relationship between all these sources?

    - How will the data be queried? This determines how the data will be partitioned and indexed, which will affect scalability and performance.

    See above.

4. ETL Process:

    - What configuration of ETL/ELT is needed?

        - Frequency/ latency of data

        - Volume processed

        - Consistency

    - Are the data connectors for the various data sources available?

    - Design the ETL including what technologies will be used.

5. Data Quality and Cleansing:

    - What are the possible errors that could happen with ETL at each component?

    - Implement unit tests to check for data quality/ integrity

6. Security and Compliance:

    - Which user groups need access to which parts of the data warehouse?

    - Consider an IAM solution to manage this.

7. Integration with Analytics Tools:

    - What BI apps will be used with the warehouse? Do they have built in connectors or is a custom format needed?

    - What kind of APIs will users need to query data for their own use outside of BI apps?

8. Monitoring and Maintenance:

    - Track performance of warehouse queries, load times, storage space, etc.

    - Maintain backups, re-indexing, query optimisation

    - Maintain documentation


## Notes after interview

EDIT: I just completed the interview - unlikely to get to the next round. Here's some thoughts.

He asked 2 questions:

1. Share your experience at work.

I shared the various pipelines I built, what the processed data was used for, and what impact it contributed overall to the site.

At each point I also talked about the specific considerations on why certain methods were used, touching on key concepts and technologies such as - SQL, Python, Task Orchestration, server infrastructure, etc.

2. You talked about scheduling, what tools do you use to manage the quality of data going into storage - for example if certain jobs rely on others, or if there are errors.

I shared about the task orchestration software built in-house but based on the concepts from Airflow (so that he can relate). This includes things like tasks, plans, queues, workers, etc. And then I shared about how I use exit() codes and errors to manage the task flow. I also added how I took initiative to build my own dashboard that reads the logs from these jobs and monitor their health. Lastly, I talked about maintaining data quality by also monitoring the data being written is what is expected in terms of volume and distinct values.

3. How familiar are you with cloud infrastructure like GCP/ AWS?

I did not have experience, but I shared that I know the basic concepts of compute, storage, database, gateways, etc.

Also added that I have experience using AWS and GCP in personal projects dealing with web scrapers.

But I did not have a strong project to share, hence I did not elaborate.

4. How did you learn these cloud data processing technologies?

I shared about the mentorship in company, and also that I learn them by personal projects. He seemed disappointed that I did not have any certifications.

I think this is an either/or kind of thing. Perhaps projects would be more impressive than having certs.

5. Any questions?

I asked about the tech stack that they use and interviewer shared that they use AWS's infrastructure (IaaS) like EC2 instances, but run their own in-house data architecture on it. 

Their data pipelines are built in Java language on the Apache ecosystem.

Data warehouse is combination of Hadoop HDFS, Hive, Spark. Theyu use Kafka Streams for real-time data and Spark for batch data. 

From here I knew it was going to be tough since their in-house system is so different from what I have learned so far. 

He did ask me about my undergrad backgroun in Chemical Engineering and commended that I had the ability to learn very quickly - I think I'll leverage on this point in the future.

## Conclusions

1. Need to know some form of cloud infrastructure for DE. This is best exemplified in a **strong, technical, ETL + frontend project** that is completely **my own idea**. Share this with interviewer and can discuss the considerations in building it.

2. More often than not, companies are looking for people who already have experience in the tech stack. The current zeitgeist for DE is **Apache Hadoop/Spark** and some form of **cloud deployment**.

3. DE roles can vary widely from a basic SQL analyst to a cloud architect.

## To do/ Resources

- [Datastax DS220 course](https://www.datastax.com/academy) covers the concepts of Data Modelling (in the context of Cassandra)

- [YT Video](https://youtu.be/i7twT3x5yv8?si=ps-lbTGn1R1O6If_) covering System Design interview guide

Other stuff I searched up online:

- Start Data Engineering [Best Practices](https://www.startdataengineering.com/post/de_best_practices/) & [Project Template](https://www.startdataengineering.com/post/data-engineering-projects-with-free-template/)

- [DE Best Practices](https://github.com/josephmachado/data_engineering_best_practices)

- [Data Engineering Practice Problems](https://github.com/danielbeach/data-engineering-practice)

- Sample DE project from [Reddit](https://www.reddit.com/r/dataengineering/comments/xyxpku/built_and_automated_a_complete_endtoend_elt/)

- Basic questions that are dealbreakers:

    How do you define structured / unstructured data?

    Structured: well-formatted, predefined schema, organized into tables with rows and columns.
    <br>
    <br>
    Unstructured: lacks predefined data model, often text-heavy or multimedia-rich. Require further processing/ parsing to be useful for analysis.

    What makes a database relational?

    - Adherence to pre-defined schemas and how each table relates to another. Non-relational databases are flexible or may not relate tables to each other.

    What is OOP?

    - Organise code into objects to allow modular and reusable pieces of code by function.

    What is distributed processing?

    - When a task is divided into subtasks that can be processed in parallel across multiple interconnected computers or nodes. Benefits of fault-tolerance, efficient resource usage, and speed.

    How do you describe denormalised data?

    - Redundant data is repeated across different tables for the sake of faster querying (no need so many joins), at the cost of space and consistency. E.g. Both `customer_name` and `customer_id` appear in the orders table, rather than just `customer_id`. So `orders` can be queried directly without having to `JOIN` on the `customers` table