---
title: Quickstart
description: Run Exquisitor with its user interface on a small sample collection of videos.
icon: rocket
---

This guide sets up the Live Services Engine (LSE) and the user interface (UI) on your own machine, with a small sample collection of short videos. Once you're done, you can search the collection in your browser at `https://localhost:5000`.

## What you need

- [git](https://git-scm.com/), Python 3.13 or newer, and [uv](https://docs.astral.sh/uv/) for the LSE
- [Node.js](https://nodejs.org/) for the UI
- [Caddy](https://caddyserver.com/docs/install), a web server that puts the LSE, the UI and the media behind one secure address
- About 10 GB of free disk space for the sample collection and the search model

## 1. Get the code

Clone the LSE and the UI next to each other, and install their dependencies:

```bash
git clone https://github.com/exq-hub/live-services-engine.git
git clone https://github.com/exq-hub/base-user-interface.git
cd live-services-engine && uv sync && cd ..
cd base-user-interface && npm install && cd ..
```

## 2. Get the sample collection

The sample collection holds 1,290 short videos from Meta's PE Video Dataset, with a keyframe every 4 seconds, a database and ready-made embeddings. It may only be used for non-commercial purposes, as described on its [dataset page](https://huggingface.co/datasets/phdok/exq-pvd). Download it (2.7 GB) and unzip it into the LSE's `data` folder:

```bash
cd live-services-engine
curl -L -o pvd.zip https://huggingface.co/datasets/phdok/exq-pvd/resolve/main/pvd.zip
unzip pvd.zip -d data
rm pvd.zip
```

This gives you a `data/pvd/` folder with the database, the embeddings, the keyframes and the videos.

## 3. Configure the LSE

Create `data/config.toml` in the `live-services-engine` folder with this content:

```toml
[default]
model_device = "auto"

[server]
host = "127.0.0.1"
port = 8000
reload = false

[logging]
level = "INFO"
directory = "./logs/"

[[collections]]
name = "pvd"
enabled = true
database_file = "./data/pvd/pvd.db"
thumbnail_media_url = "https://localhost:5000/pvd"
original_media_url = "https://localhost:5000/pvd"

  [[collections.indexes]]
  name = "CLIP"
  index_type = "zarr"
  embeddings_file = "./data/pvd/embeddings.zip"
```

This sets up one collection called `pvd` with one index over its keyframes. The media URLs point at Caddy, which serves the keyframes and videos from `data/pvd/`. All options are described in [Configuration](/configuration/).

## 4. Configure Caddy

Create a file called `Caddyfile` in the `live-services-engine` folder with this content:

```
localhost:5000 {
	@api path /exq*
	reverse_proxy @api localhost:8000

	@media path /pvd/*
	handle @media {
		root * ./data
		file_server
	}

	reverse_proxy localhost:3000
}
```

Caddy then answers at `https://localhost:5000`. It passes API requests to the LSE on port 8000, serves the media from the `data` folder, and passes everything else to the UI on port 3000.

## 5. Start everything

Start the LSE, the UI and Caddy, each in its own terminal:

```bash
# Terminal 1, in live-services-engine
uv run python main.py
```

```bash
# Terminal 2, in base-user-interface
npm run dev
```

```bash
# Terminal 3, in live-services-engine
caddy run
```

The first time the LSE starts, it downloads the search model (about 3.3 GB), which can take a few minutes. It is ready once it logs `LSE application initialized successfully`. The first time Caddy starts, it asks for your password to install its own certificate authority, so that your browser trusts `https://localhost:5000`.

## 6. Search

Open `https://localhost:5000` and pick the `pvd` collection. Type `a sailing boat on the sea` in the search box and press Enter. The results are keyframes, and clicking one opens the video it belongs to.

## Next steps

To learn what else you can do with the collection, read about [CLIP search](/search/clip/), [relevance feedback](/search/rf/) and [filters](/filters/). To set up a collection of your own media, see [Database](/database/) and [Vector indices](/indices/), and then [Configuration](/configuration/).
