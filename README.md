# OS Lab 6: Multi-Container Application Deployment with Docker

## Project Report

### Objective
Successfully containerize a 3-tier web application (MySQL, Node.js Backend, React Frontend) using Docker and Docker Compose.

---

## Project Structure

```
docker-lab/
├── database/
│   ├── init.sql          # MySQL initialization script
│   └── Dockerfile        # MySQL container definition
├── backend/
│   ├── server.js         # Node.js API server
│   ├── package.json      # Node.js dependencies
│   └── Dockerfile        # Backend container definition
├── frontend/
│   ├── src/
│   │   ├── main.jsx      # React entry point
│   │   ├── App.jsx       # Main React component
│   │   ├── App.css       # Application styles
│   │   └── index.css     # Global styles
│   ├── index.html        # HTML template
│   ├── package.json      # React dependencies
│   ├── vite.config.js    # Vite configuration
│   └── Dockerfile        # Frontend container definition
├── downloads/            # Volume for CSV exports (created at runtime)
├── docker-compose.yml    # Docker Compose orchestration
├── .gitignore            # Git ignore file
└── REPORT.md             # This file
```

---

## Task Completion Summary

### Task 1: Database Containerization ✓
- **Status:** Complete
- **Image:** `mysql:8.0`
- **Key Features:**
  - Automatic schema initialization via `/docker-entrypoint-initdb.d/init.sql`
  - Creates `user_db` database with `users` table
  - Default admin user inserted
  - Port 3306 exposed
  - Data persistence via `db_volume`
  - Health check configured

**Dockerfile Location:** `database/Dockerfile`

### Task 2: Backend Containerization ✓
- **Status:** Complete
- **Base Image:** `node:18-alpine`
- **Key Features:**
  - Listens on port 3000
  - Connects to MySQL via environment variables:
    - `DB_HOST=database`
    - `DB_USER=root`
    - `DB_PASSWORD=root`
    - `DB_NAME=user_db`
  - CSV export functionality
  - Data directory mounted to host (`./downloads`)
  - Health check configured
  - Automatic restart on failure

**Dockerfile Location:** `backend/Dockerfile`

**API Endpoints:**
- `POST /users` - Create new user
- `GET /users` - Fetch all users
- `GET /save-csv` - Export users to CSV

### Task 3: Frontend Containerization ✓
- **Status:** Complete
- **Base Image:** `node:18-alpine`
- **Key Features:**
  - React + Vite development server
  - Vite configured to bind to `0.0.0.0` (accessible externally)
  - Port 5173 exposed
  - Full UI with add user form and export functionality
  - Error handling and status messages

**Dockerfile Location:** `frontend/Dockerfile`

---

## Docker Compose Configuration

### Bonus Task: Docker Compose & Volumes ✓

**File Location:** `docker-compose.yml`

#### Networking
- Custom bridge network `oslab_network` connects all services
- Services communicate via service names (e.g., `database`, `backend`)

#### Volumes
1. **Database Persistence:**
   - Named volume `db_volume` mounted at `/var/lib/mysql`
   - Survives container deletion

2. **CSV Export Bind Mount:**
   - Host path: `./downloads`
   - Container path: `/app/data`
   - Files exported from the app appear on the host machine

#### Service Dependencies
- Backend depends on Database (waits for health check)
- Frontend depends on Backend
- Health checks ensure services are ready before dependent services start

#### Environment Variables
- Automatically configured for each service
- Database credentials passed to backend at runtime

---

## How to Build and Run

### Prerequisites
- Docker installed and running
- Docker Compose installed
- Port 3306, 3000, 5173 available on host machine

### Building Individual Images

```bash
# Database
docker build -t <dockerhub-username>/oslab-database:latest ./database

# Backend
docker build -t <dockerhub-username>/oslab-backend:latest ./backend

# Frontend
docker build -t <dockerhub-username>/oslab-frontend:latest ./frontend
```

### Running with Docker Compose (Recommended)

```bash
# Start all services
docker compose up

# Stop all services
docker compose down

# View logs
docker compose logs -f

# Remove volumes (WARNING: deletes database data)
docker compose down -v
```

### Running Individual Containers

```bash
# Database
docker run -d --name oslab-mysql \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=user_db \
  -p 3306:3306 \
  -v db_volume:/var/lib/mysql \
  <dockerhub-username>/oslab-database:latest

# Backend
docker run -d --name oslab-backend \
  -e DB_HOST=oslab-mysql \
  -e DB_USER=root \
  -e DB_PASSWORD=root \
  -e DB_NAME=user_db \
  -p 3000:3000 \
  -v ./downloads:/app/data \
  --link oslab-mysql \
  <dockerhub-username>/oslab-backend:latest

# Frontend
docker run -d --name oslab-frontend \
  -p 5173:5173 \
  <dockerhub-username>/oslab-frontend:latest
```

