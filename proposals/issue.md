Title: [Proposal] Add $(TargetFrameworkVersionMajor) MSBuild property

Body
----

Summary
-------

I'd like to propose adding a small, read-only MSBuild property called `$(TargetFrameworkVersionMajor)` that returns the major version number of the currently targeted framework. This will simplify conditional logic in `.props`/`.targets` for multi-targeted projects and improve readability.

Motivation
----------

Maintaining conditional logic across multiple target frameworks commonly requires only the major version number. Today this requires repetitive string parsing or helper properties in many locations. A dedicated property reduces duplication and improves clarity.

Proposal
--------

- Add a read-only property `TargetFrameworkVersionMajor` that is set during project evaluation based on `$(TargetFramework)`.
- For common SDK-style identifiers starting with `net`, parse the first digit sequence after `net` as the major version.
- Leave property empty if parsing fails.

Examples
--------

| $(TargetFramework) | $(TargetFrameworkVersionMajor) |
|--------------------:|:-------------------------------:|
| net462              | 4 |
| net48               | 4 |
| net5.0              | 5 |
| net6.0              | 6 |
| net8.0-windows      | 8 |
| net10.0             | 10 |

Usage example
-------------

```xml
<PropertyGroup>
  <_IsNet6OrHigher Condition="'$(TargetFrameworkVersionMajor)' != '' And $(TargetFrameworkVersionMajor) &gt;= 6">true</_IsNet6OrHigher>
</PropertyGroup>
```

Compatibility
-------------

This is an additive change. Projects that don't use it will be unaffected. For multi-targeting, each target's evaluation should expose the correct major version.

Open for discussion
-------------------

I'm open to suggestions on parsing edge-cases (e.g., `netcoreapp2.1`) and whether to include additional known monikers. My preference is to start with common SDK `net*` identifiers and expand if there's community demand.

Implementation notes
--------------------

MSBuild can compute the property during evaluation using `TargetFramework` and expose it as read-only. If desired, we can also provide a helper `.props` snippet so consumers can get equivalent behavior while the property is not yet available in their MSBuild version.

--
Thank you for considering this small proposal to improve multi-targeting ergonomics.
