# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GoTrustJet is a charter aviation trust and verification platform consisting of three independent components that work together:

1. **gotrustjet** - FastAPI backend API
2. **data_ingest** - Python data scraping service
3. **gtj-client** - React frontend application

## Architecture

### System Flow

```
Data Sources (FAA, Aviation DBs)
    ↓
data_ingest (Scraper) → validates/sanitizes data
    ↓
gotrustjet API (Backend) → stores in Supabase PostgreSQL
    ↓
gtj-client (Frontend) → displays data to users
```

### Authentication Architecture

- **Frontend**: Supabase Auth handles user authentication with MFA support
- **Backend**: JWT-based authentication using Supabase tokens
  - Tokens are verified via `SupabaseJWTBearer` in [gotrustjet/src/auth/service.py](gotrustjet/src/auth/service.py)
  - All protected endpoints use the `authentication` dependency
- **Data Ingestion**: Uses email/password credentials to authenticate with backend API
  - Tokens cached with expiry tracking in `GoTrustJetClient`

### Database Schema (Supabase PostgreSQL)

The backend uses SQLAlchemy with direct PostgreSQL connection to Supabase. Key models:

- **Operators** ([gotrustjet/src/operator/models.py](gotrustjet/src/operator/models.py)): Charter operators with trust scores, regulatory status, and certificate numbers
- **Aircraft**: Fleet information
- **Organizations**: Company/organization data
- **Users**: User accounts (managed by Supabase Auth)

Connection configured in [gotrustjet/src/common/config.py](gotrustjet/src/common/config.py) using direct PostgreSQL URL (port 5432).

### Data Ingestion Pipeline

The data_ingest service follows a pipeline architecture:

1. **Scrapers** extend `BaseScraper` ([data_ingest/src/scrapers/base_scraper.py](data_ingest/src/scrapers/base_scraper.py))
   - Rate limiting per domain
   - Exponential backoff retry logic
   - Async HTTP client with httpx
2. **Orchestrator** ([data_ingest/main.py](data_ingest/main.py)) coordinates multiple scrapers
3. **Validation** happens via Pydantic models before API insertion
4. **API Client** ([data_ingest/src/api/gotrustjet_client.py](data_ingest/src/api/gotrustjet_client.py)) handles authentication and bulk upserts
   - Duplicate detection by certificate number
   - Batch processing with continue-on-error support

## Development Commands

### Backend (gotrustjet)

```bash
cd gotrustjet/src

# Setup
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Run API server
python main.py
# Or with uvicorn directly:
uvicorn src.main:app --reload --host 0.0.0.0 --port 8000
```

**Important**: The operator router is NOT currently registered in [main.py](gotrustjet/src/main.py). To enable operator endpoints, add:

```python
from src.operator.router import operator_router
app.include_router(operator_router, tags=["operators"])
```

### Data Ingestion (data_ingest)

```bash
cd data_ingest

# Setup
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Edit .env with your credentials

# Run scraper
python main.py

# Run with debug logging
LOG_LEVEL=DEBUG python main.py

# Run specific tests
python test_scraper.py
python test_real_scraper.py
python test_with_validation.py
```

### Frontend (gtj-client)

```bash
cd gtj-client

# Install dependencies
npm install

# Start development server
npm start

# Run tests
npm test

# Run tests with coverage
npm test -- --coverage

# Build for production
npm run build
```

## Critical Integration Points

### 1. Operator Router Registration

The data_ingest service expects the `/operators` endpoint to exist in the gotrustjet API. Currently the operator router exists but is not included in [gotrustjet/src/main.py:6-10](gotrustjet/src/main.py#L6-L10). This must be added for the integration to work.

### 2. Environment Variables

**gotrustjet** requires:
- `SUPABASE_URL` - Supabase project URL
- `SUPABASE_KEY` - Supabase service role key (used as PostgreSQL password)
- `SUPABASE_JWT_SECRET` - For JWT verification

**data_ingest** requires:
- `GOTRUSTJET_URL` - Backend API URL (default: http://localhost:8000)
- `GOTRUSTJET_EMAIL` - Credentials for API authentication
- `GOTRUSTJET_PASSWORD`
- Rate limiting and scraper configuration (see [data_ingest/.env.example](data_ingest/.env.example))

**gtj-client** requires:
- `REACT_APP_SUPABASE_URL` - Supabase project URL
- `REACT_APP_SUPABASE_ANON_KEY` - Supabase anonymous key

### 3. Database Connection

The backend connects directly to Supabase PostgreSQL on port 5432, bypassing the REST API. This is configured with a hardcoded connection string in [gotrustjet/src/common/config.py:17](gotrustjet/src/common/config.py#L17).

## Module Structure

### Backend API (gotrustjet/src/)

Each domain module follows this structure:
- `models.py` - SQLAlchemy ORM models
- `schemas.py` - Pydantic schemas for request/response validation
- `service.py` - Business logic
- `router.py` - FastAPI route definitions

Common utilities in `src/common/`:
- `config.py` - Database connection and settings
- `utils.py` - Supabase client initialization
- `dependencies.py` - FastAPI dependencies

### Data Scraper (data_ingest/src/)

- `scrapers/` - Scraper implementations (FAA, aviation databases, etc.)
- `api/` - GoTrustJet API client
- `utils/` - Shared utilities (logger, validator, rate_limiter, retry)
- `config.py` - Settings management using pydantic-settings

### Frontend (gtj-client/src/)

- `pages/` - Page components (SignIn, SignUp, MFA, Dashboard, EmailConfirmation)
- `lib/` - Shared libraries (Supabase client)
- `__tests__/pages/` - Component tests using React Testing Library
- `styles/` - CSS styling

## Testing Notes

### Frontend Tests
- All page components have corresponding `.test.jsx` files
- Tests use React Testing Library with jest-dom matchers
- Run `npm test` for watch mode, `npm test -- --coverage` for coverage reports
- Supabase client is mocked in [__mocks__/@supabase](gtj-client/src/__mocks__)

### Backend Tests
No formal test suite exists yet for the FastAPI backend.

### Data Ingestion Tests
- `test_scraper.py` - Tests scraper with mock data
- `test_real_scraper.py` - Tests with real FAA data
- `test_with_validation.py` - Tests validation and sanitization
- Run individual test files directly: `python test_scraper.py`

## Security Considerations

### Data Ingestion
- All scraped data is validated with Pydantic before insertion
- HTML sanitization using bleach to prevent XSS
- SQL injection prevention via SQLAlchemy ORM
- Rate limiting to respect source servers
- Certificate-based duplicate detection

### Authentication
- MFA supported on frontend
- JWT tokens with expiry
- Tokens refreshed automatically when expired (5-minute buffer)
- Bearer token authentication on all protected endpoints
