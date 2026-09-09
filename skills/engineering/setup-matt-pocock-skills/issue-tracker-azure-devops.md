# Issue tracker: Azure DevOps

Issues and specs for this repo live as Azure Boards work items. Pull requests live in Azure Repos. Use Azure CLI with the `azure-devops` extension for all operations.

## Configuration

- **Organization**: `<organization-url>`
- **Project**: `<project>`
- **Parent work item type**: `<parent-work-item-type>`
- **Child work item type**: `<child-work-item-type>`

Pass `--org "<organization-url>" --project "<project>"` explicitly to commands. This keeps Boards operations pointed at the configured project even when the current clone uses a different remote.

Install the extension with `az extension add --name azure-devops`. Authenticate with `az login`, or use `az devops login` when the organization requires a personal access token.

## Conventions

- **Create a parent**: create specs and maps with `az boards work-item create --org "<organization-url>" --project "<project>" --type "<parent-work-item-type>" --title "..." --description "..."`.
- **Create a child ticket**: create implementation and decision tickets with `az boards work-item create --org "<organization-url>" --project "<project>" --type "<child-work-item-type>" --title "..." --description "..."`, then link each one from its parent with `az boards work-item relation add --org "<organization-url>" --id <parent-id> --relation-type Child --target-id <task-id>`.
- **Publish a Markdown body**: for specs, tickets, maps, and briefs, update the created work item through `az devops invoke --area wit --resource workItems` using JSON Patch. Set `/fields/System.Description` to the Markdown body and `/multilineFieldsFormat/System.Description` to `Markdown`.
- **Read a work item**: `az boards work-item show --org "<organization-url>" --id <id> --expand all`. Fetch its comments with `az devops invoke --org "<organization-url>" --area wit --resource comments --route-parameters project="<project>" workItemId=<id> --http-method GET --api-version 7.1-preview.4`.
- **List work items**: use `az boards query --org "<organization-url>" --project "<project>" --wiql "..."`. Scope every WIQL query with `[System.TeamProject] = @Project` and select the fields the skill needs.
- **Comment on a work item**: use `az boards work-item update --org "<organization-url>" --id <id> --discussion "..."` for plain text. For generated Markdown such as triage notes or agent briefs, call the Comments REST resource through `az devops invoke` with `--query-parameters format=Markdown`, API version `7.1-preview.4`, and a body containing `{"text":"..."}`.
- **Assign a work item**: `az boards work-item update --org "<organization-url>" --id <id> --assigned-to "<identity>"`.
- **Apply / remove triage roles**: store canonical triage roles as Azure Boards tags in `System.Tags`. Read the current tags first, merge or remove the requested role locally, then write the full semicolon-separated value with `az boards work-item update --org "<organization-url>" --id <id> --fields "System.Tags=<merged-tags>"`. Never overwrite unrelated tags.
- **Close a work item**: discover the work item's state whose category is `Completed` with `az devops invoke --org "<organization-url>" --area wit --resource workitemtypestates --route-parameters project="<project>" type="<work-item-type>" --http-method GET --api-version 7.1`, then run `az boards work-item update --org "<organization-url>" --id <id> --state "<completed-state>" --discussion "..."`. Do not assume the state is named `Done` or `Closed`.
- **Reopen a work item**: inspect the same state metadata and choose the appropriate `Proposed` or `InProgress` state rather than guessing a process-specific name.

Azure Boards work items are referenced as `AB#<id>` in commits and pull-request descriptions. A full work-item URL is also valid. Bare `#<id>` is not an Azure Boards reference.

## Pull requests as a triage surface

**PRs as a request surface: no.** _(Set to `yes` if this repo treats Azure Repos PRs as feature requests; `/triage` reads this flag.)_

**Maintainer identities:** _(When enabling the flag, list the Azure DevOps identity unique names that should be excluded from discovery. Azure Repos has no `authorAssociation` equivalent. If this is empty, ask the maintainer before listing PRs for triage.)_

