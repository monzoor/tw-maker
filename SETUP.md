# Project Setup — One-Time Initialization

> **Use this file once to initialize a new project from scratch.**
> After setup is complete, this file is no longer needed for day-to-day development.
> All architecture rules and conventions live in `.github/copilot-instructions.md`.

---

## Machine Prerequisites

Before starting, ensure the following are installed on the machine:

```bash
# Bun (package manager + script runner — replaces npm/yarn/npx for this project)
curl -fsSL https://bun.sh/install | bash

# GitHub CLI (required for Copilot Pro+ GitHub Models API access)
brew install gh
gh auth login   # authenticate once — gh auth token is used in scripts

# Node.js 22+ (required by Lighthouse MCP)
brew install node   # or use fnm/nvm

# Verify
bun --version     # should be 1.x+
gh auth status    # should show authenticated
node --version    # should be v22+
```

---

## Bootstrap — Initialize the Project from Scratch

Run these commands in order. Do not skip any step.

```bash
# 1. Create a new directory and initialize the Next.js app inside it
mkdir <project-name> && cd <project-name>
bun create next-app@latest . --yes --src-dir

# Verify what was installed before writing any code:
cat node_modules/next/package.json | grep '"version"'   # confirm Next.js version
cat node_modules/react/package.json | grep '"version"'  # confirm React version
# Then check node_modules/next/dist/docs/ for the authoritative docs for that version

# 2. Install runtime dependencies
bun add @reduxjs/toolkit react-redux axios

# 3. Install dev dependencies
bun add -D @modelcontextprotocol/sdk ts-morph \
  figma-context-mcp mcp-design-system-extractor @danielsogl/lighthouse-mcp

# 4. Place .github/copilot-instructions.md in the project
mkdir -p .github
# (copy copilot-instructions.md into .github/copilot-instructions.md)

# 5. Create .vscode/mcp.json (full content below)
mkdir -p .vscode
```

### `.vscode/mcp.json` — full content

```json
{
  "servers": {
    "next-devtools": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "next-devtools-mcp@latest"]
    },
    "storybook": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@storybook/addon-mcp@latest"]
    },
    "tw-maker": {
      "type": "stdio",
      "command": "bun",
      "args": ["run", "mcp/index.ts"]
    },
    "figma": {
      "type": "stdio",
      "command": "node",
      "args": ["node_modules/.bin/figma-developer-mcp"],
      "env": { "FIGMA_API_KEY": "${input:figma-api-key}" },
      "sandboxEnabled": true,
      "sandbox": {
        "network": { "allowedDomains": ["api.figma.com"] }
      }
    },
    "design-system-extractor": {
      "type": "stdio",
      "command": "node",
      "args": ["node_modules/.bin/mcp-design-system-extractor"],
      "sandboxEnabled": true,
      "sandbox": {
        "network": { "allowedDomains": ["localhost"] }
      }
    },
    "lighthouse": {
      "type": "stdio",
      "command": "node",
      "args": ["node_modules/.bin/lighthouse-mcp-server"],
      "sandboxEnabled": true,
      "sandbox": {
        "filesystem": { "allowWrite": ["${workspaceFolder}"] }
      }
    }
  },
  "inputs": [
    {
      "type": "promptString",
      "id": "figma-api-key",
      "description": "Figma Personal Access Token",
      "password": true
    }
  ]
}
```

---

## Project Plan (Implementation Roadmap)

### Phase 1 — Token Schema & CSS Pipeline

1. Create `tokens/tokens.json` — single source of truth for all design tokens:
   - `colors`: brand, semantic (success/warning/error/info), neutral scale
   - `typography`: fontFamily, fontSize, fontWeight, lineHeight, letterSpacing
   - `spacing`: 0–96 scale
   - `borderRadius`
   - `shadows` (elevation levels)
   - `breakpoints` (screen sizes)
2. Create `scripts/generate-tokens-css.ts` — reads `tokens/tokens.json`, outputs `app/tokens.css` with a `@theme { ... }` block using Tailwind v4 CSS variable namespaces: `--color-*`, `--font-*`, `--text-*`, `--leading-*`, `--tracking-*`, `--spacing-*`, `--radius-*`, `--shadow-*`, `--breakpoint-*`
3. Update `app/globals.css` to `@import "./tokens.css"` and remove placeholder tokens
4. Add `"tokens:generate": "bun run scripts/generate-tokens-css.ts"` to `package.json`
5. Scaffold initial components following Atomic Design — create component files ONLY, no `.stories.tsx` yet:
   - **Atoms**: `Button`, `Badge`, `Input`, `Label`, `Icon`, `Avatar`, `Spinner`, `PriceTag`, `Rating`
   - **Molecules**: `FormField` (Label + Input), `SearchBar` (Input + Button), `CartItem` (Image + PriceTag)
   - **Organisms**: `Navbar` (Logo + Nav + Cart icon), `ProductCard` (Image + Badge + PriceTag + Button), `CartDrawer`
   - **Templates**: `StorefrontLayout`, `AuthLayout`, `AccountLayout`, `CheckoutLayout`

