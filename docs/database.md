---
title: Database
description: How a collection's media, groups and metadata are stored, and what Exquisitor reads from them.
icon: database
---

Each collection has its own SQLite database file, set with `database_file` in the collection's section of `config.toml` (see [Configuration](/configuration/#%5B%5Bcollections%5D%5D)). Exquisitor only reads from it while it runs.

## How the data is organised

Exquisitor keeps what it knows about a collection in one database file. The layout follows a simplified version of the multi-dimensional media model, M³ (pronounced "emm-cube"). To work with it, you only need three ideas.

**Media.** Every item Exquisitor can show is a media. Each media has a source (its file name or path, which Exquisitor appends to the collection's media URL), an optional thumbnail, and a source type, which is one of `Image`, `Video`, `Audio`, `Text` and `Other`.

**Groups.** A media can belong to another media, which is then its group. In a video collection, for example, each keyframe belongs to the video it was taken from. Exquisitor uses groups to show the other keyframes of the same video. A filter set on a video also applies to its keyframes.

**Fields and values.** Metadata is stored as named fields, such as "Title", "Objects" or "Start (ms)". Each field has a data type, which is one of `alphanumerical` (text), `numerical_int` (whole numbers), `numerical_dec` (decimals), `date`, `time`, `timestamp` and `json`. A media can have several values for a field, or none at all. You can picture a spreadsheet with one row per media and one column per field, where a cell may hold a list or stay empty. In the database a field is called a *tagset* and a value a *tag*, which are also the names the filter endpoints use.

```
video_1.mp4  (Video)
  ├── Title: "Downtown traffic"
  └── Objects: car, person
      │
      └── image_1.jpg  (Image, in the group video_1.mp4)
            ├── CLIP Index ID: 23
            ├── Start (ms): 120000
            └── Objects: car
```

## What Exquisitor uses it for

An index knows its items only by their position in the index. When a collection is loaded, Exquisitor reads which position each media has in the index, as described in [Fields Exquisitor looks for](#fields-exquisitor-looks-for). This is how search results are turned back into media, and how the media a user marks as relevant are found in the index again.

The filter list comes from the collection's fields, and each filter's values from the values stored for that field. Every value has its own ID, which is what a filter request sends to pick values such as "car" in "Objects". For `numerical_int`, `numerical_dec`, `date`, `time` and `timestamp` fields, a filter can give a range instead. A media passes a filter if it or its group has a matching value, and only media of the searched index are returned, as described in [Filters](/filters/).

When the UI shows an item, it asks for the media's source, thumbnail, type and group, together with the values of the fields it wants to display. The related items of a group are the media that belong to it. They can be sorted by the values of a field, which is how the UI shows a video's keyframes in time order. [Temporal search](/search/temporal/) uses the time fields to place the segments of a group one after another.

## Fields Exquisitor looks for

Most fields are yours to name and fill as you like. Exquisitor recognises a few fields by name, so a collection needs them for the features that rely on them.

| Field | Type | Used for |
|-------|------|----------|
| `<index name> Index ID`, e.g. `CLIP Index ID` | `INTEGER` | The media's position in that index, starting at 0. Every media the index covers needs one. To read it from a field with another name, set the index's `tagset`, as described in [Sharing an id space](/configuration/#sharing-an-id-space). |
| `Start (ms)` | `INTEGER` | When a segment starts, in milliseconds from the start of its group. If a segment has no `Start (ms)`, Exquisitor uses `Keyframe (ms)` or else `Time (ms)`. |
| `End (ms)` | `INTEGER` | When a segment ends. Segments without one end at their start. |

Only media of the index's `source_type` are mapped, such as images for a CLIP index over keyframes or text for an index over transcripts. The time fields are needed for temporal search, and the UI also uses them to order and play a video's segments.

## Building a database

You don't need to write any SQL to build a collection's database. Describe the collection in JSON files and load them with the [Simplified M³ loader](https://github.com/Ok2610/Simplified-M3-DB). One file lists your fields and their types, one lists your media with their source, type, thumbnail and group, and one lists the values each media has. A further file can add values to fields ahead of time, but this is optional. The loader's documentation shows [the format of each file](https://github.com/Ok2610/Simplified-M3-DB/blob/main/docs/json-files.md) with examples, and [the commands](https://github.com/Ok2610/Simplified-M3-DB/blob/main/docs/commands.md) that create the database and load the files into it.

Each media an index covers needs its position in that index as a value of the index's `Index ID` field. Position 0 is the first embedding in the index's embeddings file, position 1 the second, and so on. See [Vector indices](/indices/) for the embedding files.

## Reference: tables

If you want to look inside a database file yourself, this is where each part of the model is stored. The full definitions are in the loader's [ddl.sql](https://github.com/Ok2610/Simplified-M3-DB/blob/main/m3db/ddl.sql).

| Table | What it holds |
|-------|---------------|
| `medias` | One row per media, with its `id`, `source`, `source_type` and `thumbnail_uri`. `group_id` is the `id` of the media's group, or empty if it has none. |
| `source_types` | The source types `Image`, `Video`, `Audio`, `Text` and `Other`. An index's `source_type` must be one of these names. |
| `tagsets` | One row per field, with its `id`, `name` and `tagtype_id`. A field's `id` is the filter `id` in the API. |
| `tag_types` | The data types. Each one's `description`, such as `alphanumerical` or `numerical_int`, names the table that holds its values. |
| `tags` | One row per value, with its `id`, `tagtype_id` and `tagset_id`. A value's `id` is what a filter request sends to pick it. |
| `<type>_tags`, e.g. `alphanumerical_tags` | The values of one type, with the value's `id`, the `value` itself and its `tagset_id`. Each value is stored once per field and shared by every media that has it. |
| `taggings` | Which media has which value, as pairs of `media_id` and `tag_id`. |

## Reference: code

All of Exquisitor's reads from the database go through `DatabaseRepository` in [database_repository.py](https://github.com/exq-hub/live-services-engine/blob/main/app/repositories/database_repository.py). Its docstrings describe each method and the queries behind it. The SQL that a filter is turned into is built in [db_helper.py](https://github.com/exq-hub/live-services-engine/blob/main/app/repositories/db_helper.py), whose module docstring walks through an example query.
