# Olympic Shooting Stats

A project to record and display statistics for STOS (Sociedad de Tiro Olímpico de Santiago) members.

## Goals

- Record results from different members.
- Store athlete, event information in one place.
- Show statistics through tables and charts: medal counts, scores, records and trends over time.

## Disciplines covered

| Discipline | Events |
|------------|--------|
| Rifle      | 10m Air Rifle, 50m Rifle 3 Positions |
| Pistol     | 10m Air Pistol, 25m Pistol, 25m Rapid Fire Pistol |
| Shotgun    | Trap, Skeet |

Men's, women's and mixed team events are included.

## Planned features

- [ ] Data model for athletes, events, Games and results
- [ ] Import results from CSV files
- [ ] Export information to CSV files
- [ ] Each athlete can manual entry of new results
- [ ] Trophy table by athlete
- [ ] Score history and personal bests per athlete
- [ ] Charts comparing athletes, competitions, ...
- [ ] Search and filters (year, discipline, event, gender)

## Data

Each result record will contain at least:

- Games (year and host city)
- Discipline and event
- Athlete and country
- Phase (qualification / final)
- Score and final rank

## Technology stack

The project is a web application split into two parts: a **REST API written in Go** and a **web frontend** that consumes it.

### Backend (REST API)

| Area | Technology |
|------|------------|
| Language | [Go](https://go.dev/) 1.23+ |
| HTTP routing | Standard library `net/http` (`ServeMux` with method and path patterns) |
| Database | [SQLite](https://www.sqlite.org/) (single database file) |
| Database access | [modernc.org/sqlite](https://pkg.go.dev/modernc.org/sqlite) driver (pure Go, no CGO) via `database/sql` + [sqlc](https://sqlc.dev/) for type-safe queries |
| Migrations | [golang-migrate](https://github.com/golang-migrate/migrate) |
| Authentication | JWT (so each athlete can sign in and enter their own results) |
| CSV import/export | Standard library `encoding/csv` |
| Configuration | Environment variables |
| Logging | Standard library `log/slog` (structured logs) |
| API documentation | [OpenAPI 3](https://www.openapis.org/) specification |
| Testing | `go test`, `net/http/httptest` |

### Frontend (web application)

| Area | Technology |
|------|------------|
| Framework | [Vue 3](https://vuejs.org/) + [TypeScript](https://www.typescriptlang.org/) |
| Build tool | [Vite](https://vitejs.dev/) |
| Charts | [Chart.js](https://www.chartjs.org/) |
| API communication | JSON over HTTP (`fetch`) against the Go REST API |

### Tooling and deployment

- [Docker](https://www.docker.com/) and Docker Compose to run the API and the frontend locally (the SQLite file is kept in a mounted volume).
- [golangci-lint](https://golangci-lint.run/) for Go linting.
- GitHub Actions for continuous integration (build, lint and tests).

### Hosting (AWS)

The backend will be hosted on an **AWS EC2 instance**:

| Area | Technology |
|------|------------|
| Compute | [Amazon EC2](https://aws.amazon.com/ec2/) instance (Linux) running the Go API as a single binary |
| Storage | SQLite database file on the instance's [EBS](https://aws.amazon.com/ebs/) volume |
| Backups | [Litestream](https://litestream.io/) continuously replicating the SQLite file to [Amazon S3](https://aws.amazon.com/s3/) |
| HTTPS / reverse proxy | [Caddy](https://caddyserver.com/) (automatic Let's Encrypt certificates), which also serves the built frontend |
| Process management | `systemd` service for the API |
| Networking | Elastic IP and a security group allowing only HTTP/HTTPS (and SSH restricted to administrators) |
| Deployment | GitHub Actions builds the Linux binary and deploys it to the instance |

Since SQLite is a local file, the API runs as a single instance; this is enough for the expected load of the club.

### Planned project layout

```
stos-stats/
├── backend/
│   ├── cmd/api/          # API entry point (main.go)
│   ├── internal/         # handlers, services, repositories, auth
│   ├── migrations/       # SQL migrations
│   └── api/openapi.yaml  # REST API specification
├── frontend/             # Vue + TypeScript web application
└── docker-compose.yml
```

## Getting started

### Prerequisites

- Go 1.23 or later
- Node.js 20 or later (for the frontend)
- Docker and Docker Compose

_Detailed setup and run instructions will be added once the first version of the API and the frontend is available._

## Contributing

Suggestions and corrections to the data are welcome. Please open an issue or a pull request.

## License

This project is licensed under the [GNU General Public License v3.0](https://www.gnu.org/licenses/gpl-3.0.html) (GPL-3.0). See the [LICENSE](LICENSE) file for the full text.
