# Docker Codebase Instructions

## Architecture Overview

This is a learning project demonstrating Docker containerization with two distinct applications:

1. **`interactive-mode/`**: Python-based random number generator with user input
   - Simple CLI application that demonstrates Python containerization
   - Basic interactive workflow: accepts min/max inputs, validates, generates random number

2. **`nodejs-app-first-dockerfile/`**: Full-stack Express.js web application
   - Web server running on port 80 with form-based goal tracking
   - Data flow: user submits goal via POST → updates in-memory `userGoal` variable → redirects to GET for display
   - Serves static CSS from `public/` directory
   - Uses Express.js middleware: `body-parser` for form parsing, `express.static` for assets

## Key Patterns & Conventions

### Dockerfile Best Practices in Use

**Python Application** (`interactive-mode/Dockerfile`):
- Single-stage build with lightweight base image
- Works directory isolation via `WORKDIR /app`
- Executes Python script directly with `CMD`
- Interactive mode requires `docker run -it` flag to accept input

**Node.js Application** (`nodejs-app-first-dockerfile/Dockerfile`):
- **Optimized layering pattern**: `COPY package.json` → `RUN npm install` → `COPY .` separately
  - This leverages Docker layer caching—dependencies cached unless `package.json` changes
  - Application code changes don't trigger `npm install` rebuild
- Exposes port 80 implicitly (no EXPOSE needed for basic functionality)
- Node.js runtime inherited from `node:22` base image

### Application Conventions

**Node.js Server** (`server.js`):
- Stateful in-memory data storage (not persistent across container restarts)
- HTML templates embedded in route handlers (server-side rendering)
- Forms use `method="POST"` with `name` attributes for `body-parser` parsing
- Responds with `res.redirect()` for POST-Redirect-GET pattern (prevents form resubmission)

**Styling** (`public/styles.css`):
- Minimal utility approach with focus on form UI and visual hierarchy
- Relies on CSS inheritance and simple selectors

## Common Docker Commands for This Project

### Python RNG Application
```bash
# Build: interactive-mode
docker build -t rng:latest interactive-mode/

# Run interactive (accepts user input)
docker run -it rng:latest

# Run non-interactive with input piping
echo -e "1\n100" | docker run -i rng:latest
```

### Node.js Web Application
```bash
# Build: nodejs-app-first-dockerfile
docker build -t goal-tracker:latest nodejs-app-first-dockerfile/

# Run with port mapping
docker run -p 8080:80 goal-tracker:latest

# Run in background
docker run -d -p 8080:80 --name goal-app goal-tracker:latest

# View logs
docker logs goal-app

# Stop container
docker stop goal-app
```

## Debugging & Development Notes

- **Python app**: Input val/idation happens in Python (`if max_number < min_number`)—Docker exec is not needed for testing
- **Node.js app**: In-memory goals lost on container restart; to persist, modify to use file/database storage
- **Port conflicts**: Node.js expects port 80 internally; use `-p HOST_PORT:80` flag to remap
- **Static files**: Express.js serves `public/` directory—changes require container rebuild if baked into image
- **Dependencies**: Node.js dependencies cached during build; updates require rebuild (layer 2 of the Dockerfile)

## Patterns to Maintain

1. **Layered Dockerfiles**: Keep dependency installation before application code to maximize cache efficiency
2. **Port conventions**: Python app uses stdin/stdout; Node app uses HTTP port 80
3. **Single responsibility**: Each directory represents one deployable unit
4. **Explicit base images**: Both use specific version tags (Python, Node 22) for reproducibility

## Per-project instruction files

- This repository contains small sample apps. Some projects have their own focused Copilot instructions under their
   respective `.github/` directories. Read these when working in that folder:
   - `data-volumes-02-added-dockerfile/.github/copilot-instructions.md` — detailed guidance for the feedback app
      ( explains the EXDEV cross-device rename issue, shows safe `docker run` examples, and includes cleanup commands ).

When modifying or debugging a specific sample, prefer the per-project file for project-specific patterns and exact
commands — the top-level document explains the overall layout and common patterns.
f