# Visibility & Contribution Strategy

## Active Maintainers & Reviewers — `openshift/lightspeed-service`

Based on recent PRs and commits:

| Person | Role | GitHub |
|--------|------|--------|
| **onmete** | Very active contributor/maintainer (most recent commits) | <a href="https://github.com/onmete">@onmete</a> |
| **blublinsky** | Active contributor | <a href="https://github.com/blublinsky">@blublinsky</a> |
| **bparees** | PR reviewer | <a href="https://github.com/bparees">@bparees</a> |
| **xrajesh** | PR reviewer / contributor | <a href="https://github.com/xrajesh">@xrajesh</a> |
| **tisnik** | PR reviewer | <a href="https://github.com/tisnik">@tisnik</a> |
| **thoraxe** (Erik Jacobs) | Contributor (94 commits), ols-mcp PoC author | <a href="https://github.com/thoraxe">@thoraxe</a> |
| **joshuawilson** | PR reviewer | <a href="https://github.com/joshuawilson">@joshuawilson</a> |

## How to Get Visibility

### 1. Open an Issue on `openshift/lightspeed-service`
This is where the active development is. File an issue describing what our fork adds/improves to the MCP integration. Tag it as a feature request or enhancement.

### 2. Open a PR Directly Against `openshift/lightspeed-service`
Since MCP is already integrated there, contribute improvements directly. The active reviewers (onmete, bparees, xrajesh, tisnik) will see it.

### 3. Reach thoraxe Directly
Since he authored the PoC and is still active in the main repo, he'd know whether the standalone ols-mcp is officially abandoned in favor of the integrated approach.

### 4. Reference the Jira Board
The team tracks work via Red Hat Jira (e.g., `OLS-2747`, `OLS-2788`). If access is available, filing a Jira ticket gets it on their radar formally.

## Fork Landscape

- <a href="https://github.com/thoraxe/ols-mcp">thoraxe/ols-mcp</a> — Original PoC (dormant since Nov 2025)
- <a href="https://github.com/netbrah/ols-mcp">netbrah/ols-mcp</a> — Our fork (active)
- <a href="https://github.com/moohomelab/ols-mcp">moohomelab/ols-mcp</a> — Fork (Feb 2026)
- <a href="https://github.com/dialvare/ols-mcp">dialvare/ols-mcp</a> — Fork (Dec 2025)

The standalone PoC approach may not get broad traction since the main service already absorbed MCP functionality. The best strategy is **dual-track**: maintain this fork for standalone MCP use cases while contributing upstream to `openshift/lightspeed-service` for integrated improvements.

## What This Fork Can Offer

Areas where a standalone MCP server still has value vs. the integrated approach:

1. **Lightweight deployment** — No need to run the full OLS service stack
2. **Multi-client support** — Works with any MCP-compatible client (not just the OLS console)
3. **Development/testing** — Easier to iterate on MCP protocol features independently
4. **Edge/local use cases** — Can run locally against a remote OLS API
5. **Full API surface** — Can expose all OLS API fields (`provider`, `model`, `system_prompt`, `attachments`) as MCP tool parameters
