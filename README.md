# URL Shortener

A small ASP.NET Core 10 API for creating and managing short URLs. Data is stored in PostgreSQL using Entity Framework Core. A `GET /{code}` request redirects to the saved destination and increments its click counter.

- Swagger UI: `http://localhost:8080/swagger` (with Docker Compose)
- PostgreSQL 17 with EF Core and a checked-in migration
- Two Dockerfiles for comparison: one intentionally inefficient and one optimized
- Docker Compose starts the API and database together
- GitHub Actions validates the code, builds the optimized image, and publishes it to GitHub Container Registry (GHCR)

The sample links `/dotnet` and `/docker` are created on the first start.

## Endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/links` | Create a short link |
| `GET` | `/api/links` | List all short links |
| `GET` | `/api/links/{code}` | Get one short link |
| `DELETE` | `/api/links/{code}` | Delete one short link |
| `GET` | `/{code}` | Redirect to the destination (HTTP 302) |
| `GET` | `/healthz` | Check API and database connectivity |

## Requirements

- Docker Desktop or Docker Engine with Compose v2 (`docker compose`)
- To run the API outside Docker: .NET 10 SDK and a reachable PostgreSQL 17 instance

## Run the Full Application

Run this command from the project directory:

```powershell
docker compose up --build
```

Compose builds the API image with `Dockerfile.optimized`, starts PostgreSQL, and waits for the database health check to pass before starting the API. The services are then available at:

- Swagger UI: [http://localhost:8080/swagger](http://localhost:8080/swagger)
- API health check: [http://localhost:8080/healthz](http://localhost:8080/healthz)
- PostgreSQL: `localhost:5433` (the API uses `postgres:5432` inside the Compose network)

If host port `8080` is already in use, choose another host port without changing the container port. For example, in PowerShell:

```powershell
$env:APP_PORT = "8081"
docker compose up --build
```

With this override, open Swagger at `http://localhost:8081/swagger`. The default remains port `8080` when `APP_PORT` is not set.

The API applies any pending EF Core migrations at startup. Press `Ctrl+C` to stop Compose when running in the foreground. Alternatively, stop it from another terminal:

```powershell
docker compose down
```

The named volume `postgres_data` is preserved, so data remains available after restarting. The credentials in `compose.yml` are for the local exercise only and must not be used for a public production deployment.

## Dockerfiles

### Inefficient Variant

`Dockerfile.inefficient` uses a single .NET SDK image for both publishing and runtime. As a result, the final image also contains the SDK and build tools; the process additionally runs as `root`. This variant is intended for comparison of image size and security.

### Optimized Variant

`Dockerfile.optimized` uses a multi-stage build: the SDK is only needed in the build stage, while the final image is based on the smaller ASP.NET runtime image. The project file and NuGet configuration are copied before the rest of the source code, allowing Docker to reuse the NuGet restore layer when dependencies have not changed. The application runs in the final image with the unprivileged `$APP_UID` user ID.

Build both images locally:

```powershell
docker build -f Dockerfile.inefficient -t url-shortener:inefficient .
docker build -f Dockerfile.optimized -t url-shortener:optimized .
```

Both images contain only the API. For a complete setup with the database, use the Compose command above.

## GitHub Actions and Image Publishing

The workflow is located at `.github/workflows/publish.yml` and runs for pull requests and pushes to `main`:

1. The `validate` job restores NuGet packages, checks formatting with `dotnet format`, and creates a Release build.
2. After validation succeeds, `build-and-push` builds the image with `Dockerfile.optimized`.
3. Pull requests only build the image. Pushes publish it to GitHub Container Registry (GHCR).
4. Branch and commit SHA tags are published; pushes to the default branch also receive the `latest` tag.

There is currently no separate test project in the repository. The pipeline therefore runs formatting and build checks, but no automated application tests.

The published image is publicly available at:

```text
ghcr.io/petergrininger/url-shortener:latest
```

GitHub repository: [PeterGrininger/url-shortener](https://github.com/PeterGrininger/url-shortener)

## Project Structure

```text
url-shortener/
├── .github/workflows/publish.yml   # Validation, Docker build, and GHCR publishing
├── Contracts/LinkContracts.cs      # API request and response types
├── Data/
│   ├── Migrations/                 # EF Core database migrations
│   ├── ShortLinkDbContext.cs
│   └── ShortLinkDbContextFactory.cs
├── Endpoints/LinkEndpoints.cs      # API endpoints
├── Models/ShortLink.cs
├── Dockerfile.inefficient          # Single-stage comparison variant
├── Dockerfile.optimized            # Multi-stage production variant
├── compose.yml                     # API and PostgreSQL
├── Program.cs
└── UrlShortener.csproj
```