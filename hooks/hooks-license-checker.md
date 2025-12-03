# hooks-license-checker

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-hooks-license-checker`

## Overview

Scans dependencies for license compatibility issues in real-time. Flags problematic licenses (GPL in proprietary code, unknown licenses) before they're committed, preventing legal compliance issues.

### Value Proposition

| Without | With |
|---------|------|
| License issues found late in legal review | Caught at addition time |
| Accidental GPL contamination | Blocked proactively |
| Unknown license risks | Clear visibility |
| Manual license audits | Automated checking |

---

## Contract

### Hook Configuration

```yaml
hooks:
  - module: hooks-license-checker
    config:
      # License policy
      policy:
        allowed:                        # Explicitly allowed licenses
          - MIT
          - Apache-2.0
          - BSD-2-Clause
          - BSD-3-Clause
          - ISC
          - 0BSD
          - Unlicense
          - CC0-1.0

        restricted:                     # Require approval
          - LGPL-2.1
          - LGPL-3.0
          - MPL-2.0
          - EPL-1.0
          - EPL-2.0

        prohibited:                     # Always block
          - GPL-2.0
          - GPL-3.0
          - AGPL-3.0
          - SSPL-1.0
          - CC-BY-NC-*                  # Non-commercial
          - BUSL-*                      # Business source

        unknown_action: warn            # allow | warn | deny

      # Scanning scope
      scope:
        package_managers:
          - npm
          - pip
          - cargo
          - go
          - maven
          - nuget
        scan_transitive: true           # Check transitive deps
        scan_dev_deps: false            # Skip devDependencies

      # Actions
      actions:
        on_prohibited: deny             # deny | warn
        on_restricted: ask_user         # ask_user | warn
        on_unknown: warn                # warn | deny

      # Exceptions
      exceptions:
        packages:                       # Known OK packages
          - package: "some-gpl-tool"
            reason: "CLI tool, not linked"
        paths:                          # Paths to skip
          - "test/**"
          - "scripts/**"
```

### HookResult

```python
# Prohibited license detected
HookResult(
    action="deny",
    reason="License compliance violation",
    context_injection="""
⛔ **License Violation Detected**

Adding dependency `copyleft-lib@2.0.0` would introduce **GPL-3.0** license.

**Why this is blocked:**
GPL-3.0 is a copyleft license that requires derivative works to also be GPL-licensed.
Your project uses MIT license, which is incompatible.

**Options:**
1. Find an alternative package with permissive license
2. Request legal exception (requires approval)
3. Change project license to GPL-compatible

**Alternatives with similar functionality:**
- `permissive-lib` (MIT) - https://github.com/example/permissive-lib
- `free-lib` (Apache-2.0) - https://github.com/example/free-lib
""",
    user_message="⛔ Blocked: GPL-3.0 license not allowed",
    user_message_level="error"
)

