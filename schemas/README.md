# Test schemas for da-sc

Demo JSON Schemas for testing the `/schemas` endpoint of the
[da-sc](https://github.com/adobe-rnd/da-sc) Worker locally. They are test fixtures,
not production schemas.

```text
schemas/shared/
  demo-card/v1.0.0.json    # required "title", optional "description"
  demo-card/v1.1.0.json    # adds optional "image" ($ref to demo-image v1.0.0)
  demo-image/v1.0.0.json   # required "url", optional "alt"
```

Each `$id` uses the canonical origin `http://localhost:8787`, the address of the
local da-sc Worker. Published versions are immutable: to change a schema, add a new
version file instead of editing an existing one.

## Testing with a local da-sc Worker

da-sc only serves paths listed in its allowlist. The committed allowlist contains
production entries only, so add the entries below locally and **never commit them**.

1. Create a fine-grained GitHub token for `stefangrimm/da-test` only, with
   *Contents: Read-only* permission and a short expiration.

2. In the da-sc checkout, create `.dev.vars` (ignored by git):

   ```
   GITHUB_TOKEN=<your token>
   JSON_SCHEMA_GITHUB_CONTENTS_URL=https://api.github.com/repos/stefangrimm/da-test/contents
   JSON_SCHEMA_GITHUB_REF=schema-delivery-dev
   JSON_SCHEMA_CANONICAL_ORIGIN=http://localhost:8787
   ```

   `wrangler.toml` already sets `JSON_SCHEMA_CACHE_MODE=bypass` for local development,
   so edited files are not cached.

3. In da-sc, replace the allowlist in `src/json-schema-allowlist.ts` temporarily:

   ```ts
   // TEMPORARY: local da-test entries, do not commit.
   export const JSON_SCHEMA_ALLOWLIST: JsonSchemaAllowlist = [
     'schemas/shared/demo-card/v1.0.0.json',
     'schemas/shared/demo-card/v1.1.0.json',
     'schemas/shared/demo-image/v1.0.0.json',
     // Intentionally missing in this repository: tests the GitHub 404 case.
     'schemas/shared/demo-card/v2.0.0.json',
   ];
   ```

4. Start the Worker with `npm run dev` and check the responses:

   ```sh
   B=http://localhost:8787/schemas/shared
   curl -i $B/demo-card/v1.0.0.json    # 200, application/schema+json
   curl -i $B/demo-card/v2.0.0.json    # 404 (allowlisted, missing in GitHub)
   curl -i $B/demo-card/v9.9.9.json    # 404 (not allowlisted, no GitHub call)
   ```

5. When done, revert the allowlist so the entries are never committed:

   ```sh
   git checkout -- src/json-schema-allowlist.ts
   ```

The files must be pushed to the `schema-delivery-dev` branch on GitHub before the
Worker can read them; local, unpushed changes are not visible to it.
