# VS Code Based Editing

Having that configuration editing is basically validated JSON editing, using VS Code as an editor seems like an appealing solution:

- Supposedly requires less work
- An integrated solution can be **more** attractive for developers due to a non-interrupted DX.

The solution can be split into two flows: [CLI-Driven](#cli-driven) and [Extension-drive](#extension-driven)

## CLI-driven

1. CLI Uploads config and schema in an "editing directory". Id and version included in either schema, file name, special file. 🚧 Pending metadata storage decision
1. CLI either hints or opens that directory. (Can use `code` command if available) 🚧 Pending flow design
2. In VS Code we can edit the config value with schema validation built-in. 🚧 Pending testing of validation
3. In CLI an `apply` / `save` command is called to apply the changes. 🚧 Pending command name decision

## Extension-driven

> 🚧 Requires design