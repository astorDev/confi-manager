# Configuration Migration / Versioning

Throughout application lifecycle not only configuration values, but also their "list" (configuration schema) changes. It brings a whole set of challenges.

1. `my-app` start with version `0.1`. Submits schema and values to `v0.1` document in confi-manager, the configuration is also marked as `latest`
2. The running app reads `my-app/0.1` from confi-manager to get up-to-date values.
3. `my-app` with version `0.2` gets deployed. Supplies schema `v0.2` to confi-manager and reads `my-app/previous` configuration values from confi-manager. `previous` supplies values from `latest` excluding those that are not present in the new configuration.
4. App merges those values with high-precedence with the values it has from appsettings.json etc. Sends all to `my-app/0.2`. As a result both existing values (with high priority) and new values (if a field was added) are present in `my-app/0.2`. Also removed values are not supplied (since they are not present in new schema).

## Forced Reset

In normal situation, when a new version is deployed existing values from confi-manager should take precedence over the values configured locally. (The config server should be source of truth for configuration values it serve). However, it's possible to imagine a situation when with a newer software version we would want to "force" a new configuration. For example if we stabilized an experimental feature in this version and want to set a feature toggle to true.

Here's a list of requirements we want our solution to match:

- Supports multiple "forced-resets" throughout application life.
- Don't require knowing next version number beforehand.

A solution that covers all of the concerns seems to be `generation` field:

```json
{
    "feature_x_enabled" : {
        "type" : "bool",
        "x-confi" : {
            "generation" : 2 // 0 by default (0 instead of 1 as an extra safety)
        }
    }
}
```

Here's how it works:

1. confi-manager stored `feature_x_enabled` as `false` with `generation` as `0` by default.
2. application v2026.04 serves new schema to `confi-manager`
3. manager serves curret config to application v2026.04, **but** excludes all fields where generation doesn't match, so `feature_x_enabled` is excluded.
4. application uses (sends to confi-manager) `true` from it's appsettings.json, just like it would for a new field.
5. confi-manager's `my-app/v2026.04` stores `feature_x_enabled` as `true`.

## Rollbacks

When we deploy an app `v2026.04` we also create a configuration `v2026.04` (consist of schema and values). The configuration `v2026.04` starts to be considered `latest` and a new configuration will be "branching" from it and it's schema. Therefore all changes to `v2026.03` would be "lost".

> 🚧 Requires Investigation.