# Restricted license - ask user
HookResult(
    action="ask_user",
    approval_prompt="""
⚠️ **License Requires Approval**

Dependency `lgpl-lib@1.0.0` uses **LGPL-3.0** license.

LGPL is allowed for dynamic linking but requires:
- Providing license notice
- Allowing users to replace the library
- Not modifying the library source

Is this dependency used appropriately?
""",
    approval_options=["Approve (dynamic linking)", "Reject", "Need more info"],
    approval_default="reject"
)
```

---

## Architecture

```python
class LicenseCheckerHook:
    """Check dependencies for license compliance."""

    def __init__(self, config: LicenseCheckerConfig):
        self.config = config
        self.policy = LicensePolicy(config.policy)
        self.scanners = self._initialize_scanners()
        self.license_db = LicenseDatabase()

    async def __call__(self, event: str, data: dict) -> HookResult:
        # Check on dependency file changes
        if event != "tool:post" or not self._is_dep_file_change(data):
            return HookResult(action="continue")

        file_path = data.get("file_path", "")
        content = data.get("content", "")

        # Detect package manager
        pkg_manager = self._detect_package_manager(file_path)
        if not pkg_manager:
            return HookResult(action="continue")

        # Parse dependencies
        deps = await self._parse_dependencies(pkg_manager, content)

        # Check each dependency
        violations = []
        warnings = []
        approvals_needed = []

        for dep in deps:
            result = await self._check_dependency(dep, pkg_manager)

            if result.status == "prohibited":
                violations.append(result)
            elif result.status == "restricted":
                approvals_needed.append(result)
            elif result.status == "unknown":
                if self.config.policy.unknown_action == "deny":
                    violations.append(result)
                else:
                    warnings.append(result)

        # Handle results
        if violations:
            return self._deny_violations(violations)
        elif approvals_needed:
            return self._request_approval(approvals_needed)
        elif warnings:
            return self._warn_user(warnings)

        return HookResult(action="continue")

    def _is_dep_file_change(self, data: dict) -> bool:
        """Check if this is a dependency file change."""
        file_path = data.get("file_path", "")
        dep_files = [
            "package.json", "package-lock.json",
            "requirements.txt", "Pipfile", "pyproject.toml",
            "Cargo.toml", "Cargo.lock",
            "go.mod", "go.sum",
            "pom.xml", "build.gradle",
            "*.csproj", "packages.config"
        ]
        return any(
            file_path.endswith(f) or fnmatch.fnmatch(file_path, f)
            for f in dep_files
        )

    async def _check_dependency(
        self,
        dep: Dependency,
        pkg_manager: str
    ) -> LicenseCheckResult:
        """Check a single dependency's license."""

        # Check exceptions first
        if self._is_excepted(dep):
            return LicenseCheckResult(
                dependency=dep,
                status="allowed",
                reason="Exception configured"
            )

        # Get license info
        license_info = await self._get_license(dep, pkg_manager)

        if not license_info or license_info.license == "UNKNOWN":
            return LicenseCheckResult(
                dependency=dep,
                status="unknown",
                license=None,
                reason="Could not determine license"
            )

        # Check against policy
        license_id = self._normalize_license(license_info.license)

        if self.policy.is_prohibited(license_id):
            return LicenseCheckResult(
                dependency=dep,
                status="prohibited",
                license=license_id,
                reason=f"{license_id} is prohibited by policy"
            )

        if self.policy.is_restricted(license_id):
            return LicenseCheckResult(
                dependency=dep,
                status="restricted",
                license=license_id,
                reason=f"{license_id} requires approval"
            )

        if self.policy.is_allowed(license_id):
            return LicenseCheckResult(
                dependency=dep,
                status="allowed",
                license=license_id
            )

        # Not explicitly listed
        return LicenseCheckResult(
            dependency=dep,
            status="unknown",
            license=license_id,
            reason=f"License {license_id} not in policy"
        )

    async def _get_license(
        self,
        dep: Dependency,
        pkg_manager: str
    ) -> LicenseInfo | None:
        """Get license information for dependency."""
        scanner = self.scanners.get(pkg_manager)
        if not scanner:
            return None

        # Check cache first
        cache_key = f"{pkg_manager}:{dep.name}:{dep.version}"
        cached = self.license_db.get(cache_key)
        if cached:
            return cached

        # Fetch from registry
        license_info = await scanner.get_license(dep)

        # Cache result
        if license_info:
            self.license_db.set(cache_key, license_info)

        return license_info


class LicensePolicy:
    """Evaluate licenses against policy."""

    def __init__(self, config: dict):
        self.allowed = set(config.get("allowed", []))
        self.restricted = set(config.get("restricted", []))
        self.prohibited = set(config.get("prohibited", []))

    def is_allowed(self, license_id: str) -> bool:
        """Check if license is explicitly allowed."""
        return self._matches(license_id, self.allowed)

    def is_restricted(self, license_id: str) -> bool:
        """Check if license requires approval."""
        return self._matches(license_id, self.restricted)

    def is_prohibited(self, license_id: str) -> bool:
        """Check if license is prohibited."""
        return self._matches(license_id, self.prohibited)

    def _matches(self, license_id: str, patterns: set) -> bool:
        """Check if license matches any pattern."""
        for pattern in patterns:
            if pattern.endswith("*"):
                if license_id.startswith(pattern[:-1]):
                    return True
            elif license_id == pattern:
                return True
        return False


