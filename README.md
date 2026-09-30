# `--exclude-filter` deletes genuinely shipped packages when two projects resolve a shared dependency differently

Minimal reproduction for `dotnet-CycloneDX` **6.2.0** (latest release at the time of writing).

Passing **any** value to `--exclude-filter` — including a package name that does not exist anywhere in
the graph — removes seven real, shipped runtime packages from the BOM.

## Repro

Requires only public packages from nuget.org. No local feed.

```bash
dotnet tool install --global CycloneDX --version 6.2.0
dotnet restore Repro.slnx

# 1. Without the flag
dotnet-CycloneDX Repro.slnx -o out -fn base.json -F Json --disable-package-restore

# 2. With a filter naming a package that does not exist
dotnet-CycloneDX Repro.slnx -o out -fn ef.json -F Json --disable-package-restore -ef DoesNotExist
```

## Actual

| Run | Components |
|---|---|
| no filter | **37** |
| `-ef DoesNotExist` | **30** |

The second run prints:

```
The following orphaned packages have been removed:
  - Microsoft.IdentityModel.Abstractions@8.14.0
  - Microsoft.IdentityModel.JsonWebTokens@7.7.1
  - Microsoft.IdentityModel.Logging@7.7.1
  - Microsoft.IdentityModel.Protocols@7.7.1
  - Microsoft.IdentityModel.Protocols.OpenIdConnect@7.7.1
  - Microsoft.IdentityModel.Tokens@7.7.1
  - System.IdentityModel.Tokens.Jwt@7.7.1
Found 30 packages
```

Every one of those is a runtime dependency that `ProjB` genuinely ships. `DoesNotExist` matches nothing,
so the filter argument plays no part in which packages are removed.

## Expected

Either the seven packages are retained, or `--exclude-filter` removes only what it was asked to remove.

## Why the packages look orphaned

Both projects reference `Microsoft.Data.SqlClient` 6.1.6, whose nuspec declares

```xml
<dependency id="Microsoft.IdentityModel.Protocols.OpenIdConnect" version="7.7.1" />
```

i.e. the open range `[7.7.1, )`. The two projects resolve it differently, and both resolutions are correct
in their own context:

* **ProjA** also pins `Microsoft.IdentityModel.Protocols.OpenIdConnect` at **8.23.0**, so NuGet unifies the
  whole family upward and SqlClient's dependency resolves to **8.23.0** there.
* **ProjB** has no such pin, so the same dependency resolves to **7.7.1**.

`ProjectAssetsFileService` stores each dependency as the *range* rather than the resolved version, and the
range is then resolved per project. The merge keeps **one** entry per `(name, version)`, so only one
project's edge set survives — `ProjA` is processed first, and `ProjB`'s edges are discarded.

The 7.7.1 components remain in `components` but are now referenced by nothing. On its own that is
cosmetic. `--exclude-filter` additionally enables `ExcludeFilterHelper.RemoveOrphanedPackages`, which walks
reachability from the direct references and deletes everything it cannot reach — so the unreferenced
components are removed from the document entirely.

Swapping which project is processed first swaps which version line is deleted, which is what makes this
hard to notice: the BOM is well-formed and the job succeeds either way.

## Why it matters

Consumers of the BOM see `components`. A package deleted this way is invisible to vulnerability scanning,
even though the software genuinely ships it. In the real solution this was found in, the same pattern
removed **25** shipped packages across two released SBOMs, including an ASP.NET Core server's entire JWT
validation stack.

## Related

* [#977](https://github.com/CycloneDX/cyclonedx-dotnet/issues/977) — the same merge behaviour, reported as
  a stray orphaned component and closed as an inherent limitation of merging resolution contexts. That is
  correct as far as it goes; this report is about what `--exclude-filter` then does with the orphan.
* [#1126](https://github.com/CycloneDX/cyclonedx-dotnet/issues/1126) and
  [#1130](https://github.com/CycloneDX/cyclonedx-dotnet/issues/1130) — the same
  `RemoveOrphanedPackages` sweep deleting real components, reached through different causes of missing
  edges (unpopulated `IsDirectReference`, and RID-specific targets).
* [#1110](https://github.com/CycloneDX/cyclonedx-dotnet/issues/1110) /
  [#1113](https://github.com/CycloneDX/cyclonedx-dotnet/pull/1113) — the *throwing* form of a range with
  several satisfying versions. I built #1113 at `e640ee4` and ran it against the real solution: the four
  unreferenced components and the twelve deletions are unchanged. That PR handles a range whose satisfying
  version exists only in another project's assets; here the range resolves successfully in **both**
  projects, just to different versions, so that code path is not entered. Separate defect, not a gap in
  the PR.

## Environment

* `dotnet-CycloneDX` 6.2.0
* .NET SDK 10.0.401, `net10.0` target
* Windows 11; also expected on Linux, nothing here is platform specific
