---
id: heroku-deployment-pipeline
alias: Heroku Deployment Pipeline
type: kit
is_base: false
version: 1
tags:
  - foundation
  - infrastructure
  - deployment
description: Complete Heroku deployment automation with buildpacks, environment configuration, add-ons, and CI/CD integration for production applications
---

# Heroku Deployment Pipeline Kit

A comprehensive kit for deploying applications to Heroku with automated pipelines, environment management, and production best practices.

## End State

After applying this kit, the application will have:

**Heroku Configuration:**
- `Procfile` defining process types (web, worker, etc.)
- `app.json` for app configuration and add-ons
- Environment variables configured per environment (staging, production)
- Buildpacks configured for application runtime
- Review apps enabled for pull request deployments

**Deployment Pipeline:**
- GitHub Actions or CI/CD integration for automated deployments
- Staging environment that auto-deploys from main branch
- Production environment with manual approval gates
- Review apps created for each pull request
- Automated database migrations on deploy
- Health check endpoints for zero-downtime deployments

**Heroku Resources:**
- Heroku Postgres database with automated backups
- Redis add-on for caching/sessions
- Log aggregation (Papertrail or Logentries)
- Monitoring add-on (New Relic or Datadog)
- SSL certificates configured
- Custom domains with DNS configuration

**Process Management:**
- Web dynos configured with appropriate scaling
- Background worker dynos for job processing
- Scheduler add-on for cron jobs
- Release phase commands for migrations

## Implementation Principles

- **12-Factor App**: Follow Heroku's 12-factor methodology
- **Environment variables**: Store all configuration in environment variables, never in code
- **Stateless processes**: Application processes should be stateless and share-nothing
- **Disposability**: Processes should start fast and shut down gracefully
- **Build/Release/Run**: Separate build, release, and run stages
- **Logs as streams**: Treat logs as event streams, not files
- **Process formation**: Scale via process formation, not server size
- **Port binding**: Applications should bind to port provided by `$PORT`
- **Concurrency**: Scale horizontally via process model
- **Dev/prod parity**: Keep development and production environments as similar as possible

## Pattern 1: Procfile Configuration

Define process types for your application:

```
# Procfile
web: node server.js
worker: node worker.js
scheduler: node scheduler.js
release: npm run migrate
```

**For different frameworks:**

**Node.js/Express:**
```
web: node server.js
worker: node worker.js
release: npm run migrate
```

**Python/Django:**
```
web: gunicorn {{PROJECT_NAME}}.wsgi --log-file -
worker: celery -A {{PROJECT_NAME}} worker --loglevel=info
release: python manage.py migrate
```

**Ruby/Rails:**
```
web: bundle exec puma -C config/puma.rb
worker: bundle exec sidekiq -C config/sidekiq.yml
release: bundle exec rails db:migrate
```

**Go:**
```
web: ./bin/{{PROJECT_NAME}}
worker: ./bin/{{PROJECT_NAME}} worker
release: ./bin/{{PROJECT_NAME}} migrate
```

## Pattern 2: app.json Configuration

Define app configuration and add-ons:

```json
{
  "name": "{{PROJECT_NAME}}",
  "description": "{{PROJECT_DESCRIPTION}}",
  "repository": "https://github.com/{{ORG}}/{{REPO}}",
  "logo": "https://{{CDN_URL}}/logo.png",
  "keywords": ["{{KEYWORD1}}", "{{KEYWORD2}}"],
  "success_url": "/health",
  "env": {
    "NODE_ENV": {
      "required": true,
      "value": "production"
    },
    "DATABASE_URL": {
      "required": true,
      "description": "PostgreSQL connection string"
    },
    "REDIS_URL": {
      "required": false,
      "description": "Redis connection string"
    },
    "SECRET_KEY": {
      "required": true,
      "generator": "secret"
    }
  },
  "formation": {
    "web": {
      "quantity": 1,
      "size": "standard-1x"
    },
    "worker": {
      "quantity": 1,
      "size": "standard-1x"
    }
  },
  "addons": [
    {
      "plan": "heroku-postgresql:standard-0",
      "as": "DATABASE"
    },
    {
      "plan": "heroku-redis:premium-0",
      "as": "REDIS"
    },
    {
      "plan": "papertrail:choklad",
      "as": "PAPERTRAIL"
    },
    {
      "plan": "scheduler:standard",
      "as": "SCHEDULER"
    }
  ],
  "buildpacks": [
    {
      "url": "heroku/nodejs"
    }
  ],
  "scripts": {
    "postdeploy": "npm run migrate && npm run seed"
  }
}
```

