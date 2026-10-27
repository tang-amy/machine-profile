# machine-profile

A Claude skill for tracking the specs of every computer you run analyses on, and
recalling any of them from any machine.

Each profile captures the OS, chip, RAM, shell, R / Python / conda / Bioconductor
versions, and the versions of the R packages you care about (anything not
installed is flagged). Profiles are stored in Claude's memory, so they follow you
across sessions and devices — no synced folder needed. Built for a single-cell /
Bioconductor R workflow, but the package list is editable.

## Install

Drop the `machine-profile/` folder into your Claude skills directory:

```
~/.claude/skills/machine-profile/SKILL.md
```

Claude loads it automatically. Invoke it as `/machine-profile`.

Only `SKILL.md` is required. `examples/` and this README are for reference.

## Commands

| Command | What it does |
|---|---|
| `/machine-profile <name>` | Load that machine's saved profile. If it doesn't exist yet, collect and save it. |
| `/machine-profile list` | List every saved machine. |
| `/machine-profile <name> --refresh` | Re-collect and overwrite a machine's profile (asks first). |
| `/machine-profile collect-from-linux` | Print the bash collector to run and paste back. |
| `/machine-profile collect-from-mac` | Same bash collector (auto-detects macOS). |
| `/machine-profile collect-from-windows` | Print the PowerShell collector. |
| `/machine-profile update-packages` | Show the current R package list. |
| `/machine-profile update-packages add <pkgs>` | Add packages to the list. |
| `/machine-profile update-packages remove <pkgs>` | Remove packages. |
| `/machine-profile update-packages set <pkgs>` | Replace the whole list (a bare list does the same). |
| `/machine-profile update-packages reset` | Revert to the built-in default list. |

## Typical usage

**First time on a machine** — collect its specs:

```
/machine-profile collect-from-linux
```

Run the printed script in your terminal, copy the output, and paste it back with
a name:

```
save as Lab-Linux
<paste output>
```

**Later, from any computer** — recall it:

```
/machine-profile Lab-Linux
```

Claude then assumes that machine's R / package versions when giving advice, so
code and install suggestions match the environment you're actually on.

If your session is linked to the computer (device bridge), Claude can run the
collector for you — no copy-paste needed.

## Collecting specs yourself

The collectors are plain scripts; you can run them without Claude. `bash` for
macOS / Linux, PowerShell for Windows. Each prints a markdown spec block you paste
back. Edit the `PKGS=` line to change which R packages it checks, or use
`update-packages` so every future collector uses your list.

Notes:
- If R lives inside a conda env, activate it first (`conda activate <env>`) so
  `Rscript` matches the R you actually use.
- On Windows, R is often not on `PATH`, so R / Bioconductor / package lines may
  read "not found" even when installed.
- Under WSL, use the bash collector — it's Linux.

## Storage

Everything lives in one memory file, `/topics/machines.md`:
- A `## <Machine name>` section per machine (plain markdown — editable by asking
  Claude, or by hand if you keep a copy).
- An optional `<!-- PKGS-START --> ... <!-- PKGS-END -->` block holding your
  custom package list.

See [`examples/machines.md`](examples/machines.md) for a complete example,
including the `Lab-Linux` profile.

## License

Do whatever you like with it.