---

## Accessing the Application

Once all containers are running:

1. **Frontend:** http://localhost:5173
2. **Backend API:** http://localhost:3000
3. **Database:** localhost:3306

### Testing Features

1. **Add User:**
   - Fill in Email, First Name, Last Name
   - Click "Add User"
   - User appears in the list

2. **View Users:**
   - Users are displayed in the list
   - Initial admin user is pre-populated

3. **Export to CSV:**
   - Click "Save Users to CSV"
   - File saved to `./downloads/users_dump.csv` on host machine
   - Can be viewed directly without entering the container

4. **Refresh:**
   - Click "Refresh" to fetch latest users

---

## Docker Hub Image Deployment

### Pushing Images to Docker Hub

```bash
# Login to Docker Hub
docker login

# Tag images with your Docker Hub username
docker tag oslab-database:latest <dockerhub-username>/oslab-database:latest
docker tag oslab-backend:latest <dockerhub-username>/oslab-backend:latest
docker tag oslab-frontend:latest <dockerhub-username>/oslab-frontend:latest

# Push to Docker Hub
docker push <dockerhub-username>/oslab-database:latest
docker push <dockerhub-username>/oslab-backend:latest
docker push <dockerhub-username>/oslab-frontend:latest
```

### Image Links (Update with your credentials)
- **Database:** `https://hub.docker.com/r/<dockerhub-username>/oslab-database`
- **Backend:** `https://hub.docker.com/r/<dockerhub-username>/oslab-backend`
- **Frontend:** `https://hub.docker.com/r/<dockerhub-username>/oslab-frontend`

---

## Key Concepts Demonstrated

### 1. Process Isolation
- Each service runs in its own container with isolated filesystem
- Dependencies managed within container (node_modules, MySQL runtime)
- No host system pollution

### 2. Networking
- Custom bridge network enables inter-container communication
- Services discover each other via DNS (service names)
- No port exposure between containers, only to host

### 3. Volume Management
- **Named Volume:** Database persistence across container restarts
- **Bind Mount:** Enables host machine access to exported CSV files
- Data survives container lifecycle

### 4. Orchestration
- Docker Compose manages 3 services as single unit
- Health checks ensure proper startup order
- Dependencies configured for automatic restart on failure

### 5. Environment Configuration
- Services configured via environment variables
- Database credentials never hardcoded
- Easy to change configuration without rebuilding images

---

## Technical Details

### API Integration
- Frontend uses `http://localhost:3000` for backend API calls
- CORS enabled on backend for cross-origin requests
- Automatic user list refresh after adding new users

### Database Connection
- Backend uses `mysql2` connection pool for efficiency
- Automatic connection retries on failure
- Support for multiple concurrent connections

### CSV Export
- Uses `csv-writer` library for standard CSV format
- Files persisted to host machine via bind mount
- Includes all user fields: ID, Email, First Name, Last Name

### Frontend UI
- Responsive design with basic styling
- Input validation for email field
- Status messages for user feedback
- Loading state during API calls
- Refresh button to manually sync with backend

---

## Troubleshooting

### Container fails to start
- Check Docker logs: `docker compose logs <service-name>`
- Verify ports are not already in use
- Ensure sufficient disk space

### Backend cannot connect to database
- Verify database container is healthy: `docker compose ps`
- Check database logs: `docker compose logs database`
- Ensure network connectivity with: `docker network inspect oslab_network`

### Frontend cannot reach backend
- Verify backend is running: `curl http://localhost:3000/users`
- Check CORS is enabled on backend (it is in provided code)
- Verify API_URL in frontend code matches service port

### CSV file not appearing in downloads folder
- Verify bind mount is correctly configured in docker-compose.yml
- Check backend logs for errors: `docker compose logs backend`
- Ensure `/app/data` directory exists in backend container (created automatically)

---

## Summary

This project successfully demonstrates:
✓ Containerization of a complete 3-tier application
✓ Multi-container orchestration with Docker Compose
✓ Volume management for persistence and host access
✓ Networking between isolated containers
✓ Environment-based configuration
✓ Health checks and dependency management
✓ Production-ready patterns (Alpine images, health checks, restart policies)

All mandatory tasks completed. Bonus Docker Compose orchestration implemented with full features.

---

**Submission Date:** December 6, 2025
**Docker Compose Version:** 3.9
**Images Used:** mysql:8.0, node:18-alpine

