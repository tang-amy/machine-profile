---
name: "machine-profile"
description: "Set or recall which physical machine (computer) the user is working on this session. Invoked as \"/machine-profile\" plus a name (e.g. /machine-profile PC 2), \"/machine-profile list\", \"/machine-profile collect-from-linux|mac|windows\" (prints the collector script to run and paste back), \"/machine-profile update-packages ...\" (change which R packages the collectors check), or a name with --refresh. Machine profiles are stored in Claude's memory at /topics/machines.md, so any computer can recall any other. Loads that machine's specs (OS, chip, RAM, shell, R/Python/conda/Bioconductor, and the user's R package set with versions) so environment-specific help is correct, or gathers and saves them the first time. Works on macOS, Linux, and Windows."
---

# /machine-profile — machine profiles (stored in memory)

The user runs the same analyses on several computers and wants to recall any
machine's specs from any machine. Profiles are kept in Claude's own memory, not
on disk, so they are available from every session and device.

**Storage:** one memory file, `/topics/machines.md`, with a `## <Machine name>`
section per machine. Each section holds the spec block from the gather step. The
file may also hold a package-list block (see *update-packages*).

Always read the memory file (for its version token) before writing it, and use
`memory_str_replace` to update a single machine's section rather than rewriting
the whole file.

## Parse the argument

- `list` → list the machine names saved in `/topics/machines.md`. Nothing else.
- `collect-from-linux` / `collect-from-mac` (or `-macos`) / `collect-from-windows`
  → print the matching collector script for the user to run and paste back. Do
  not gather, save, or read memory. See *Emit a collector* below.
- `update-packages [...]` → view or change which R packages the collectors check.
  See *update-packages* below.
- a name + `--refresh` → re-gather that machine's specs and overwrite its section (overwrite is intended here — see the guard below).
- a bare name → load that section if it exists; gather + save if it doesn't.

## list

`memory_read /topics/machines.md` and report the `## ` machine-section headings.
If the file doesn't exist, say no machines are saved yet.

## update-packages

The R packages the collectors check are stored in `/topics/machines.md` between
`<!-- PKGS-START -->` and `<!-- PKGS-END -->` markers, as one space-separated
line. When that block is absent, the built-in default list in the collectors
(below) is used.

Interpret the argument:
- (no args) → show the current effective list (the saved block if present, else
  the default). Change nothing.
- `add <pkg...>` → add those names (seed the block from the current default if
  none exists yet), skipping duplicates.
- `remove <pkg...>` → drop those names from the saved list.
- `set <pkg...>`, or a bare list of names → replace the saved list entirely.
- `reset` → delete the block so the built-in default is used again.

Package names may be space- or comma-separated; normalize to space-separated.
After any change: read the file for its version token, then `memory_str_replace`
the block (or, if absent, insert it immediately after the frontmatter), confirm
in one line, and show the new list. Saved machine sections are snapshots — never
rewrite them here; a new list only affects future collections.

## Emit a collector (collect-from-<os>)

