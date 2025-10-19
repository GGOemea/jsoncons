# Proposal: Add $(TargetFrameworkVersionMajor) MSBuild property

Summary
-------

Introduce a new read-only MSBuild property, `$(TargetFrameworkVersionMajor)`, that returns the major version number of the currently targeted framework.

Rationale
---------

Developers frequently use `$(TargetFramework)` (for example `net48`, `net6.0`, `net8.0-windows`) when writing conditional MSBuild logic. Many conditional checks, however, only need the major version (e.g., 4, 6, 8). Extracting the major version today requires verbose string operations and duplicated logic across multiple `.props`/`.targets` files.

This proposal adds a small, well-defined property that improves readability, reduces duplication, and standardizes a common pattern.

Specification
-------------

Name: `TargetFrameworkVersionMajor`
Scope: read-only, automatically populated by MSBuild during project evaluation when `TargetFramework` or `TargetFrameworks` is set.
Type: string containing a decimal integer (e.g. `4`, `6`, `8`, `10`).

Behavior
--------

- If `TargetFramework` is set to a known SDK-style TF identifier, extract its numeric major version. Examples:
  - `net462` -> `4`
  - `net48` -> `4`
  - `net5.0` -> `5` (note: historically `net5.0` maps to 5)
  - `net6.0` -> `6`
  - `net8.0-windows` -> `8`
  - `net10.0` -> `10`
- For multi-targeting (`TargetFrameworks`) MSBuild should expose the property for each evaluated target (i.e., each target evaluation gets its own value).
- If the framework cannot be parsed, the property should be empty.

Parsing details
---------------

The parsing algorithm should be conservative and consistent with SDK usage:

1. For `TargetFramework` values that begin with `net` followed by a digits sequence (optionally with a dot and more digits), parse the first integer after `net` as the major version.
   - `net462` -> parse leading digit(s) `4` -> `4`
   - `net48` -> `4`
   - `net5.0` -> `5`
   - `net6.0` -> `6`
2. For other framework strings (for example, `netcoreapp2.1` or `monoandroid10.0`), implement best-effort parsing or leave empty; prefer explicit mapping in MSBuild if needed.

Compatibility and migration
---------------------------

This property is strictly additive and read-only. Existing build logic is unchanged. Projects can adopt the new property to simplify conditional logic gradually.

Examples
--------

Simplified conditional checks:

```xml
<PropertyGroup>
  <_IsNet8 Condition="'$(TargetFrameworkVersionMajor)' == '8'">true</_IsNet8>
</PropertyGroup>

<Target Name="ValidateRefs" BeforeTargets="PrepareForBuild">
  <Error Condition="'$(TargetFrameworkVersionMajor)' == '8' And '$(REF_PATH_NET8)' == ''"
         Text="Reference path for .NET 8 is not set." />
</Target>
```

Recommended MSBuild snippet (fallback approach for users who can't rely on built-in property yet):

```xml
<!-- Minimal end-user fallback: define TargetFrameworkVersionMajor if not provided by the SDK -->
<PropertyGroup>
  <TargetFrameworkVersionMajor Condition="'$(TargetFrameworkVersionMajor)' == ''">
    $([System.Text.RegularExpressions.Regex]::Match('$(TargetFramework)','^net(\d+)').Groups[1].Value)
  </TargetFrameworkVersionMajor>
</PropertyGroup>
```

Open questions
--------------

- Should MSBuild support parsing for all historical framework monikers (`netcoreapp`, `netstandard`, `monoandroid`, etc.) or only the `net*` SDK-style identifiers? My recommendation is to start conservative and extend as needed.

Implementation note
-------------------

The implementation inside MSBuild should be lightweight: compute this property during project evaluation using the already-available `TargetFramework` value and expose it as read-only. The property should be available to `.props` and `.targets` evaluated after `TargetFramework` is set.

Acceptance criteria
-------------------

- New property exists and returns the major version for common `net*` TFMs.
- It is reproducible for single-target and multi-target evaluations.
- Existing projects remain unaffected.

Acknowledgements
----------------

Thanks to the many maintainers who manage multi-targeting complexities in SDK/MSBuild. This small feature aims to reduce repetitive conditional parsing across the ecosystem.
