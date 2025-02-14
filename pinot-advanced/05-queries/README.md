# Run  Pinot Queries

This folder contains the code for creating an Apache Pinot Tables, and runs some queries.

## The Exercises

The exercise includes:

- Create Schema
- Create Table
- Ingest Data
- Run Queries

## Run the docker compose file

```bash
docker-compose up
```

## Launching the UI

Once that's run, you can navigate the Pinot UI - [http://localhost:9000](http://localhost:9000)

## Exercise 1 - Load data

- Next, we will populate some data in the tables
- Run the following script to ingest data from rawdata

```bash
cd data
wget https://data.gharchive.org/2021-07-21-9.json.gz
gunzip *.json.gz
/opt/pinot/bin/pinot-admin.sh LaunchDataIngestionJob -jobSpecFile /scripts/job-spec.yaml
```

- Use the UI to validate that the table was created and data was added.

## Exercise 2 - Run Queries

We will be using the API to create these tables with indexes.

- Navigate to the following URL to access the Query Console: [http://localhost:9000/#/query](http://localhost:9000/#/query)
- Run the following queries:

```SQL
-- Return count of multivalue data as a long
SELECT count(*), countMV(commit_author_names), countMV(label_ids)
FROM github_events;
-- Return min of multivalue data
SELECT minMV(label_ids), maxMV(label_ids), sumMV(label_ids), avgMV(label_ids), minMaxRangeMV(label_ids)
FROM github_events;
-- Return percentile of multivalue data
SELECT percentileMV(label_ids, 95), percentileEstMV(label_ids, 95), percentileTDigestMV(label_ids, 95)
FROM github_events;
-- distinct count for simple and multivalue data
SELECT distinctCount(created_at),
       distinctCountBitmap(created_at),
       distinctCountMV(label_ids),
       distinctCountBitmapMV(label_ids),
       distinctCountMV(commit_author_names),
       distinctCountBitmapMV(commit_author_names)
FROM github_events;
-- Returns HyperLogLog response serialized as string
SELECT distinctCountRawHLLMV(commit_author_names)
FROM github_events;

-- Aggregation w/ filters, groups and orders, mainly on MV columns
SELECT commit_author_names, count(*)
FROM github_events
WHERE commit_author_names = 'Hiro Hamada'
GROUP BY commit_author_names
ORDER BY count(*) DESC LIMIT 5;

SELECT label_ids, count(*)
FROM github_events
WHERE label_ids > 0
GROUP BY label_ids
ORDER BY label_ids, count(*) LIMIT 5;

SELECT arrayLength(label_ids) as size, count(*)
FROM github_events
WHERE label_ids > 0
GROUP BY size
ORDER BY size DESC, count (*) LIMIT 5;

SELECT repo_name, label_ids, count(*)
FROM github_events
WHERE arrayMin(label_ids) > 1000000000
GROUP BY repo_name, label_ids
ORDER BY label_ids DESC LIMIT 5;

SELECT repo_name, label_ids, count(*)
FROM github_events
WHERE label_ids > 1000000000
GROUP BY repo_name, label_ids
ORDER BY label_ids ASC LIMIT 5;

-- Use groovy function in query
SELECT groovy('{"returnType":"STRING","isSingleValue":true}', 'def x = 0; arg0.eachWithIndex{item, idx -> if (item.startsWith("Y")) { x = item }}; return x', commit_author_names) AS teammate,
       commit_author_names
FROM github_events
WHERE commit_author_names = 'Gun1Yun'
ORDER BY teammate;

-- use IN clause
SELECT repo_name, valueIn(label_ids, 199293022, 204137300, 3171280082) AS id, count(*)
FROM github_events
WHERE label_ids IN (199293022, 204137300, 3171280082)
  and arraylength(label_ids) > 1
GROUP BY repo_name, id
ORDER BY count(*) ASC LIMIT 5;

-- Selection w/ filters using JSON index and transformations on JSON and String columns
SELECT jsonExtractKey(actor_json, '$[*]'), jsonExtractKey(payload_pull_request, '$[*]'), arrayLength(jsonExtractKey(payload_pull_request, '$[*]')) AS keyCnt
FROM github_events
WHERE payload_pull_request != 'null'
ORDER BY keyCnt LIMIT 5;

SELECT jsonExtractScalar(actor_json, '$.id', 'LONG'), jsonExtractScalar(payload_commits, '$[*].author.name', 'STRING'), jsonExtractScalar(payload_commits, '$[*].author.name', 'STRING_ARRAY')
FROM github_events
WHERE payload_commits != 'null'
ORDER BY created_at LIMIT 5;

SELECT actor_json,
       repo_name,
       length(repo_name),
       upper(repo_name),
       lower(repo_name),
       reverse(repo_name),
       substr(repo_name, 1, 3),
       concat(repo_name, type, '_')
FROM github_events
WHERE json_match(actor_json, '"$.display_login"=''maria-canals'' AND "$.id"=73915791')
ORDER BY created_at LIMIT 5;

SELECT type, strpos(type, 'Event', 1), replace(type, 'Push', 'Pull'), rpad(type, 20, '_'), lpad(type, 20, '_'), codepoint(type), chr(codepoint(type))
FROM github_events
WHERE json_match(payload_commits, '"$[*].distinct"=''true''')
  AND startswith(type, 'Push') = 'true'
ORDER BY created_at LIMIT 5;

SELECT id, type, repo_name
FROM github_events
WHERE json_match(payload_commits, '"$[*].distinct" IS NULL')
ORDER BY created_at LIMIT 5;

-- This predicate "$[0].distinct"=''true'' AND "$[1].distinct"=''true'' does not work due to how JSON index is built
SELECT jsonExtractScalar(payload_commits, '$[*].distinct', 'STRING_ARRAY')
FROM github_events
WHERE json_match(payload_commits, '"$[0].distinct"=''true''')
  AND json_match(payload_commits, '"$[1].distinct"=''true''')
ORDER BY created_at LIMIT 5;

-- Selection w/ filters using TEXT index
SELECT id, type, repo_name
FROM github_events
WHERE text_match(payload_pull_request, '"return a zero exit code even when"')
ORDER BY created_at LIMIT 5;

SELECT id, type, repo_name
FROM github_events
WHERE text_match(payload_pull_request, 'Kafka AND "Apache License" AND "ENVIRONMENT VARIABLES"')
ORDER BY created_at LIMIT 5;

SELECT id, type, repo_name
FROM github_events
WHERE created_at > '2021-07-21T08:00:00Z'
  AND text_match(payload_pull_request, 'react*')
ORDER BY created_at LIMIT 5;

SELECT id, type, repo_name
FROM github_events
WHERE text_match(payload_pull_request, '/.*ImageInput/')
ORDER BY created_at LIMIT 5;

-- FST based regexp_like works differently with non-FST based one.
SELECT id, type, repo_name, repo_name_fst
FROM github_events
WHERE regexp_like(repo_name_fst, '.*Eat')
ORDER BY repo_name LIMIT 5

SELECT id, type, repo_name, repo_name_fst
FROM github_events
WHERE regexp_like(repo_name, '.*Eat$')
ORDER BY repo_name LIMIT 5

-- Explain Plan

-- Single stage Query
EXPLAIN PLAN FOR SELECT repo_name, count(repo_name) FROM github_events GROUP BY repo_name

-- Multistage Query
EXPLAIN PLAN FOR SELECT distinctCount(created_at), distinctCountBitmap(created_at), distinctCountMV(label_ids), distinctCountBitmapMV(label_ids), distinctCountMV(commit_author_names), distinctCountBitmapMV(commit_author_names) FROM github_events

```

- Navigate to the Tables by going here: [http://localhost:9000/#/tables](http://localhost:9000/#/tables)

## Validate deployment

Make sure:

- All Queries run and are performant

## Success

There! 
You've just run several queries on Pinot!
