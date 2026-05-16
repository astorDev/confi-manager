# Confi Manager Storage

Core storage unit of confi manager is JSON: Configuration Value JSON and Schema JSON. Based on that 2 storage options seem appealing:

- MongoDB
- Git

| Feature                                  | MongoDb | Git          |
| ---------------------------------------- | ------- | ------------ |
| [Bring Your Own Storage](#byos)          | ✅       | ☑️ partially |
| [What you see is what you get](#wysiwyg) | ❌       | ✅           |
| Architectural Unambiguity                | ✅       | ❌           |

## BYOS

Both MongoDb and Git will let us use "bring your own storage approach". For example we can let user connect to MongoDb Atlas or Github. With MongoDb the approach is trivial: one server per confi for company or company environment. With Git it's highly dependent on the way we architect the storage itself though and it may be the case, that we need a git allowance that existing doesn't provide or doesn't provide for free (like creating an unlimited number of repositories).

## WYSIWYG

With Git storage the way we store files will be roughly the same as we represent them in the [editor](../editor/README.md). This makes the system "transparent", which is typically considered a benefit: For example Obsidian -> Markdown Files, Agents -> Markdown Files. Reusing an existing technology normally makes system more internally robust and simple at the same time, since complexity is moved to an underlying system. See details in the [Git Section](#git).

## Git

Git seems to be pretty standard. Supposedly for that reason, [Sprint Cloud Config](../inspirations/sprint-cloud-config.md) architectured they solution around git.

However, "Git" doesn't answer a couple of high-level architectural questions about the storage. Here's what still needs to be desided:

- 🚧 "versions as branches" VS "versions as folders"
- 🚧 "app as repo" VS "app as folder"

An advantage of git is that it has a "built-in" mechanism for approvals: git branches.
With "app as repo" approach all [access](../access/README.md) concerns, including [approvals](../access/approvals.md) can be "delegated" to Github, Gitlab, etc with PRs and repository access management. This is presumably how [Sprint Cloud Config](../inspirations/sprint-cloud-config.md) solves it. However, without an ability to automatically create such repos the DX may become tedious.

**Github / Gitlab**:

- Doesn't limit numbers of repos we can create. 
- Github API allows creating repos.

However, if using it we no longer purely use "git". Now we use Github or Gitlab API. This can be fine as an integration extension, but definately doesn't classify for the core product.