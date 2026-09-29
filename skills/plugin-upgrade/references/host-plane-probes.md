# Host-plane dual-cohort probes (`!!js` in `cordis.patch.yml`)

Builds on [rollup R-02](rollup-0.1.2.md). Host-plane plugins (the TUI kind) do not change artifacts; instead they push the cohort difference into `!!js` probes at the composition layer. Three forms, taken from the real patch in [dsh-TUI #622](https://github.com/ccch1mneyyy/dsh-TUI/pull/622):

**Currency of the quoted examples.** The three forms below are quoted verbatim from dsh-TUI #622, written against the **`dsh-v0.1.1-rc.2 → dsh-v0.1.2-alpha.1`** edge (corridor row 2 in [references/README.md](README.md)), so their *package* coordinates age with the host. On 0.1.7 `@deepseek-ai/dsh-agent-presets` no longer exists — it was split into `dsh-agent-preset` and `dsh-agent-preset-registry` ([DSH-0.1.7-J1-16](jump-0.1.5-rc.2-to-0.1.7-rc.1.md)) — and presets stopped being per-directory YAML files, becoming declarations carried by a bundle's ordered patch list ([DSH-0.1.7-J1-03](jump-0.1.5-rc.2-to-0.1.7-rc.1.md)). Where resolution does fail, what a form returns is its own failure branch, not a signal: form 2 reaches `catch { return false }` and form 3 falls into its `roots` fallback. That is conditional — an installation that still carries the retired dependency on the resolution path (a stale copy left in a profile's `node_modules`, say) resolves instead and silently reports the other branch. The technique itself (the three forms, `createRequire(ctx.baseUrl)`, snapshot comparison, both disciplines) is version-agnostic; re-derive the target from the target tag's installed tree before reusing a form, and do not read a form's branch as evidence that the probe target is absent.

```yaml
# 1. Subpath that exists only on the new host: insert only if resolve succeeds, otherwise the whole row self-disables; also yield when the official row with the same id is enabled
- id: dsh-tui-subagent-model-selection-settings
  name: '@deepseek-ai/dsh-tool-subagent/model-selection-settings'
  disabled: !!js >-
    (() => {
      const require = process.getBuiltinModule('node:module').createRequire(ctx.baseUrl)
      try { require.resolve('@deepseek-ai/dsh-tool-subagent/model-selection-settings') } catch { return true }
      return [...ctx.loader.entries()].some(entry => entry.options.id === 'subagent-model-selection-settings' && !entry.disabled)
    })()

# 2. Capability taken over by a shipped preset: decide by reading a preset file that really exists in the target tag; the host row yields
- id: command-goal
  disabled: !!js >-
    (() => {
      const require = process.getBuiltinModule('node:module').createRequire(ctx.baseUrl)
      const fs = process.getBuiltinModule('node:fs')
      try {
        const preset = require.resolve('@deepseek-ai/dsh-agent-presets/presets/standard/agent.cordis.yml')
        return fs.readFileSync(preset, 'utf8').split(/\r?\n/u).some(line => line.trim() === '- id: command-goal')
      } catch { return false }
    })()

# 3. Config shape varies by cohort: probe the package directory to decide whether to provide roots (see DSH-0.1.2-A1-21)
- id: dsh-tui-agent-presets
  name: '@deepseek-ai/dsh-agent-presets'
  config: !!js >-
    (() => {
      const require = process.getBuiltinModule('node:module').createRequire(ctx.baseUrl)
      const fs = process.getBuiltinModule('node:fs'), path = process.getBuiltinModule('node:path')
      try {
        const manifest = require.resolve('@deepseek-ai/dsh-agent-presets/package.json')
        if (fs.existsSync(path.join(path.dirname(manifest), 'presets'))) return { default: 'standard' }
      } catch {}
      return { default: 'standard', roots: [{ path: /* the dsh CLI config/agent-presets directory in rc.2 */ '', trust: 'system' }] }
    })()
```
Two disciplines:

1. The probe target must be something that really exists in the target tag — a `require.resolve` subpath, a file inside the package, or a package directory. Using another loader row as a "capability marker" silently stops working the moment that row changes (the first dsh-TUI version used `plugin-package-inventory-deepseek` to stand for alpha and was changed after review).
2. Probe results go into snapshots: evaluate the `!!js` expression on each of the rc.2 and alpha baselines and compare (dsh-TUI's `verify:patch-surface` evaluates with `evaluate` from `@deepseek-ai/cordis-plugin-loader` against `baseUrl`; the snapshot records the effective state, not the raw YAML).