When set to `yes`, PRs run through the same roles and states as work items:

- **Read a PR**: `az repos pr show --org "<organization-url>" --id <pr-id>`. Take the repository ID from the returned `repository.id`, then fetch threads with `az devops invoke --org "<organization-url>" --area git --resource pullRequestThreads --route-parameters project="<project>" repositoryId="<repository-id>" pullRequestId=<pr-id> --http-method GET --api-version 7.1`.
- **Inspect the diff**: `az repos pr checkout --id <pr-id>`, then compare the checked-out source branch with the PR's target branch using git.
- **List PRs for triage**: `az repos pr list --org "<organization-url>" --project "<project>" --repository "<repository>" --status active`, then keep only PRs whose `createdBy.uniqueName` is not in **Maintainer identities**.
- **Comment on a PR**: create a general thread with `az devops invoke --org "<organization-url>" --area git --resource pullRequestThreads --route-parameters project="<project>" repositoryId="<repository-id>" pullRequestId=<pr-id> --http-method POST --api-version 7.1 --in-file <body.json>`. The body contains `{"comments":[{"parentCommentId":0,"content":"...","commentType":1}],"status":1}`. Delete the temporary body file after the request.
- **Apply / remove PR triage roles**: Azure Repos PR labels are separate from Azure Boards work-item tags. Use `az devops invoke --area git --resource pullRequestLabels` to list, create, or delete labels for the PR, preserving unrelated labels.
- **Close a PR**: `az repos pr update --org "<organization-url>" --id <pr-id> --status abandoned`.

Azure Boards work-item IDs and Azure Repos PR IDs are separate. Resolve `AB#42` as a work item, `PR 42` or a PR URL as a pull request, and ask when given a bare number.

## When a skill says "publish to the issue tracker"

Create an Azure Boards work item of the configured parent type for a spec or parent effort. Create a work item of the configured child type for an implementation or decision ticket and link it as a child of the parent.

For `/to-spec`, the published spec is the parent work item. For `/to-tickets`, use the source Azure Boards parent when one exists. If the source is only a conversation or a local document, create one parent work item for the effort before creating its child tickets.

## When a skill says "fetch the relevant ticket"

Resolve `AB#<id>` or the work-item URL, then read the work item and all comments.

## Wayfinding operations

Used by `/wayfinder`. The **map** uses the configured parent type and its **child** tickets use the configured child type. These default to Feature and Task.

- **Map**: create a work item of the configured parent type whose body holds Destination / Notes / Decisions-so-far / Fog, then add the `wayfinder:map` tag without removing existing tags.
- **Child ticket**: create a work item of the configured child type, add the matching `wayfinder:<type>` tag, then add it as a child with `az boards work-item relation add --org "<organization-url>" --id <map-id> --relation-type Child --target-id <child-id>`. Once claimed, assign it to the driving developer.
- **Blocking**: Azure Boards' native Predecessor/Successor links are canonical. For each edge where `<blocker>` must finish before `<blocked>`, add `<blocker>` as the predecessor of `<blocked>` with `az boards work-item relation add --org "<organization-url>" --id <blocked> --relation-type Predecessor --target-id <blocker>`. A ticket is unblocked when every predecessor is in a state whose category is `Completed`.
- **Frontier query**: `az boards query` supports only flat WIQL, so use the WIQL REST resource through `az devops invoke` for a `FROM WorkItemLinks` query over `System.LinkTypes.Hierarchy-Forward`. For each child, inspect `System.LinkTypes.Dependency-Reverse` predecessor relations and state metadata; drop children with an incomplete predecessor or an assignee. First in map order wins.
- **Claim**: assign the child work item to the driving developer before any work. This is the session's first write.
- **Resolve**: add the answer with `--discussion`, transition the child work item to the dynamically discovered `Completed` state, then append a context pointer to the map's Decisions-so-far.
