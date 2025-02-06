# Welcome

You can find information about code produced by Orange in this space.

Orange team can be reach on Slack in channel `#bidbax_basefarm_platform_ext`

## Git conventions

### Conventional commits

Commit messages should follow the [`Conventional Commits`](https://www.conventionalcommits.org/en/v1.0.0/) standard before a merge request to the `main` branch is accepted.

---

``` text
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

---

### Types

* **build**: Changes that affect the build system or external dependencies (example scopes: gulp, broccoli, npm)
* **ci**: Changes to our CI configuration files and scripts (example scopes: Travis, Circle, BrowserStack, SauceLabs)
* **docs**: Documentation only changes
* **feat**: A new feature
* **fix**: A bug fix
* **perf**: A code change that improves performance
* **refactor**: A code change that neither fixes a bug nor adds a feature
* **style**: Changes that do not affect the meaning of the code (white-space, formatting, missing semi-colons, etc)
* **test**: Adding missing tests or correcting existing tests

### Scopes

* **bastion**
* **cloudshell**
* **csirt**
* **ddos**
* **dns**
* **mgmt**
* **buildserver**
* **socksproxy**
* **soteria**
* **splunk**
* **vwan**
* **policy**

### Examples

``` text
ci(splunk): update spn client id

feat(vwan): test Virtual WAN in Core Test subscription

fix(mgmt): add missing custom tags to management vm resources
```

## Naming conventions

A couple of naming conventions for bicep code and workflows, so it is easy to jump in without knowning the entire service to make changes.

### Bicep

All services should have a `main.bicep` file with a linked parameter file (`parameters/main.bicepparm`).

Use bicep parameter files and not ARM (JSON) parameter files.

Use module to split up the code in blocks, so it is easier to read and modify later on.

Code related to virtual network should be placed in a sub-directory of `modules/` named `vnets` and security rules should be placed in sub-directory of `vnets/` named `securityRules`.

If you need other folders, like `scripts/`, it should be placed as a sub-directory directly below the service parent folder.

Example structure,

📦 bastion  
 ┣ 📂 modules  
 ┃ ┣ 📂 vnets  
 ┃ ┃ ┣ 📂 securityRules  
 ┃ ┃ ┃ ┗ 📜 my-nsg-rules.json  
 ┃ ┃ ┗ 📜 vnet.bicep  
 ┃ ┣ 📜 bastionHosts.bicep  
 ┃ ┗ 📜 virtualNetworkPeering.bicep  
 ┣ 📂 parameters  
 ┃ ┗ 📜 main.bicepparam  
 ┣ 📂 scripts  
 ┃ ┗ 📜 my-script.sh  
 ┗ 📜 main.bicep  

### Github Actions Workflow

All workflows should have a name with a emoji in front, 🚀 for deployment and 🔋 for reusable workflows.

Add `deploy-` or `reusable-` as prefix to the workflow file name.

Do not set `name:` parameter for reusable workflows, leave it out of code.

Add `Deploy` as a prefix to workflow name, if workflow creates a deployment in Azure.

Add `workflow_dispatch:` to the workflow, so it can be executed without a PR/code change

Add `concurrency:` to workflow, so that we don't run more than one instance of a workflow at the same time.

All workflows should have a `schedule:` block so that each workflow are exected weekly.

### MKdocs

Install `markdownlint` extension for VScode to get correct markdown syntax linting.

All documents below `docs/`, should be correctly formated in accordance with markdown rules.
