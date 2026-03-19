# Datasource Knowledge

Anton reads this file when connecting data sources. For each source, the YAML
block defines the fields Python collects. The prose below describes auth flows,
common errors, and how to handle OAuth2 — Anton handles those using the scratchpad.

Credentials are injected as `DS_<FIELD_UPPER>` environment variables
before any scratchpad code runs. Never embed raw values in code strings.

---

## PostgreSQL
```yaml
engine: postgres
display_name: PostgreSQL
pip: psycopg2-binary
name_from: database
fields:
  - { name: host,     required: true,  secret: false, description: "hostname or IP of your database server" }
  - { name: port,     required: false, secret: false, description: "port number", default: "5432" }
  - { name: database, required: true,  secret: false, description: "name of the database to connect" }
  - { name: user,     required: true,  secret: false, description: "database username" }
  - { name: password, required: true,  secret: true,  description: "database password" }
  - { name: schema,   required: false, secret: false, description: "defaults to public if not set" }
  - { name: ssl,      required: false, secret: false, description: "enable SSL (true/false)" }
test_snippet: |
  import psycopg2, os
  conn = psycopg2.connect(
      host=os.environ['DS_HOST'], port=os.environ.get('DS_PORT','5432'),
      dbname=os.environ['DS_DATABASE'], user=os.environ['DS_USER'],
      password=os.environ['DS_PASSWORD'],
  )
  conn.close()
  print("ok")
```

Common errors: "password authentication failed" → wrong password or user.
"could not connect to server" → wrong host/port or firewall blocking.

---

## HubSpot
```yaml
engine: hubspot
display_name: HubSpot
pip: hubspot-api-client
auth_method: choice
auth_methods:
  - name: pat
    display: "Private App Token (recommended)"
    fields:
      - { name: access_token, required: true, secret: true, description: "HubSpot Private App token (starts with pat-na1-)" }
  - name: oauth2
    display: "OAuth2 (for multi-account or publishable apps)"
    fields:
      - { name: client_id,     required: true,  secret: false, description: "OAuth2 client ID" }
      - { name: client_secret, required: true,  secret: true,  description: "OAuth2 client secret" }
    oauth2:
      auth_url: https://app.hubspot.com/oauth/authorize
      token_url: https://api.hubapi.com/oauth/v1/token
      scopes: [crm.objects.contacts.read, crm.objects.deals.read]
      store_fields: [access_token, refresh_token]
test_snippet: |
  import hubspot, os
  client = hubspot.Client.create(access_token=os.environ['DS_ACCESS_TOKEN'])
  client.crm.contacts.basic_api.get_page(limit=1)
  print("ok")
```

For Private App Token: HubSpot → Settings → Integrations → Private Apps → Create.
Recommended scopes: `crm.objects.contacts.read`, `crm.objects.deals.read`, `crm.objects.companies.read`.

For OAuth2: collect client_id and client_secret, then use the scratchpad to:
1. Build the authorization URL using `auth_url` + params above
2. Start a local HTTP server on port 8099 to catch the callback
3. Open the URL in the user's browser with `webbrowser.open()`
4. Extract the `code` from the callback, POST to `token_url` for tokens
5. Return `access_token` and `refresh_token` to store in wallet

---

## Snowflake
...

## Adding a new data source

Follow the YAML format above. Add to `~/.anton/datasources.md` (user overrides).
Anton merges user overrides on top of the built-in registry at startup.