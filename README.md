<h1 align="center">Atlans</h1>

<p align="center">
  <strong>Build maps and spatial analyses by connecting blocks, not by writing scripts.</strong>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/License-AGPL--3.0--only-blue.svg" alt="License: AGPL-3.0-only"/>
  <img src="https://img.shields.io/badge/self--hosted-yes-2ea44f" alt="Self-hosted"/>
  <img src="https://img.shields.io/badge/built%20with-PostGIS-4169E1?logo=postgresql&logoColor=white" alt="Built with PostGIS"/>
</p>

---

## What is Atlans?

Atlans is a visual studio for geospatial work. You draw your analysis as a flow of blocks on a canvas: read a layer, clip it to an area, compute what you need, and publish the result as a map, a file or an email. Then Atlans runs it for you, as often as you like.

It is made for people who work with maps every day and would rather spend their time on the question than on the plumbing: analysts, researchers, public agencies, environmental teams and anyone who keeps repeating the same GIS steps by hand.

## What you can do with it

- 🧩 **Build analyses visually.** Drag blocks onto a canvas and connect them. There are more than 60 ready to use, from reading Shapefiles, GeoJSON and WFS services to buffers, spatial joins and dissolves.
- ⏰ **Put routines on autopilot.** Run a flow every morning, every hour, or when a file arrives or a webhook calls.
- 🗺️ **Share results as maps.** Publish layers to a map portal and send a public link to anyone.
- 👥 **Work as a team.** Organize work in shared workspaces, each person with the right level of access.
- 📁 **Keep files in one place.** A built-in drive stores your inputs and outputs next to the flows that use them.
- 📈 **See what happened.** Follow each run live, block by block, and look back at the history whenever you need.
- 🤖 **Work with your AI assistant.** Atlans speaks MCP, so assistants such as Claude can find data, build flows and run them for you ([docs/mcp.md](docs/mcp.md)).
- 🔐 **Keep data where it belongs.** The heavy lifting happens on executors you run on your own machines, and credentials stay encrypted end to end.

## How it works

1. **You design** a flow in the web editor.
2. **Atlans schedules** it and hands each run to an executor, a small worker you install on a server or desktop.
3. **The executor runs** the blocks and streams progress back, so you watch every step light up on the canvas.
4. **You get the result** as a map, a file in the drive, a table in your database or a message in your inbox.

Want the full picture? [docs/architecture.md](docs/architecture.md) explains every piece.

## Try it on your computer

You need **Docker** (with Compose v2.17 or newer), **make**, **openssl** and a **PostgreSQL database with PostGIS** that Atlans can reach. Atlans does not start a database of its own, so point it at one you already have.

```bash
git clone https://github.com/jlanio/atlans-studio.git atlans
cd atlans

make bootstrap      # creates .env and strong secrets for you
# Open .env and set DATABASE_URL to your Postgres (with the postgis and uuid-ossp extensions).
# If Postgres runs on this same machine, use host.docker.internal instead of localhost.

make up-dev         # starts Atlans
docker compose exec api alembic upgrade head                 # prepares the database
docker compose exec api python -m app.cli create-admin       # creates your first user
```

Then open **http://localhost:3000** and sign in. 🎉

Going to a real server, with HTTPS and your own domain? Follow the step-by-step guide in [docs/self-hosting.md](docs/self-hosting.md).

## Learn more

| If you want to… | Read |
|---|---|
| Install Atlans on your own server | [docs/self-hosting.md](docs/self-hosting.md) |
| Keep it running: upgrades, backups, troubleshooting | [docs/operations.md](docs/operations.md) |
| Understand how the pieces fit together | [docs/architecture.md](docs/architecture.md) |
| Look up settings, services, API routes and commands | [docs/reference.md](docs/reference.md) |
| Connect an AI assistant | [docs/mcp.md](docs/mcp.md) |
| Add data sources from the catalog | [docs/sources.md](docs/sources.md) |
| Create your own blocks | [docs/creating-nodes.md](docs/creating-nodes.md) |
| Set up executors and certificates | [docs/mtls-bootstrap.md](docs/mtls-bootstrap.md) |

## Under the hood

For the curious: the API is written in Python with FastAPI, the editor in Next.js and React Flow, data lives in PostgreSQL + PostGIS, files in MinIO (S3-compatible), and Valkey (Redis-compatible) carries live events. Executors connect over WebSocket with mutual TLS, and every job they receive is encrypted and signed. The details are in the [technical reference](docs/reference.md).

## Contributing

Ideas, bug reports and pull requests are welcome. Start with [CONTRIBUTING.md](CONTRIBUTING.md) and the contributor agreement, [CLA.md](CLA.md). Found a security issue? Please report it privately as described in [SECURITY.md](SECURITY.md).

## License

Copyright (C) 2026 Joselanio Ferreira de Morais.

Atlans is free software, under the GNU Affero General Public License, version 3 (AGPL-3.0-only): the text is in [LICENSE](LICENSE). You may use, study, modify and redistribute the code under those terms; anyone who offers a modified version over a network must offer its source code to the people who use it (section 13).

- The name and the logos: [TRADEMARKS.md](TRADEMARKS.md).
- Third-party components and their licenses: [THIRD-PARTY-NOTICES.md](THIRD-PARTY-NOTICES.md).

---

<p align="center">
  <sub>Made with FastAPI, Next.js, PostGIS and a lot of GIS. 🌎</sub>
</p>