> ⚠️ Do NOT create `.stories.tsx` files in Phase 1. Stories are written in Phase 2 **after** Storybook is installed and confirmed running. Creating story files before Storybook is installed is a wasted step and may be written against the wrong API.

**Token workflow:** Edit `tokens/tokens.json` → run `bun run tokens:generate` → Tailwind picks up the new CSS variables automatically.

---

### Phase 2 — Storybook 8

6. Install Storybook: run `bunx storybook@latest init` and select `@storybook/nextjs` framework when prompted. **Verify Storybook starts (`bun run storybook`) before writing any story files.**
7. `.storybook/main.ts`: stories glob covers `components/atoms/**/*.stories.@(ts|tsx)`, `components/molecules/**/*.stories.@(ts|tsx)`, `components/organisms/**/*.stories.@(ts|tsx)`, `components/templates/**/*.stories.@(ts|tsx)`; addons: `@storybook/addon-essentials`, `@storybook/addon-a11y`, `@storybook/addon-themes`
8. `.storybook/preview.ts`: import `../app/globals.css` so tokens are available in stories
9. Now write `.stories.tsx` files for all atoms and molecules co-located with their component files. Only do this after steps 6–8 are complete and Storybook is confirmed running.
10. Add `"storybook"` and `"storybook:build"` scripts to `package.json`

---

### Phase 3 — MCP Servers

Next.js 16 has built-in MCP support at `/_next/mcp` and ships a ready-made `next-devtools-mcp` package for runtime introspection. We run **six MCP servers** side by side.

#### 3a — Next.js DevTools MCP (built-in, no code required)

11. Install `next-devtools-mcp` as a dev dependency: `bun add -D next-devtools-mcp`
12. Add it to `.vscode/mcp.json` (already done above).
13. Start `bun run dev` — `next-devtools-mcp` auto-connects to `/_next/mcp` on the running dev server.

#### 3b — Design System MCP (custom)

14. Create `mcp/index.ts` — entry point using `StdioServerTransport`, registers all custom tools
15. Implement tools in `mcp/tools/`:
    - **`list_components`** — walks `components/` recursively, returns file paths + component names grouped by atomic level
    - **`read_component`** — takes a file path, returns full source code + structured props summary extracted via `ts-morph`
    - **`write_story`** — takes a file path + story content string, writes (or overwrites) the `.stories.tsx` co-located with the component
    - **`get_design_tokens`** — reads and returns `tokens/tokens.json` so agents have full token context when generating stories
16. Add `"mcp": "bun run mcp/index.ts"` to `package.json`

#### 3c — Storybook MCP (official)

17. Already registered in `.vscode/mcp.json` — no additional installation needed, runs via `npx`.

#### 3d — Figma Context MCP

18. Already installed (`figma-context-mcp`) and registered in `.vscode/mcp.json`. Requires `FIGMA_API_KEY` — VS Code will prompt on first use.

#### 3e — Design System Extractor MCP

19. Already installed (`mcp-design-system-extractor`) and registered in `.vscode/mcp.json`.

#### 3f — Lighthouse MCP

20. Already installed (`@danielsogl/lighthouse-mcp`) and registered in `.vscode/mcp.json`.

---

## Verification Checklist

- [ ] `bun run tokens:generate` → creates `app/tokens.css` with `@theme` variables
- [ ] Edit `tokens/tokens.json` → run `bun run tokens:generate` → changes reflect in the app
- [ ] `bun run dev` → app starts without errors
- [ ] `bun run storybook` → Storybook opens, components render with token-based styles
- [ ] In VS Code Copilot: ask about app errors → `next-devtools` MCP `get_errors` is called
- [ ] In VS Code Copilot: ask to list components → `design-system` MCP `list_components` is called
- [ ] In VS Code Copilot: ask to write a story for Button → `design-system` MCP tools are called and `Button.stories.tsx` is written
