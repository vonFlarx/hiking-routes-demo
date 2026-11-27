## [candidate] (major bump)

Breaking changes: API field rename and introduction of required API token for write operations and application startup.

### Breaking

- The 'distance_km' field in the Route API has been renamed to 'distance'. (C1)
- POST /routes/ with 'distance_km' in the request body returns a 400 Bad Request. (C2)
- POST requests to /routes/ and /routes/<id>/ratings/ now require a valid 'X-API-Token' header, otherwise they return 401 Unauthorized. (C4)

### Database

- Database migration 0003 renames the 'distance_km' column to 'distance' in the route table. (C5)

### Configuration

- The application now requires the 'API_TOKEN' environment variable to start. (C3)
