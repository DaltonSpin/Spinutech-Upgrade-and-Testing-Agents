# Sitefinity Upgrade Agents — Backlog for Spinutech Fork

**Repo:** `Spinutech/Sitefinity-Upgrade-and-Testing-Agents` (forked from `Sitefinity/Sitefinity-Upgrade-and-Testing-Agents`)

This document tracks additions and modifications that must be made to the Spinutech fork before the next AI-driven Sitefinity upgrade is performed. Items are scoped to gaps or project-layout assumptions that differ from the upstream repo's defaults — particularly for solutions that use a shared solution-root `packages\` folder with a legacy `packages.config`-style `.NET Framework` web project.

---

## Item 1 — Add a Post-Upgrade Review Agent

### Why this agent is needed

The existing AI-driven upgrade process rewrites `.csproj` files to update Sitefinity package versions. It does this correctly from a package-version standpoint, but it has no awareness of project-specific layout decisions — for example, where the `packages\` folder lives relative to the project file, or whether HintPath prefixes were customized before the upgrade ran.

The result is a class of silent breakage where the `.csproj` is syntactically valid and the upgrade tool reports success, but the project will not build because assembly references now resolve to paths that do not exist on disk. This is not caught until a developer attempts a full solution build or opens the project in Visual Studio and notices warning triangles on the References node.

A dedicated **post-upgrade review agent** should run automatically after the upgrade tool completes, before any changes are committed, and gate the commit on a clean review result.

---

### Checks to implement

#### Check 1 — NuGet HintPath prefix integrity (required)

**Background:** In solutions that follow the standard NuGet `packages.config` layout, the `packages\` folder lives at the solution root — one level above the Sitefinity web project subfolder. Every `HintPath` in the web project's `.csproj` must therefore use a `..\ ` prefix to navigate up to the solution root:

```xml
<!-- Correct -->
<HintPath>..\packages\SomePackage.1.0.0\lib\net472\SomePackage.dll</HintPath>

<!-- Broken — what the upgrade tool produces -->
<HintPath>packages\SomePackage.1.0.0\lib\net472\SomePackage.dll</HintPath>
```

The AI-assisted upgrade tool rewrites the `.csproj` to update package versions but does not preserve existing path prefixes. It anchors HintPaths relative to the project file without accounting for the `..\ ` that was already present, silently producing paths that resolve to a non-existent folder. The project loads without error in Visual Studio but all assembly references are broken and the solution will not build.

**Check to add:** After the upgrade tool runs, scan the Sitefinity web project's `.csproj` for any `<HintPath>` that begins with `packages\` (i.e., missing the `..\ ` prefix). If any are found, the agent should:

1. Report the count and list a sample of affected paths.
2. Apply the fix automatically: replace `<HintPath>packages\` with `<HintPath>..\packages\` across the file.
3. Re-verify the count drops to zero before allowing the commit to proceed.

Suggested validation script (PowerShell — set `$proj` to the relative path of the Sitefinity web project `.csproj`):

```powershell
$proj = "<SitefinityWebProject>\<SitefinityWebProject>.csproj"  # adjust per solution
$broken = (Select-String -Path $proj -Pattern "<HintPath>packages\\").Count
if ($broken -gt 0) {
    Write-Warning "$broken broken HintPath(s) detected — applying fix..."
    $content = Get-Content $proj -Raw
    $content = $content -replace '<HintPath>packages\\', '<HintPath>..\packages\'
    Set-Content $proj -Value $content -NoNewline
    $remaining = (Select-String -Path $proj -Pattern "<HintPath>packages\\").Count
    if ($remaining -gt 0) {
        Write-Error "Fix incomplete — $remaining path(s) still broken. Manual review required."
        exit 1
    }
    Write-Host "HintPath fix applied successfully."
}
```

---

### Additional checks to consider adding in future iterations

The following are not yet implemented but represent the same category of silent post-upgrade breakage worth guarding against:

- **`packages.config` / `.csproj` version sync** — Verify that every `<package>` entry in `packages.config` has a corresponding `<Reference>` HintPath in the `.csproj` at the same version, and vice versa.
- **New packages introduced by the upgrade** — Confirm any net-new packages added to `packages.config` actually exist in the `packages\` folder (i.e., NuGet restore ran successfully after the upgrade).
- **Binding redirect accuracy** — After a major version bump, `Web.config` `<bindingRedirects>` for Sitefinity assemblies should be reviewed to ensure they reflect the new version range.

---

*Last updated: 2026-06-18*
*Origin: Discovered during a Sitefinity 15.4 AI-assisted upgrade on a solution using a solution-root `packages\` folder layout.*