class NpmLicenseScanner:
    """Scan npm packages for licenses."""

    async def get_license(self, dep: Dependency) -> LicenseInfo | None:
        """Get license from npm registry."""
        url = f"https://registry.npmjs.org/{dep.name}/{dep.version}"

        async with aiohttp.ClientSession() as session:
            async with session.get(url) as resp:
                if resp.status != 200:
                    return None
                data = await resp.json()

        license_field = data.get("license")
        if isinstance(license_field, dict):
            license_field = license_field.get("type")

        return LicenseInfo(
            package=dep.name,
            version=dep.version,
            license=license_field or "UNKNOWN",
            source="npm-registry"
        )

    async def parse_dependencies(self, content: str) -> list[Dependency]:
        """Parse package.json for dependencies."""
        data = json.loads(content)
        deps = []

        for dep_type in ["dependencies", "devDependencies", "peerDependencies"]:
            if dep_type in data:
                for name, version in data[dep_type].items():
                    deps.append(Dependency(
                        name=name,
                        version=self._resolve_version(version),
                        dep_type=dep_type
                    ))

        return deps


class PipLicenseScanner:
    """Scan Python packages for licenses."""

    async def get_license(self, dep: Dependency) -> LicenseInfo | None:
        """Get license from PyPI."""
        url = f"https://pypi.org/pypi/{dep.name}/{dep.version}/json"

        async with aiohttp.ClientSession() as session:
            async with session.get(url) as resp:
                if resp.status != 200:
                    return None
                data = await resp.json()

        info = data.get("info", {})
        license_field = info.get("license") or self._extract_from_classifiers(
            info.get("classifiers", [])
        )

        return LicenseInfo(
            package=dep.name,
            version=dep.version,
            license=license_field or "UNKNOWN",
            source="pypi"
        )

    def _extract_from_classifiers(self, classifiers: list[str]) -> str | None:
        """Extract license from classifiers."""
        for classifier in classifiers:
            if classifier.startswith("License :: OSI Approved :: "):
                return classifier.split("::")[-1].strip()
        return None
```

---

## License Categories

### Permissive (Usually Safe)

| License | SPDX ID | Notes |
|---------|---------|-------|
| MIT | MIT | Most permissive |
| Apache 2.0 | Apache-2.0 | Patent grant |
| BSD 2-Clause | BSD-2-Clause | Minimal restrictions |
| BSD 3-Clause | BSD-3-Clause | No endorsement clause |
| ISC | ISC | Simplified MIT |

### Weak Copyleft (Caution)

| License | SPDX ID | Notes |
|---------|---------|-------|
| LGPL 2.1 | LGPL-2.1 | OK if dynamically linked |
| LGPL 3.0 | LGPL-3.0 | OK if dynamically linked |
| MPL 2.0 | MPL-2.0 | File-level copyleft |
| EPL 2.0 | EPL-2.0 | Module-level copyleft |

### Strong Copyleft (Usually Blocked)

| License | SPDX ID | Notes |
|---------|---------|-------|
| GPL 2.0 | GPL-2.0 | Viral copyleft |
| GPL 3.0 | GPL-3.0 | Viral copyleft |
| AGPL 3.0 | AGPL-3.0 | Network copyleft |

### Problematic

| License | SPDX ID | Notes |
|---------|---------|-------|
| SSPL | SSPL-1.0 | MongoDB license |
| BSL | BUSL-* | Time-delayed open source |
| CC BY-NC | CC-BY-NC-* | Non-commercial only |
| Proprietary | - | Requires explicit grant |

---

## Examples

### Example 1: GPL Dependency Blocked

```python
# User adds axios-gpl (fictional) to package.json

