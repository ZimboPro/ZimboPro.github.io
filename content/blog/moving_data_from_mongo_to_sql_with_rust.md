+++
title = "Moving data from MongoDB to SQL with Rust"
description = "My journey of moving data from MongoDB to SQLite and Postgres using Rust"
date = 2025-02-08
[taxonomies]
tags=["Rust", "Database", "SQL", "MongoDB"]
+++

## Intro

In 2024, I started helping a friend to manage her website and server. However, the database being used was MongoDB and the tooling that she wanted to use such as PowerBI prefers SQL databases and not Mongo. There are tools that allow it to work but she preferred to have the database converted to SQL so that she would have access to a larger amount of tools and ecosystem.  

So I agreed to help her and this blog is to share what I learnt and the roadblocks I encountered.

## Available tools

Like most software engineers, I first checked to see if there are existing tools or repos that will be able to do this conversion. Why reinvent the wheel if it already exists?

I did manage to find tools such as [Airbyte](https://airbyte.com/) and [Estuary](https://estuary.dev/). However, there was always something causing these tools to fail. For example, the one required that there be 3 instances of the database. I suppose so that it doesn't hog all the resources, however, there was only 1 instance available. I am no expert with MongoDB and I did not manage to increase the number of instances.

So those tools were unfortunately not applicable to my scenario.

I checked Github repos after that but did not find any that were promising.

## Manual method it is

So all that is left is the manual method, or rather a custom tool. Fortunately, with this conversion I did have access to the server code which was written in NodeJS and used ExpressJS and the mongoose package. So fortunately I was able to get the schema. However, as one might know, the schema for MongoDB is by no means strictly enforced. Or at least not in the server code.

So I did what any sane developer would do, I wrote some code to get all the _fields_ for each _table_ in MongoDB. Well, I managed to get the list of _tables_ as well as the _fields_ (there might be extra steps or tooling but I cannot remember exactly).

## Getting the Data

Since I have gotten all the metadata, I can use the schemas and the fields to create structs in Rust and then extract the data right? Right?

Um, not quite.

So when I wrote the custom tool to get the data directly from MongoDB with the official MongoDB Rust client I failed. It could be my lack of knowledge about MongoDB and the client but I was not able to get the data at all.

So what now?

## Multi-step process

So is there a method to export the data from MongoDB? Yes, there is [mongodump](https://www.mongodb.com/docs/database-tools/mongodump/) and it did exactly what I wanted. It gave me the following data:

```
-- data
  -- dump
    -- db-name
      -- table.bson
      -- table-1.bson
      -- table-2.bson
```
So lets read those files with Rust.

...

Um, at least when I wrote the tool there was no crate available.

So convert to JSON?

Thank goodness, there is [bsondump](https://www.mongodb.com/docs/database-tools/bsondump/) which does exactly that. So a quick script later and I know have JSON. Great.

```
-- data
  -- dump
    -- db-name
      -- table.bson
      -- table.json <-
      -- table-1.bson
      -- table-1.json <-
      -- table-2.bson
      -- table-2.json <-
```

## Extracting the data

So just using Serde-Json will allow to get the data. Easy-peasy lemon-squeezy. At least it should have been.

Firstly, the JSON files where structured weirdly.

```JSON
{"_id":{"$oid":....}
{"_id":{"$oid":....}
....
{"_id":{"$oid":....}
{"_id":{"$oid":....}
```

I had expected the files to be JSON arrays but that was not the case. So before deserializing the data, I first had to read the file and split it into lines. After that I was able to deserialize.

One bug I had though was that some files would be empty. Turns out, even if there was no data in the table, a file would still be created. So just adding a simple check solved it.

### Data structure

So remember earlier I mentioned that the schema is not really enforced? So that came back to bite me. I had some really unexpected data conversion issues but at the same time it is understandable since MongoDB accepts anything.

The worst one, was the profile table that had an array of images which are all URL links. No problem, that was by design. But what I did not expect was the actual data.

```JSON
{
  ...
  "images": [
    "https://url.to/some/image.jpg",
    null,
    false
  ],
  ...
}
```

That was unexpected. So now I had to tweak the struct to be able to parse the JSON.

``` rust
pub struct Profile {
  ...
  pub images: Vec<Option<Bson>>,
  pub value: Option<Bson>,
  ...
}
```

Some other gotchas was numbers. I also had to parse them as `Bson` objects as can be seen above, before converting into into the appropriate number format. But that was because they are stored a bit differently when converted from Bson files to JSON.

```JSON
{
  ...
  "value":{"$numberInt":"4500"},
  ...
}
```

Fortunately, dates was possible to convert directly with the Bson crate.

```rust
pub struct Profile {
  ...
  #[serde(rename = "updatedAt")]
  #[serde(with = "bson::serde_helpers::chrono_datetime_as_bson_datetime")]
  pub updated_at: chrono::DateTime<chrono::Utc>,
  ...
}
```

## IDs

This requires its own section. In MongoDB, ObjectID's are used and not UUIDs that are often used in SQL databases. And unfortunately no _official_ way to convert between them. This was quite the headache as I wanted to ensure the links and relations between tables was maintained in the conversion and the easiest way would be to directly converted the IDs.

Some Googling later, I managed to find (after several hours, possibly even days) a [simple](https://github.com/eadbox/bson-objectid-to-uuid) way (after I tried to create a custom function). It is written in Ruby so had to tweak it for Rust. ObjectId's are 24 characters long and by understanding the UUID structure you can do the following.

```rust
pub fn convert_bson_id_to_uuid(bson_id: &str) -> uuid::Uuid {
  let s = format!(
    "{}{}-{}-{}{}-{}{}-{}",
    UUID_PREFIX,
    &bson_id[0..2], // Prefix + first 2 oid digits
    &bson_id[2..6], // Next 4 oid digits
    UUID_VERSION,
    &bson_id[6..9], // Version (0x4) + next 3 oid digits
    UUID_VARIANT,
    &bson_id[9..12],  // Variant (0b101) + next 3 oid digits
    &bson_id[12..24]  // Last 12 oid digits
  );
  uuid::Uuid::parse_str(&s).unwrap()
}

const UUID_PREFIX: &str = "0bdaea";
const UUID_VERSION: &str = "4";
const UUID_VARIANT: &str = "a";
```

## Inserting into a SQL database

To make life simpler for myself, I used an ORM to create the database and insert the values. I originally was going to use Diesel but I was simultaneously rewriting the server in Rust using [Loco](https://loco.rs/) which uses SeaORM. So I used that instead. Fortunately I started planning what the database would like and had generated the structs from when I created the database. So I was able to reuse those structs after a copy-paste.

While planning the database, I had to split the images into a separate table. SeaORM supports SQLite and Postgres. While Postgres does support arrays, SQLite does not. I could have just joined the array into a single string and separated the values with commas but I decided to do it properly.

So in the custom tool I had to map the MongoDB structure to the SQL structure while also cleaning up the data and converting the values to the required format. Thank goodness for the *From* trait, it did make life easier.

## Conclusion

So what did I learn? This conversion took me much longer than I anticipated since I was doing this after hours and amongst all my other responsibilities. This helped me to understand MongoDB a bit better.

However, the biggest take away for me was why I don't plan on ever using MongoDB (or DynamoDB since they are similar) in the future unless the client demands it. I prefer statically typed code (hence why I love Rust), as well as strongly typed datasources. So, MongoDB unfortunately is not for me.

I do see the appeal but if not managed properly it can quickly become a nightmare.

## Note

I am writing this blog several months after doing the conversion so I either forgot some of the steps I took or no longer have the code when trying a few things. However, this blog article should give some direction to anyone in the future if they have to convert MongoDB to SQL.