First `memory_read /topics/machines.md`; if a `<!-- PKGS-START -->` block exists,
replace the `PKGS=...` value in the script below with that saved list before
showing it (otherwise leave the script's default). Then reply with just the
matching collector inside a fenced code block, plus one line telling the user to
run it and paste the output back with a machine name (e.g. "save as PC 2").
`collect-from-linux` and `collect-from-mac` both use the **bash** collector — it
auto-detects macOS vs Linux. `collect-from-windows` uses the **PowerShell**
collector. Don't gather or save in this mode.

## Load an existing profile

`memory_read /topics/machines.md`, find the `## <name>` section, and treat it as
the active machine for the rest of the session: when giving R/Python/package or
path advice, assume these versions. Confirm in one line, e.g.
*"On PC 2 (M3 Max, Sonoma, R 4.4.1, Bioc 3.19, Seurat 5.1.0)."* Don't echo the
whole section unless asked.

## Overwrite guard (always check before saving)

Before writing specs for a name that **already has a section** in
`/topics/machines.md`, STOP and warn the user first: name the machine and the
`_updated_` date of the existing section, and ask them to confirm the overwrite.
Proceed only on an explicit yes. This applies to `--refresh` too (still confirm,
since it discards the old specs) and to any paste-and-save where the name exists.
Saving a brand-new name needs no warning.

## Gather + save (unknown machine, or confirmed overwrite)

Specs can only be read on the machine itself. Apply the saved package list the
same way as in *Emit a collector* (substitute it into `PKGS=` if the block
exists). If this session is linked to the user's computer, run the matching
collector with device bash / the shell; otherwise give the user the collector for
their OS (below) and ask them to paste the output back. The user may also
pre-collect and paste a spec block on their own — accept that the same way,
applying the overwrite guard. Missing R packages are reported as *not installed*
so a profile doubles as a per-machine checklist.

### macOS / Linux (bash)

```bash
PKGS="Seurat SeuratObject tidyverse tidyr dplyr stringr magrittr Matrix ggplot2 cowplot patchwork ggpubr ggsignif ggupset ggnewscale ggseqlogo RColorBrewer viridis VennDiagram pheatmap ComplexHeatmap EnhancedVolcano dittoSeq xtable readxl xlsx hdf5r sctransform glmGamPoi scDblFinder scProportionTest SingleR celldex monocle Signac scTCRvis SeuratDisk SeuratData presto clusterProfiler ReactomePA pathview org.Hs.eg.db org.Mm.eg.db AnnotationHub EnsDb.Mmusculus.v79 chromVAR motifmatchr JASPAR2020 TFBSTools BSgenome.Mmusculus.UCSC.mm10 devtools remotes"
have(){ command -v "$1" >/dev/null 2>&1; }
val(){ printf '%s' "${1:-not found}"; }
if [ "$(uname)" = "Darwin" ]; then
  ov="$(sw_vers -productVersion 2>/dev/null)"
  case "${ov%%.*}" in 15)c=Sequoia;;14)c=Sonoma;;13)c=Ventura;;12)c=Monterey;;11)c="Big Sur";;*)c=;;esac
  OS="$(sw_vers -productName 2>/dev/null) $c $ov"
  CHIP="$(sysctl -n machdep.cpu.brand_string 2>/dev/null) ($(sysctl -n hw.physicalcpu 2>/dev/null)-core)"
  gb=$(( $(sysctl -n hw.memsize 2>/dev/null) /1073741824 ))
  if sysctl -n hw.optional.arm64 2>/dev/null | grep -q 1; then RAM="${gb}GB Unified Memory"; else RAM="${gb}GB"; fi
else
  OS="$(lsb_release -ds 2>/dev/null || sed -n 's/^PRETTY_NAME="\(.*\)"/\1/p' /etc/os-release 2>/dev/null || uname -sr)"
  CHIP="$(grep -m1 'model name' /proc/cpuinfo|cut -d: -f2|sed 's/^ *//') ($(nproc)-core)"
  RAM="$(( $(grep -m1 MemTotal /proc/meminfo|awk '{print $2}') /1048576 ))GB"
fi
[ -d "$HOME/.oh-my-zsh" ] && SH="$(basename "${SHELL:-sh}") (Oh My Zsh)" || SH="$(basename "${SHELL:-sh}")"
have R && RV="$(R --version|head -1|sed -E 's/.*version ([0-9.]+).*/\1/')" || RV="not found"
have python3 && PY="$(python3 --version 2>&1|awk '{print $2}')" || PY="not found"
if have conda; then CONDA="$(conda --version 2>/dev/null)"
elif [ -n "${CONDA_EXE:-}" ]; then CONDA="$("$CONDA_EXE" --version 2>/dev/null)"
else for d in "$HOME/anaconda3" "$HOME/miniconda3" "$HOME/miniforge3" "/opt/anaconda3" "/opt/miniconda3"; do [ -x "$d/bin/conda" ] && { CONDA="$("$d/bin/conda" --version 2>/dev/null)"; break; }; done; fi
[ -z "${CONDA:-}" ] && CONDA="not found"
[ -n "${CONDA_DEFAULT_ENV:-}" ] && CONDA="$CONDA (env: $CONDA_DEFAULT_ENV)"
if have Rscript; then BIOC="$(Rscript -e 'v<-tryCatch(as.character(BiocManager::version()),error=function(e)NA);b<-tryCatch(as.character(packageVersion("BiocManager")),error=function(e)NA);if(is.na(v))cat("not found") else cat(sprintf("%s (BiocManager %s)",v,b))' 2>/dev/null)"; else BIOC="not found"; fi
[ -z "${BIOC:-}" ] && BIOC="not found"
PL="- _(R / Rscript not found)_"
if have Rscript; then
  out="$(PKGS="$PKGS" Rscript -e 'for(p in strsplit(Sys.getenv("PKGS"),"[[:space:]]+")[[1]]){if(!nzchar(p))next;v<-tryCatch(as.character(packageVersion(p)),error=function(e)NA);if(is.na(v))cat(sprintf("- %s: _not installed_\n",p)) else cat(sprintf("- **%s:** %s\n",p,v))}' 2>/dev/null)"
  [ -n "$out" ] && PL="$out"
fi
printf '**OS:** %s\n**CPU/Chip:** %s\n**RAM:** %s\n**Shell:** %s\n**R:** %s\n**Python:** %s\n**Conda:** %s\n**Bioconductor:** %s\n\n_R packages:_\n%s\n' \
  "$(val "$OS")" "$(val "$CHIP")" "$(val "$RAM")" "$SH" "$(val "$RV")" "$(val "$PY")" "$(val "$CONDA")" "$(val "$BIOC")" "$PL"
```

### Windows (PowerShell)

Native Windows only. Under WSL, use the bash collector instead (it is Linux).

```powershell
$PKGS = "Seurat SeuratObject tidyverse tidyr dplyr stringr magrittr Matrix ggplot2 cowplot patchwork ggpubr ggsignif ggupset ggnewscale ggseqlogo RColorBrewer viridis VennDiagram pheatmap ComplexHeatmap EnhancedVolcano dittoSeq xtable readxl xlsx hdf5r sctransform glmGamPoi scDblFinder scProportionTest SingleR celldex monocle Signac scTCRvis SeuratDisk SeuratData presto clusterProfiler ReactomePA pathview org.Hs.eg.db org.Mm.eg.db AnnotationHub EnsDb.Mmusculus.v79 chromVAR motifmatchr JASPAR2020 TFBSTools BSgenome.Mmusculus.UCSC.mm10 devtools remotes"
$os  = Get-CimInstance Win32_OperatingSystem
$cpu = Get-CimInstance Win32_Processor | Select-Object -First 1
$CHIP = "$($cpu.Name.Trim()) ($($cpu.NumberOfCores)-core)"
$gb  = [math]::Round((Get-CimInstance Win32_ComputerSystem).TotalPhysicalMemory/1GB)
$SH  = "PowerShell $($PSVersionTable.PSVersion)"
function Have($c){ [bool](Get-Command $c -ErrorAction SilentlyContinue) }
$RV = if (Have R) { ((& R --version 2>$null | Select-Object -First 1) -replace '.*version ([0-9.]+).*','$1') } else { "not found" }
$PY = if (Have python) { (& python --version 2>&1) -replace 'Python ','' } elseif (Have py) { (& py --version 2>&1) -replace 'Python ','' } else { "not found" }
$CONDA = if (Have conda) { (& conda --version 2>$null) }
         elseif ($env:CONDA_EXE) { (& $env:CONDA_EXE --version 2>$null) }
         else { $p = @("$env:USERPROFILE\anaconda3\Scripts\conda.exe","$env:USERPROFILE\miniconda3\Scripts\conda.exe","$env:USERPROFILE\miniforge3\Scripts\conda.exe") | Where-Object { Test-Path $_ } | Select-Object -First 1; if ($p) { (& $p --version 2>$null) } else { "not found" } }
if ($env:CONDA_DEFAULT_ENV) { $CONDA = "$CONDA (env: $($env:CONDA_DEFAULT_ENV))" }
$BIOC = if (Have Rscript) { (& Rscript -e 'v<-tryCatch(as.character(BiocManager::version()),error=function(e)NA);b<-tryCatch(as.character(packageVersion("BiocManager")),error=function(e)NA);if(is.na(v))cat("not found") else cat(sprintf("%s (BiocManager %s)",v,b))' 2>$null) } else { "not found" }
$PL = "- _(R / Rscript not found)_"
if (Have Rscript) {
  $env:PKGS = $PKGS
  $out = & Rscript -e 'for(p in strsplit(Sys.getenv("PKGS"),"[[:space:]]+")[[1]]){if(!nzchar(p))next;v<-tryCatch(as.character(packageVersion(p)),error=function(e)NA);if(is.na(v))cat(sprintf("- %s: _not installed_\n",p)) else cat(sprintf("- **%s:** %s\n",p,v))}' 2>$null
  if ($out) { $PL = ($out -join "`n") }
}
@"
**OS:** $($os.Caption) (build $($os.BuildNumber))
**CPU/Chip:** $CHIP
**RAM:** ${gb}GB
**Shell:** $SH
**R:** $RV
**Python:** $PY
**Conda:** $CONDA
**Bioconductor:** $BIOC

_R packages:_
$PL
"@
```

(On Windows R is often not on PATH, so R/Bioconductor/package lines may read "not found" even when installed.)

### Write it

Write the spec block into memory as a `## <name>` section in `/topics/machines.md`
(create the file if absent; else read it and `memory_str_replace` that section,
or append a new one). Prefix the section with an `_updated YYYY-MM-DD_` line.
Confirm in one line and set it as the active machine.

## Notes

- If R lives inside a conda env, activate it first so `Rscript` matches the R the user actually uses.
- Profiles are plain markdown in memory — ask Claude to tweak a line anytime, same as hand-editing.
- A machine's specs are benign config, fine to keep in memory.