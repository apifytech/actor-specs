# Actor file specification

This JSON file must be present at `.actor/actor.json` and defines core properties of a single web Actor.

The file has the following structure:

```jsonc
{
  // Required, indicates that this is an Actor definition file and the specific version of the Actor specification.
  "actorSpecification": 1,
  
  // Properties
  "name": "google-search-scraper",
  "title": "Google Search Scraper",
  "description": "A 200-char description",
  "version": "0.0", // Required
  "buildTag": "latest", // If omitted, builds with "latest" tag
  "environmentVariables": {
    "MYSQL_USER": "my_username",
    "MYSQL_PASSWORD": "@mySecretPassword"
  },
  
  // If true, the Actor indicates it can be run in the Standby mode,
  // to get started and be kept alive by the system to handle incoming HTTP REST requests by the Actor's web server.
  "usesStandbyMode": true,
 
  // A meta object enabling impelemtations to pass arbitrary additional properties
  "meta": {
    "something": "bla bla"
  },
  
  // Optional min and max memory for running this Actor (TODO: shouldn't this also be in "meta"?)
  "minMemoryMbytes": 128,
  "maxMemoryMbytes": 4096,
  
  // Used when user doesn't specify memory, the 𝔸ctor run will start with this amount.
  // It might be a basic arithemtic expression referencing ${variables} from Actor input.
  // This right value can optimize user experience vs. compute costs.
  "defaultMemoryMbytes": "${maxCrawlPages} * 256 + 128",
  
  // Links to other Actor defintion files
  "dockerfile": "./Dockerfile", // If omitted, the system looks for "./Dockerfile" and "../Dockerfile"
  "readme": "./README.md", // If omitted, the system looks for "./ACTOR.md" and "../README.md"
  "changelog": "../../../shared/CHANGELOG.md",
  
  // Links to input/output JSON schema files, or inlined JSON schema objects.
  "input": "./input_schema.json",
  "output": "./output_schema.json",
  
  // Links to storages schema files, or inlined schema objects.
  // These aren't standard JSON schema files, but our own format. See ./DATASET_SCHEMA.md
  "storages": {
    "keyValueStore": "./key_value_store_schema.json",
    "dataset": "../shared_schemas/generic_dataset_schema.json",
    "requestQueue": "./request_queue_schema.json"
  },
   
  // Link to OpenAPI schema file for the Standby web server API
  "webServer": "./web_server_openapi.json",
  // TODO: I'd use just above, instead of extra line with a second server:
  // "standbyServer": "./standby_openapi.json",
  // See https://github.com/apify/actor-specs/issues/40

  // Scripts that might be used by the CLI to ease the local Actor development.
  "scripts": {
    "post-create": "npm install",
    "run": "npm start"
  }
}
```

## Random notes

The `.actor/actor.json` replaces the legacy `apify.json` file.
Here are the notes comparing the format to the previous version:

- We removed the `template` property as it's not needed for anything, it only stored the original template
- There's a new `title` field for a human-readable name of the Actor.
  We're moving towards having human-readable names shown for Actors everywhere,
  so it makes sense to define `title` directly in the source code.
- Similarly, we added `description` for the short description of what the Actor does.
- When calling `actor push` and the `title` or `description` are already set
  on the Actor (maybe SEO-optimized versions from copywriter),
  by default we do not overwrite them
  unless `apify push` is called with options `--force-title` or `--force-description`.
- The `name` doesn't contain username, so that the Actor can be easily deployed
  to any user account. This is useful for tutorials and examples, as well as
  pull requests done externally to create Actors from existing source code files
  owned by external developers
  (the developer might not have Apify account yet, and we might want to show them deployment
  to some testing account).
  Note that `apify push` has option `--target=eva/my-actor:0.0` that allows
  deployment of the Actor under a different user account, using permissions
  and personal API token of the current user.
  We should also add options to override only parts of this, 
  like `--target-user` (ID or username), `--name`, `--build-tag` and `--version`,
  it would be useful e.g. in CI for beta versions etc.
- Note that `version` and `buildTag` are shared across Actor deployments to
  all user accounts, similarly as with software libraries,
  and hence they are part of `actor.json`.
- The `dockerfile` property points to a Dockerfile that is to be used to build the
  Actor image. If not present, the system looks for Dockerfile in the `.actor` directory
  and if not found, then in Actor's top-level
  directory. This setting is useful if the source code repository has some
  other Dockerfile in the top-level directory, to separate Actor Docker image from the
  other one. Note that paths in Dockerfile are ALWAYS relative to the Dockerfile's location.
  When calling `apify run`, the system runs the Actor using the Dockerfile.
- `env` was renamed to `environmentVariables` for more clarity. `apify build` or `apify run`
  could have an option `--apply-env-vars-to-build` like we have it on platform.
- The `dockerfile` and `readme` directives are optional, the system falls back to reasonable
  defaults, first in `.actor` directory and then in the top-level directory.
- "scripts" section added, see https://apify.slack.com/archives/C04HB9V90DT/p1672826248186569