## Pattern 3: Environment Configuration

Set up environment-specific configuration:

**Staging Environment:**
```bash
# Set staging environment variables
heroku config:set NODE_ENV=staging --app {{APP_NAME}}-staging
heroku config:set LOG_LEVEL=debug --app {{APP_NAME}}-staging
heroku config:set API_URL=https://api-staging.example.com --app {{APP_NAME}}-staging
```

**Production Environment:**
```bash
# Set production environment variables
heroku config:set NODE_ENV=production --app {{APP_NAME}}-prod
heroku config:set LOG_LEVEL=info --app {{APP_NAME}}-prod
heroku config:set API_URL=https://api.example.com --app {{APP_NAME}}-prod
```

**Using .env files:**
```bash
# Load from .env.staging
heroku config:set $(cat .env.staging | xargs) --app {{APP_NAME}}-staging

# Load from .env.production
heroku config:set $(cat .env.production | xargs) --app {{APP_NAME}}-prod
```

## Pattern 4: GitHub Actions Deployment

Automate deployments with GitHub Actions:

```yaml
# .github/workflows/deploy.yml
name: Deploy to Heroku

on:
  push:
    branches:
      - main
      - staging
  pull_request:
    branches:
      - main

jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/staging'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Heroku Staging
        uses: akhileshns/heroku-deploy@v3.12.12
        with:
          heroku_api_key: ${{secrets.HEROKU_API_KEY}}
          heroku_app_name: "{{APP_NAME}}-staging"
          heroku_email: "{{HEROKU_EMAIL}}"
          appdir: "."
          procfile: "web: node server.js"
          healthcheck: "https://{{APP_NAME}}-staging.herokuapp.com/health"
          check_health: true
          rollback_on_healthcheck_failed: true

  deploy-production:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    needs: []
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Heroku Production
        uses: akhileshns/heroku-deploy@v3.12.12
        with:
          heroku_api_key: ${{secrets.HEROKU_API_KEY}}
          heroku_app_name: "{{APP_NAME}}-prod"
          heroku_email: "{{HEROKU_EMAIL}}"
          appdir: "."
          procfile: "web: node server.js"
          healthcheck: "https://{{APP_NAME}}.herokuapp.com/health"
          check_health: true
          rollback_on_healthcheck_failed: true
          usedocker: false
```

## Pattern 5: Review Apps Configuration

Enable review apps for pull requests:

```json
// app.json
{
  "environments": {
    "review": {
      "addons": [
        "heroku-postgresql:essential-0",
        "heroku-redis:dev"
      ],
      "scripts": {
        "postdeploy": "npm run migrate"
      },
      "env": {
        "NODE_ENV": "development",
        "REVIEW_APP": "true"
      }
    }
  }
}
```

**Enable in Heroku Dashboard:**
- Go to app settings
- Enable "Review Apps"
- Connect GitHub repository
- Configure automatic creation for pull requests

## Pattern 6: Database Migrations

Run migrations during release phase:

**Procfile:**
```
release: npm run migrate
```

**package.json scripts:**
```json
{
  "scripts": {
    "migrate": "node scripts/migrate.js",
    "migrate:rollback": "node scripts/rollback.js"
  }
}
```

**Migration script (Node.js example):**
```javascript
// scripts/migrate.js
const { execSync } = require('child_process');
const pg = require('pg');

async function migrate() {
  const client = new pg.Client({
    connectionString: process.env.DATABASE_URL,
    ssl: { rejectUnauthorized: false }
  });

  try {
    await client.connect();
    console.log('Running migrations...');
    
    // Run migration commands
    execSync('npx sequelize-cli db:migrate', { stdio: 'inherit' });
    
    console.log('Migrations completed successfully');
  } catch (error) {
    console.error('Migration failed:', error);
    process.exit(1);
  } finally {
    await client.end();
  }
}

migrate();
```

## Pattern 7: Health Check Endpoint

Implement health check for zero-downtime deployments:

