+++
title = 'Typesense Reference'
date = 2023-09-01T10:28:45-07:00
draft = false
+++

Whenever I need some sort of search engine, I often reach for Typesense over ElasticSearch due to its simplicity and ease of use as long as the dataset can fit into memory. This is just a quick Typesense reference sheet for me so I'm writing this as if no one else is reading it.

## Installation

See: [https://typesense.org/docs/guide/install-typesense.html](https://typesense.org/docs/guide/install-typesense.html)

**Linux Binary**

```
# x64
curl -O https://dl.typesense.org/releases/0.25.0/typesense-server-0.25.0-linux-amd64.tar.gz
tar -xzf typesense-server-0.25.0-linux-amd64.tar.gz

# arm64
curl -O https://dl.typesense.org/releases/0.25.0/typesense-server-0.25.0-linux-arm64.tar.gz
tar -xzf typesense-server-0.25.0-linux-arm64.tar.gz

# Start Typesense
export TYPESENSE_API_KEY=xyz
mkdir $(pwd)/typesense-data # Use a directory like /var/lib/typesense in production
./typesense-server --data-dir=$(pwd)/typesense-data --api-key=$TYPESENSE_API_KEY --enable-cors
```

**Docker Compose**

```
version: '3.4'
services:
  typesense:
    image: typesense/typesense:0.25.0
    restart: on-failure
    ports:
      - "8108:8108"
    volumes:
      - ./typesense-data:/data
    command: '--data-dir /data --api-key=xyz --enable-cors'
```

```
mkdir $(pwd)/typesense-data
docker-compose up
```

## Create a Typesense collection

```
curl "http://10.139.17.194:8108/collections" \
 -X POST \
 -H "Content-Type: application/json" \
 -H "X-TYPESENSE-API-KEY: UbxEjqT8gzpqrVZe2" -d '{
"name": "stories",
"fields": [
{"name": "id", "type": "string" },
{"name": "id_author", "type": "int64" },
{"name": "title", "type": "string" },
{"name": "cast", "type": "string", "optional": true },
{"name": "description", "type": "string" },
{"name": "modified_timestamp", "type": "int64" },
{"name": "tags", "type": "string[]", "optional": true }
],
"default_sorting_field": "modified_timestamp"
}'
```

## Create Typesense collection with auto type detection

```
curl "http://10.139.17.194:8108/collections" \
 -X POST \
 -H "Content-Type: application/json" \
 -H "X-TYPESENSE-API-KEY: UbxEjqT8gzpqrVZe2" -d '{
"name": "stories",
"fields": [
{"name": ".*", "type": "auto" }
]
}'
```

## Search for a story by title

```
curl -H "X-TYPESENSE-API-KEY: UbxEjqT8gzpqrVZe2" \
"http://10.139.17.194:8108/collections/stories/documents/search\
?q=test&query_by=title,tags,cast,description&include_fields=id,title"
```

## Retrieve a collection

```
curl -H "X-TYPESENSE-API-KEY: UbxEjqT8gzpqrVZe2" \
 -X GET \
 "http://10.139.17.194:8108/collections/stories"
```

## List all collections

```
curl -H "X-TYPESENSE-API-KEY: UbxEjqT8gzpqrVZe2" \
 "http://10.139.17.194:8108/collections"
```

## Delete a collection

```
curl -H "X-TYPESENSE-API-KEY: UbxEjqT8gzpqrVZe2" \
     -X DELETE \
    "http://10.139.17.194:8108/collections/stories"
```
