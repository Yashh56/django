# Django on Render Deployment Checklist

Use this checklist before deploying a Django project to Render.

## 1. Project and Dependencies

- [ ] The project runs locally from a clean virtual environment.
- [ ] `requirements.txt` exists and is current:

  ```bash
  python -m pip freeze > requirements.txt
  ```

- [ ] These packages are included when used by the project:
  - `gunicorn`
  - `whitenoise`
  - `dj-database-url`
  - `psycopg2-binary`
- [ ] `manage.py` is committed to the repository.
- [ ] The Django project package and settings module are committed.
- [ ] `.gitignore` excludes virtual environments, local databases, secrets, and generated static files.

## 2. Django Settings

- [ ] The production secret key comes from an environment variable. Do not commit a real secret:

  ```python
  import os

  SECRET_KEY = os.environ['SECRET_KEY']
  ```

  A generated local fallback may be used only for local development.

- [ ] `DEBUG` is environment-controlled and disabled on Render:

  ```python
  DEBUG = os.environ.get('RENDER') is None
  ```

- [ ] `ALLOWED_HOSTS` includes Render's hostname:

  ```python
  ALLOWED_HOSTS = [
      host for host in [
          os.environ.get('RENDER_EXTERNAL_HOSTNAME'),
          'localhost',
          '127.0.0.1',
      ] if host
  ]
  ```

- [ ] WhiteNoise is installed and listed in `MIDDLEWARE`, with commas between every entry:

  ```python
  'whitenoise.middleware.WhiteNoiseMiddleware',
  ```

- [ ] Static files have both a URL and a collection directory:

  ```python
  STATIC_URL = 'static/'
  STATIC_ROOT = BASE_DIR / 'staticfiles'
  ```

- [ ] If using WhiteNoise compressed files, configure the storage backend as appropriate for the Django version.
- [ ] No password, API key, database URL, or production secret is hardcoded in source code.

## 3. Database

- [ ] Do not rely on SQLite for production. Render's filesystem is ephemeral.
- [ ] Create a Render PostgreSQL database.
- [ ] Add the database URL to the web service environment, normally as `DATABASE_URL`.
- [ ] Configure Django with `dj-database-url`:

  ```python
  import dj_database_url

  DATABASES = {
      'default': dj_database_url.config(
          conn_max_age=600,
          conn_health_checks=True,
      )
  }
  ```

- [ ] Confirm migrations run successfully against PostgreSQL.
- [ ] Plan backups and data migration before moving from SQLite.

## 4. URLs and Application Behavior

- [ ] `ROOT_URLCONF` points to the correct URL module.
- [ ] The root URL `/` has an intentional route. A deployed service with only `/admin/` will return 404 at its home page.
- [ ] Admin and all expected application routes are included.
- [ ] Templates, static assets, and media references use valid paths.
- [ ] Any required Django app is included in `INSTALLED_APPS`.

## 5. Build and Start Commands

A typical `build.sh` is:

```bash
#!/usr/bin/env bash
set -o errexit
python -m pip install -r requirements.txt
python manage.py collectstatic --no-input
python manage.py migrate
```

- [ ] `build.sh` is executable, or Render is configured to run it with Bash.
- [ ] Render Build Command is configured, for example:

  ```text
  ./build.sh
  ```

- [ ] Render Start Command points to the correct WSGI or ASGI module:

  ```text
  gunicorn myproject.wsgi:application
  ```

  For ASGI:

  ```text
  gunicorn myproject.asgi:application -k uvicorn.workers.UvicornWorker
  ```

- [ ] The module name matches the actual project package. Replace `myproject` when necessary.
- [ ] The service binds through Gunicorn; do not use `python manage.py runserver` in production.

## 6. Render Environment Variables

Configure these in the Render service, not in committed files:

- [ ] `SECRET_KEY`: a long, random production secret.
- [ ] `DATABASE_URL`: the internal Render PostgreSQL connection string.
- [ ] `RENDER`: set by Render or configured as needed by the settings strategy.
- [ ] Any third-party API keys, email credentials, OAuth secrets, and allowed origins.
- [ ] `CSRF_TRUSTED_ORIGINS`, if required by the application:

  ```text
  https://django-uhuw.onrender.com
  ```

## 7. Local Pre-deployment Checks

Run these from the activated virtual environment:

```bash
python manage.py check --deploy
python manage.py check
python manage.py migrate --check
python manage.py collectstatic --no-input
```

Test the important URLs:

```bash
python manage.py shell -c "from django.test import Client; c=Client(); print(c.get('/').status_code); print(c.get('/admin/').status_code)"
```

Also verify manually that:

- [ ] The home page loads with status `200`.
- [ ] The admin page loads or redirects to login.
- [ ] Static CSS and JavaScript load.
- [ ] Forms and CSRF-protected requests work.
- [ ] Database-backed pages can read and write data.
- [ ] Error pages do not expose debug tracebacks.

## 8. After Deploying

- [ ] Confirm the Render deploy reaches `Live`.
- [ ] Read the build logs and confirm migrations and `collectstatic` completed.
- [ ] Open the Render URL and test `/`.
- [ ] Check Render logs for `404`, `500`, database, host, and static-file errors.
- [ ] Verify `DEBUG` is `False` in the deployed environment.
- [ ] Confirm the application uses PostgreSQL rather than an ephemeral SQLite file.
- [ ] Test a real user flow, not only the health check.

## Common Errors

| Error | Likely cause | Fix |
| --- | --- | --- |
| `404` at `/` | No root URL pattern exists | Add a home view and `path('', ...)` route |
| `STATIC_ROOT` error during `collectstatic` | `STATIC_ROOT` is missing | Set `STATIC_ROOT = BASE_DIR / 'staticfiles'` |
| `DisallowedHost` | Render hostname is not allowed | Add `RENDER_EXTERNAL_HOSTNAME` to `ALLOWED_HOSTS` |
| Debug page in production | `DEBUG` is hardcoded to `True` | Read it from environment variables and disable it on Render |
| Data disappears after restart | SQLite is stored on ephemeral disk | Use Render PostgreSQL and `DATABASE_URL` |
| `ModuleNotFoundError` | Dependency missing from requirements | Install it and regenerate `requirements.txt` |
| Static files return 404 | Static collection or WhiteNoise configuration is incomplete | Configure `STATIC_ROOT`, run `collectstatic`, and verify middleware |
| Gunicorn cannot import application | Incorrect module path | Match the start command to the actual Django package |