**Node.js/Express:**
```javascript
// routes/health.js
app.get('/health', async (req, res) => {
  try {
    // Check database connection
    await db.query('SELECT 1');
    
    // Check Redis connection
    await redis.ping();
    
    res.status(200).json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      uptime: process.uptime()
    });
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      error: error.message
    });
  }
});
```

**Python/Django:**
```python
# views/health.py
from django.http import JsonResponse
from django.db import connection
import redis

def health_check(request):
    try:
        # Check database
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
        
        # Check Redis
        r = redis.from_url(os.environ.get('REDIS_URL'))
        r.ping()
        
        return JsonResponse({
            'status': 'healthy',
            'timestamp': timezone.now().isoformat()
        })
    except Exception as e:
        return JsonResponse({
            'status': 'unhealthy',
            'error': str(e)
        }, status=503)
```

## Pattern 8: Logging Configuration

Configure structured logging:

**Node.js:**
```javascript
// logger.js
const winston = require('winston');

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.json(),
  defaultMeta: { service: '{{PROJECT_NAME}}' },
  transports: [
    new winston.transports.Console({
      format: winston.format.simple()
    })
  ]
});

// In production, logs go to stdout/stderr automatically
module.exports = logger;
```

**Python:**
```python
# logger.py
import logging
import sys

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[logging.StreamHandler(sys.stdout)]
)

logger = logging.getLogger(__name__)
```

## Pattern 9: Scaling Configuration

Configure dyno scaling:

```bash
# Scale web dynos
heroku ps:scale web=2 --app {{APP_NAME}}

# Scale worker dynos
heroku ps:scale worker=3 --app {{APP_NAME}}

# Use different dyno types
heroku ps:resize web=standard-2x --app {{APP_NAME}}
heroku ps:resize worker=performance-m --app {{APP_NAME}}
```

**Auto-scaling with Heroku Metrics:**
```bash
# Install Heroku Metrics CLI plugin
heroku plugins:install heroku-metrics

# View metrics
heroku metrics --app {{APP_NAME}}

# Set up autoscaling (requires add-on or custom solution)
```

## Pattern 10: SSL and Custom Domains

Configure SSL and custom domains:

```bash
# Add custom domain
heroku domains:add www.example.com --app {{APP_NAME}}

# Add SSL certificate
heroku certs:add server.crt server.key --app {{APP_NAME}}

# Or use ACM (Automatic Certificate Management)
heroku certs:auto:enable --app {{APP_NAME}}
```

**DNS Configuration:**
```
Type: CNAME
Name: www
Value: {{APP_NAME}}.herokuapp.com
```

## Best Practices

1. **Use release phase**: Run migrations in release phase, not in application startup
2. **Health checks**: Implement health check endpoints for zero-downtime deployments
3. **Environment parity**: Keep staging and production as similar as possible
4. **Log aggregation**: Use Papertrail or Logentries for log management
5. **Monitoring**: Set up New Relic or Datadog for application monitoring
6. **Database backups**: Enable automated backups for production databases
7. **Review apps**: Use review apps for testing pull requests
8. **Config vars**: Never commit secrets, use Heroku config vars
9. **Buildpacks**: Pin buildpack versions for consistency
10. **Dyno types**: Choose appropriate dyno types based on workload

## Common Pitfalls

- **Database migrations in startup**: Causes slow deploys and potential downtime
- **Hardcoded URLs**: Use environment variables for all external URLs
- **Missing health checks**: Prevents zero-downtime deployments
- **Logging to files**: Heroku filesystem is ephemeral, log to stdout
- **Stateful processes**: Heroku dynos are stateless, use external storage
- **Missing release phase**: Migrations should run in release phase
- **Unoptimized slug size**: Large slugs slow down deployments
- **Missing error tracking**: Set up Sentry or similar for error monitoring

## Verification Criteria

After generation, verify:
- ✓ Procfile defines all process types correctly
- ✓ app.json includes all required add-ons
- ✓ Environment variables are set for each environment
- ✓ Health check endpoint returns 200 for healthy state
- ✓ Database migrations run successfully in release phase
- ✓ Review apps are created for pull requests
- ✓ Staging auto-deploys from staging branch
- ✓ Production requires manual approval
- ✓ SSL certificates are configured for custom domains
- ✓ Logs are accessible via Papertrail or similar
- ✓ Monitoring is set up (New Relic/Datadog)
- ✓ Database backups are enabled for production

