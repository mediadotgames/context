# Repository Map

Repositories in the MediaDotGames organization.

## Core Pipeline

**headlines-enrichment**

Primary enrichment pipeline. Python/uv. 7-stage pipeline: story selection → validation → clustering → labeling → PI evaluation → scope classification → refresh. Writes to RDS PostgreSQL.

**headlines-ingestion**

Pipeline responsible for inserting headline data into the database from NewsAPI.ai.

## Database

**db**

Database migrations and seed data. Contains SQL migrations for schema changes (validation_outputs, enriched_headlines, outlet_bias_scores, heatmap views).

## Documentation

**context**

Architecture and system documentation (this repo).

**docs**

Supplementary documentation (bias methodology, etc.).

## Infrastructure

**collector**

Data collection service.

**scraper**

Web scraping infrastructure used to extract article content.

**googleshoot_db_sync**

Service syncing Google Sheets data into the database.

## Frontend

**fe-api**

Backend API used by frontend interfaces.

**front**

Frontend application.

**FEsandbox**

Experimental frontend development environment.