# Hook response:
HookResult(
    action="deny",
    reason="GPL-3.0 license prohibited",
    context_injection="""
⛔ **License Violation: GPL-3.0**

Package: `axios-gpl@1.0.0`
License: GPL-3.0

**Why blocked:**
GPL-3.0 requires any derivative work to also be licensed under GPL.
Your project (MIT) cannot include GPL dependencies.

**Alternatives:**
- `axios` (MIT) - Same functionality, permissive license
- `node-fetch` (MIT) - Lighter alternative
"""
)
```

### Example 2: LGPL Requires Approval

```python
# User adds chart.js which uses LGPL dependency

# Hook asks:
HookResult(
    action="ask_user",
    approval_prompt="""
⚠️ **LGPL Dependency Detected**

`chart.js@4.0.0` → `color@4.0.0` (LGPL-2.1)

LGPL is conditionally acceptable if:
✓ Used as dynamic library (not modified)
✓ License notice included
✓ Users can replace the library

Is this usage compliant?
""",
    approval_options=["Approve", "Reject", "Need legal review"]
)
```

### Example 3: Unknown License Warning

```python
# Package has no license field

# Hook warns:
HookResult(
    action="inject_context",
    context_injection="""
⚠️ **Unknown License Warning**

Package: `mystery-lib@0.5.0`
License: UNKNOWN

Could not determine license from:
- package.json
- LICENSE file
- npm registry

**Risk:** Without explicit license, code is "all rights reserved" by default.

**Recommendations:**
1. Check package repository for LICENSE file
2. Contact maintainer to add license
3. Find alternative with clear license
""",
    user_message="⚠️ Warning: mystery-lib has no license",
    user_message_level="warning"
)
```

---

## Transitive Dependency Scanning

```python
async def scan_transitive(
    self,
    direct_deps: list[Dependency],
    pkg_manager: str
) -> list[TransitiveDep]:
    """Scan transitive dependencies."""

    if pkg_manager == "npm":
        # Use npm ls --json for full tree
        result = await asyncio.create_subprocess_exec(
            "npm", "ls", "--json", "--all",
            stdout=asyncio.subprocess.PIPE
        )
        stdout, _ = await result.communicate()
        tree = json.loads(stdout.decode())
        return self._parse_npm_tree(tree)

    elif pkg_manager == "pip":
        # Use pipdeptree
        result = await asyncio.create_subprocess_exec(
            "pipdeptree", "--json",
            stdout=asyncio.subprocess.PIPE
        )
        stdout, _ = await result.communicate()
        return self._parse_pip_tree(json.loads(stdout.decode()))

    # ... other package managers
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `policy.allowed` | list | [] | Allowed licenses |
| `policy.restricted` | list | [] | Licenses needing approval |
| `policy.prohibited` | list | [] | Blocked licenses |
| `policy.unknown_action` | string | "warn" | Action for unknown |
| `scope.scan_transitive` | bool | true | Check transitive deps |
| `scope.scan_dev_deps` | bool | false | Include devDependencies |
| `actions.on_prohibited` | string | "deny" | Action on prohibited |

---

## Security Considerations

- Makes network calls to package registries
- Caches license data locally
- No sensitive data exposed
- Offline mode available with pre-populated cache

---

## Dependencies

### Required
- `aiohttp` - HTTP client
- `spdx-tools` - License parsing (optional)

### Optional
- Package manager CLIs for transitive scanning

---

## Open Questions

1. **SBOM generation**: Should we generate Software Bill of Materials?
2. **License changes**: Alert when dependency changes license?
3. **Legal integration**: Connect to legal approval workflow?
4. **Dual licensing**: How to handle dual-licensed packages?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
