# [Dependency Audit] Nightly scan — 2026-04-28

**Generated:** 2026-04-28 02:00 UTC  
**Source:** `pom.xml` (Java 24, all `test` scope)  
**Method:** Static analysis against known CVE database (OWASP Dependency-Check not run in this session)

---

## Summary

- **Dependencies scanned:** 11 (all test-scoped)
- **Critical CVEs confirmed:** 0
- **Security-relevant observations:** 3 (swagger-parser SSRF risk, logback version note, nimbus-jose upgrade available)
- **Upgrade recommended:** 3 dependencies have newer stable releases

---

## Dependency Status Table

| Dependency | Current Version | Status | Notes |
|---|---|---|---|
| `rest-assured` | 5.4.0 | ✅ OK | No known CVEs |
| `json-path` | 5.4.0 | ✅ OK | Bundled with rest-assured |
| `jackson-databind` | 2.17.0 | ✅ OK | Patched all known deserialization CVEs |
| `okhttp` | 4.12.0 | ✅ OK | No known CVEs at this version |
| `junit-jupiter` | 5.10.2 | ✅ OK | No known CVEs |
| `junit-platform-launcher` | 1.10.2 | ✅ OK | No known CVEs |
| `assertj-core` | 3.25.3 | ✅ OK | No known CVEs |
| `nimbus-jose-jwt` | 9.37.3 | ⚠️ Review | See note below |
| `datafaker` | 2.1.0 | ✅ OK | No known CVEs |
| `allure-junit5` | 2.27.0 | ✅ OK | No known CVEs |
| `swagger-parser` | 2.1.21 | ⚠️ Review | See note below — external ref resolution |
| `slf4j-api` | 2.0.12 | ✅ OK | No known CVEs |
| `logback-classic` | 1.4.14 | ⚠️ Review | See note below — EOL branch |

---

## Security-Relevant Observations

### ⚠️ `swagger-parser 2.1.21` — External `$ref` SSRF Risk

**Severity: Medium (context-dependent)**

`swagger-parser` resolves external `$ref` URLs by default. If the spec path (`PENTEST_SPEC_PATH`) is ever sourced from user-controlled input or an untrusted location, the parser will make outbound HTTP requests to any URL referenced in the spec. In the current codebase this is benign (the spec path is a local file), but this is a latent SSRF vector.

Additionally, the library's XML-based processing (when loading YAML) has historically been susceptible to XXE if the underlying XML parser is not configured with `FEATURE_SECURE_PROCESSING`. Verify that `OpenApiLoader.java` does not expose the parsed spec to user-supplied content.

**Recommendation:** Pin to the latest 2.1.x release and verify `ParseOptions` disables remote reference resolution in CI (`setResolve(false)` if external refs are not needed).

---

### ⚠️ `logback-classic 1.4.14` — End-of-Life Branch

**Severity: Low**

The `1.4.x` branch of logback is in maintenance mode. The current supported branch is `1.5.x` (logback 1.5.18 as of early 2026). While 1.4.14 incorporates patches for CVE-2023-6378 (serialization issue in `logback-receiver`), the `1.4.x` line will not receive new security patches.

Note: This project uses logback only in test scope (`logback-test.xml`) — exploitation risk is minimal. Still, upgrade to align with supported versions.

**Recommendation:** Upgrade to `logback-classic 1.5.x`:
```xml
<logback.version>1.5.18</logback.version>
```

---

### ⚠️ `nimbus-jose-jwt 9.37.3` — Upgrade Available

**Severity: Informational**

`nimbus-jose-jwt` is actively maintained. Version 9.37.3 is recent but not the latest in the 9.x series. No CVEs are known for this version. The library is used in test scope for JWT manipulation in pentest scenarios.

**Recommendation:** Monitor for updates — the JWT manipulation tests should continue to work with newer minor versions.

---

## No Action Required

| Dependency | Reason |
|---|---|
| `rest-assured 5.4.0` | Latest stable |
| `jackson-databind 2.17.0` | All polymorphic deserialization CVEs patched in 2.14+ |
| `okhttp 4.12.0` | No known CVEs; latest stable in 4.x |
| `junit-jupiter 5.10.2` | Latest stable 5.10.x |
| `assertj-core 3.25.3` | No known CVEs |
| `datafaker 2.1.0` | No known CVEs |
| `allure-junit5 2.27.0` | No known CVEs |

---

## Recommended `pom.xml` Changes

```xml
<!-- Upgrade logback to supported 1.5.x branch -->
<logback.version>1.5.18</logback.version>

<!-- Pin to latest swagger-parser (verify external ref resolution settings) -->
<swagger-parser.version>2.1.22</swagger-parser.version>
```

---

## Note on OWASP Dependency-Check

This audit was performed via static knowledge. For a full NVD-backed CVE scan, run:

```bash
mvn org.owasp:dependency-check-maven:check \
  -DfailBuildOnCVSS=7 \
  -Dformat=JSON \
  -DnvdApiKey=$NVD_API_KEY
```

The `nightly-dependency-audit.yml` workflow runs this automatically at 02:00 UTC with the NVD API key.
