## eoapi.stac

![](https://user-images.githubusercontent.com/10407788/151456592-f61ec158-c865-4d98-8d8b-ce05381e0e62.png)v

### Installation

Set the following environmental variables

```bash
- POSTGRES_USER=user
- POSTGRES_PASS=password
- POSTGRES_DBNAME=postgis
- POSTGRES_HOST_READER=localhost
- POSTGRES_HOST_WRITER=localhost
- POSTGRES_PORT=5432
```

Run with `uv` to install dependencies and start the server:

```bash
uv run uvicorn eoapi.stac.app:app --host 0.0.0.0 --port 8081
```
