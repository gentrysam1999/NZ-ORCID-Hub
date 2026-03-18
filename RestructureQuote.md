# ORCID Hub Docker Restructure Quote

### Scope of Work

- **Build Validation & Finalisation:** Confirm the newly restructured Docker image builds successfully across all required instances.
- **Dependencies:** Add Shibboleth repository for RockyLinux 9.
- **Dockerfile Restructure:** Improve readability, efficiency, and reduce image size.
- **Build Validation:** Ensure the image builds successfully, resolving build errors as needed.
- **Runtime Validation:** Ensure the image runs correctly, including:
    - Connection to PostgreSQL database (DB must spin up and allow a valid connection).
    - Verification that application instances (app, worker, scheduler) can spin up and run correctly.

### Estimated Time = 24 hours

- **8 hours** – Rebuild and restructure the Dockerfile until the image builds successfully.
- **16 hours** – Debugging and fixes to ensure the image runs correctly, including app, worker, and scheduler.

**Note on Estimates:**

- The initial **8 hours** is the minimum time required for the rebuild and restructure.
- Due to the outdated and partially unstructured nature of the project, we cannot guarantee that the application will work immediately following this phase.
- The following **16 hours** is our best estimate for runtime bug fixing. While we are confident this will allow the image to build and run, we cannot guarantee full functionality within this timeframe. At the very least, this work will provide a clear picture of any remaining issues.

### Access Requirements Needed

- **Docker Hub Account:** Access is required to publish new images of the ORCID_Hub instances (app, worker, scheduler).

The Earliest time that I can see us fitting this in is mid October when Alex is back from Europe.

- Our immediate focus is on the **app** instance. Since worker and scheduler images were previously built, we expect they can be pulled directly from Docker Hub without needing a rebuild.