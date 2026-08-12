# Setup Issues

## Python Version

* While setting up the Python environment with uv but got the following error:

```
Creating virtual environment at: .venv
Resolved 99 packages in 10ms
      Built markupsafe==3.0.2
  × Failed to build `pillow==10.2.0`
  ├─▶ The build backend returned an error
  ╰─▶ Call to `backend.build_wheel` failed (exit status: 1)

      [stderr]
      Traceback (most recent call last):
        File "<string>", line 14, in <module>
          requires = get_requires_for_build({})
        File "/Users/admin/.cache/uv/builds-v0/.tmpXHlaEy/lib/python3.14/site-packages/setuptools/build_meta.py", line 333, in get_requires_for_build_wheel
          return self._get_build_requires(config_settings, requirements=[])
                 ~~~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        File "/Users/admin/.cache/uv/builds-v0/.tmpXHlaEy/lib/python3.14/site-packages/setuptools/build_meta.py", line 301, in _get_build_requires
          self.run_setup()
          ~~~~~~~~~~~~~~^^
        File "/Users/admin/.cache/uv/builds-v0/.tmpXHlaEy/lib/python3.14/site-packages/setuptools/build_meta.py", line 317, in run_setup
          exec(code, locals())  # noqa: S102 # exec is intentional here
          ~~~~^^^^^^^^^^^^^^^^
        File "<string>", line 31, in <module>
        File "<string>", line 28, in get_version
      KeyError: '__version__'

      hint: This usually indicates a problem with the package or the build environment.
  help: `pillow` (v10.2.0) was included because `backend` (v0.1.0) depends on `pillow`

  ~/PycharmProjects/New_devs_App/backend    main ?1   
```

### Root cause

`backend/.python-version` was set to `3`, so uv selected the latest installed Python (**3.14**).  
`pillow==10.2.0` (pinned in [`backend/pyproject.toml`](../backend/pyproject.toml)) does not support Python 3.14 and fails while building from source.

The editor also reported an invalid interpreter because `python.defaultInterpreterPath` points at `backend/.venv/bin/python`, and that venv was incomplete after the failed sync.

### Fix

1. Pin a compatible Python version in [`backend/.python-version`](../backend/.python-version):

```text
3.12
```

2. Recreate the venv and sync dependencies:

```bash
cd backend
rm -rf .venv
uv sync --python 3.12
```

3. Select the Python interpreter:

`./backend/.venv/bin/python`

Verified with Python 3.12.10; Pillow 10.2.0 installs cleanly and the invalid interpreter warning clears.

## Supabase URL

I found this error when running `docker-compose up`:

```
backend-1   | ERROR:app.core.supabase_connection_pool:❌ Failed to initialize Supabase connection pool: supabase_url is required
backend-1   | ERROR:app.main:❌ Supabase connection pool initialization failed: supabase_url is required
```

### Root cause

`SUPABASE_URL` is not set. The backend tries to start a Supabase connection pool, fails, and logs the error. Startup still continues in challenge/mock mode. Local data uses Postgres and Redis from docker-compose, not Supabase.

### Fix

No fix needed. I do not set `SUPABASE_URL` for this assignment. I just check that the server finishes starting (`Application startup complete` / listening on `:8000`), then open http://localhost:3000 and http://localhost:8000/docs.
