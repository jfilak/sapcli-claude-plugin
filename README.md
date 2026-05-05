# sapcli Claude Code Plugin

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that brings SAP ABAP system exploration and analysis capabilities directly into your AI-assisted development workflow.

Built on top of [sapcli](https://github.com/jfilak/sapcli) -- the command line interface to SAP products.

## What's Included

### Agents

- **abap-explorer** -- An ABAP architect agent that analyzes SAP system implementations by reading ABAP code, navigating package hierarchies, querying database tables, and finding object usages. Automatically invoked when you ask Claude about ABAP code, classes, packages, or SAP system objects.

- **abap-classic-developer** -- A classic ABAP developer agent that works with the classic ABAP objects — programs, includes, classes, interfaces, function groups, and dictionary objects (tables, structures, data elements) — including implementing features, modifying code, writing and analyzing unit tests (AUnit), and running static checks (ATC). Automatically invoked when you ask Claude to implement, modify, or test classic ABAP objects.

### Skills

- **abap-system-info** -- Fetches and displays ABAP system information such as system ID, client, and user details. Useful for troubleshooting and confirming connection details.
- **abap-sql-query** -- Executes SQL queries against ABAP system to fetch and return structured data.

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI installed and configured
- [sapcli](https://github.com/jfilak/sapcli) installed and accessible on your `PATH`
- Python 3.10+
- SAP system connection configured via environment variables (see [sapcli configuration](https://github.com/jfilak/sapcli/blob/master/doc/configuration.md))

### Installing sapcli

```bash
pipx install git+https://github.com/jfilak/sapcli.git
```

### Configuring SAP Connection

Create a configuration file and source it before starting Claude Code:

```bash
cat > .sapcli.openrc << _EOF
export SAP_USER=<your_user>
export SAP_PASSWORD=<your_password>
export SAP_ASHOST=<your_host>
export SAP_CLIENT=<your_client>
export SAP_PORT=8000
export SAP_SSL=no
_EOF

source .sapcli.openrc
```

## Installation

### Via Claude Code Plugin Marketplace (recommended)

The easiest way to install this plugin is through the Claude Code plugin marketplace.

**Step 1: Add the marketplace**

Run the following command inside Claude Code to register the marketplace:

```shell
/plugin marketplace add jfilak/sapcli-claude-plugin
```

**Step 2: Install the plugin**

```shell
/plugin install sapcli-abap-tools@sapcli-plugins
```

Or use the interactive UI -- run `/plugin`, go to the **Discover** tab, find **sapcli-abap-tools**, and choose your installation scope:

- **User scope** -- install for yourself across all projects
- **Project scope** -- install for all collaborators on this repository
- **Local scope** -- install for yourself in this repository only

**Step 3: Activate**

```shell
/reload-plugins
```

**Updating the plugin**

To fetch the latest version:

```shell
/plugin marketplace update sapcli-plugins
```

You can also enable auto-updates via the `/plugin` UI under the **Marketplaces** tab.

**Uninstalling**

```shell
/plugin uninstall sapcli-abap-tools@sapcli-plugins
```

### Team-wide setup

To automatically prompt team members to install this plugin when they trust the project folder, add the following to your project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "sapcli-plugins": {
      "source": {
        "source": "github",
        "repo": "jfilak/sapcli-claude-plugin"
      }
    }
  },
  "enabledPlugins": {
    "sapcli-abap-tools@sapcli-plugins": true
  }
}
```

### Manual installation (alternative)

Clone this repository and register it directly in your Claude Code settings:

```bash
git clone https://github.com/jfilak/sapcli-claude-plugin.git
```

Then add to your `.claude/settings.json`:

```json
{
  "plugins": ["../sapcli-claude-plugin"]
}
```

Adjust the path to match where you cloned the repository relative to your project.

## Usage Examples

Once the plugin is loaded, you can interact with your SAP system naturally through Claude Code:

```
> What classes are in package $MY_PACKAGE?

> Explain the implementation of class ZCL_MY_CLASS

> Show me where function module Z_MY_FM is used

> Query the MARA table for material 12345

> Create a new ABAP function module enabling System Logon for a given ICF path

> Cover the class ZCL_UNDER_TESTED with unit tests

> /abap-system-info
```

The **abap-explorer** agent is triggered automatically when your questions relate to ABAP code or SAP system objects. It uses `sapcli` commands under the hood to read source code, navigate packages, run SQL queries, and trace object usage. The **abap-classic-developer** agent is triggered automatically when you ask to implement, modify, or test classic ABAP objects.

## Supported sapcli Operations

The plugin leverages the following sapcli capabilities:

| Capability | Command | Description |
|---|---|---|
| Package browsing | `sapcli package list` | List and explore package hierarchies |
| Read source code | `sapcli read program/class/functionmodule` | Read ABAP source objects |
| Export to files | `sapcli checkout` | Export source code for local analysis |
| SQL queries | `sapcli datapreview osql` | Query SAP database tables |
| Where-used | `sapcli program whereused` | Find object references |
| Object search | `sapcli abap find` | Search for objects by name |
| System info | `sapcli abap systeminfo` | Retrieve system metadata |

## License

See [sapcli license](https://github.com/jfilak/sapcli/blob/master/LICENSE) for the underlying CLI tool.
