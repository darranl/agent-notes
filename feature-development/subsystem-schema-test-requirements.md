# WildFly Subsystem Schema Test Requirements

## Overview

WildFly subsystems use schema-based testing to verify that XML configuration can be correctly parsed and marshalled. Each schema version requires a corresponding test XML file that provides comprehensive coverage of all schema elements, attributes, and complex types.

## Test File Location and Naming

Test XML files are located in:
```
src/test/resources/org/wildfly/extension/{subsystem-name}/
```

Naming convention:
- `{subsystem-name}-{version}.xml` - For stable versions (e.g., `elytron-oidc-client-1.0.xml`)
- `{subsystem-name}-preview-{version}.xml` - For preview versions (e.g., `elytron-oidc-client-preview-2.0.xml`)

## Test Requirements

### 1. Complete Schema Coverage

Every test XML file MUST include:

- **All elements** defined in the corresponding XSD schema
- **All attributes** for each element (including optional ones)
- **All complex types** with representative examples
- **Multiple instances** of repeatable elements where applicable
- **Both literal and expression forms** for every attribute that supports expression substitution (see Section 1a)

**Coverage Checklist:**
- [ ] All top-level elements (e.g., `realm`, `provider`, `secure-deployment`, `secure-server`)
- [ ] All child elements within each complex type
- [ ] All optional elements (at least one instance)
- [ ] All attributes (including optional ones like `algorithm` on credentials)
- [ ] Multiple instances of elements that can appear multiple times
- [ ] Each attribute appears at least once as a plain literal value
- [ ] Each attribute appears at least once as a `${property:default}` expression

### 1a. Expression Coverage Requirements

WildFly subsystem tests verify that both literal values and expression-substituted values can be parsed and round-tripped through the management model. **For every attribute that supports expressions** (defined with `.setAllowExpression(true)` in the Java attribute builder), the test XML must include at least one literal instance and at least one `${...}` expression instance somewhere across all entries of that complex type.

**Recommended pattern**: Maintain one "full literal" entry and one "full expression" entry for each complex type, rather than spreading coverage thinly across many sparse entries. For example:
- A `google` provider entry that includes every possible provider attribute as a literal value
- A `redhat-sso-expressions` provider entry that includes every possible provider attribute as an expression

This makes it easy to verify coverage at a glance and ensures new attributes added to the schema are added to both entries.

**Resource path key attributes are exempt**: The `name` attribute on elements like `credential` and `redirect-rewrite-rule` is the WildFly management address path key, NOT a model attribute. It is registered via `PathElement.pathElement(...)`, not via `SimpleAttributeDefinitionBuilder`, and does NOT support expression substitution. These `name` keys do not need expression coverage.

**Alternative attributes**: Some attributes are defined as mutually exclusive alternatives (via `.setAlternatives(...)` in Java, e.g., `auth-server-url` and `provider-url`). Combined coverage across all entries satisfies the requirement — one entry can use `auth-server-url` and another can use `provider-url`, and each needs both a literal and expression form across those combined entries.

### 2. Element Ordering Requirements

**CRITICAL**: Element ordering in test XML files MUST match the marshaller's output order, NOT the XSD schema order.

The XSD schema uses `<xs:all>` which allows elements in any order during parsing, but the marshaller outputs elements in a specific order determined by the subsystem's write logic.

**How to Determine Correct Ordering:**

1. Run the test with your initial XML file
2. If the test fails with a comparison error, examine the test output in:
   ```
   target/surefire-reports/{SubsystemName}SubsystemTestCase.txt
   ```
3. The "expected" section shows the marshaller's actual output order
4. Reorder elements in your test XML to match the marshaller's order
5. Re-run tests to verify

**Common Ordering Patterns:**

For the elytron-oidc-client subsystem (as an example):
- Basic configuration elements come first (e.g., `realm-public-key`, `auth-server-url`)
- Security/SSL elements follow (e.g., `ssl-required`, `truststore`)
- CORS elements are grouped together
- Timeout/connection elements are grouped
- New feature elements often come after core elements
- Request object elements follow authentication format elements

### 3. Cumulative Schema Evolution

Schema versions are cumulative - each new version builds upon the previous one:

- **Version 1.0**: Base elements
- **Version 2.0**: Base + new elements from 2.0
- **Preview 2.0**: Base + 2.0 + preview-specific elements (e.g., `scope`)
- **Preview 3.0**: Base + 2.0 + preview-2.0 + preview-3.0 elements (e.g., request object support)
- **Preview 4.0**: Base + 2.0 + preview-2.0 + preview-3.0 + preview-4.0 elements (e.g., OIDC logout)

**Best Practice**: When creating a new version's test file, start by copying the previous version's test file, update the namespace, then add only the new elements introduced in that version.

**Coverage propagation is mandatory**: When a coverage gap is discovered and fixed in an earlier version's test file (e.g., adding a missing literal to `1.0.xml`), that fix MUST be propagated to ALL subsequent versions. Each file should be logically identical to its predecessor except for:
1. The namespace URI (e.g., `urn:wildfly:elytron-oidc-client:1.0` → `urn:wildfly:elytron-oidc-client:2.0`)
2. New attributes/elements added by that schema version (added to the "full literal" and "full expression" entries)

Failing to cascade coverage fixes forward means later versions will silently have the same gap.

### 4. Test Values

Use realistic but clearly test-oriented values:

- **Keys/Certificates**: Use valid-format but obviously test keys
- **URLs**: Use localhost or example.com domains
- **Ports**: Use standard ports (8080, 8180, etc.)
- **Credentials**: Use UUIDs or clearly fake values
- **Timeouts**: Use reasonable values (not extreme)

### 5. Multiple Instances

Include multiple instances of repeatable elements to verify:
- Different configuration patterns
- Different credential types (e.g., `secret`, `jwt`, `secret-jwt`)
- Different deployment scenarios (e.g., with realm, with provider, bearer-only)

## Verification Process

### Running Tests

```bash
cd wildfly/elytron-oidc-client
mvn clean test -Dtest=ElytronOidcClientSubsystemTestCase
```

### Test Failure Analysis

When a test fails:

1. **Check the error type**:
   - `ComparisonFailure`: Element ordering issue or missing/extra elements
   - `ParseException`: Invalid XML or schema violation
   - Other errors: Logic or configuration issues

2. **For ComparisonFailure**:
   - Open `target/surefire-reports/ElytronOidcClientSubsystemTestCase.txt`
   - Compare "expected" (marshaller output) vs "actual" (your XML)
   - Look for differences in element order or content
   - Update your XML to match the expected output

3. **For ParseException**:
   - Verify XML is well-formed
   - Check element names match schema exactly
   - Verify required attributes are present
   - Ensure values match expected types (boolean, integer, string, etc.)

## Review Checklist for AI Agents

When reviewing or creating subsystem test files:

- [ ] Test file exists for each schema version
- [ ] Namespace in test XML matches schema version
- [ ] All schema elements are represented at least once as a literal value
- [ ] All schema elements are represented at least once as an expression (`${...}` form)
- [ ] All attributes (including optional) are included in both literal and expression form
- [ ] Element ordering matches marshaller output (verify by running tests)
- [ ] Multiple instances of repeatable elements are included
- [ ] Test values are realistic but clearly for testing
- [ ] New version test files build upon previous versions
- [ ] Coverage fixes in any version have been propagated to all later versions
- [ ] Tests pass without comparison failures

**Coverage verification strategy**: The most reliable way to verify expression coverage is to identify the "full literal" entry for each complex type and check it contains every attribute, then do the same for the "full expression" entry. Cross-file agent verification is prone to false positives — always confirm reported gaps by directly reading the XML before making changes.

## Common Pitfalls

1. **Ordering Assumption**: Don't assume XSD order = marshaller order
2. **Missing Optional Elements**: Include optional elements to ensure they can be parsed
3. **Incomplete Coverage**: Missing even one element can leave gaps in validation
4. **Copy-Paste Errors**: When copying from previous versions, update all version-specific references
5. **Namespace Mismatches**: Ensure namespace exactly matches the schema version
6. **Expression Coverage Gaps**: Forgetting that each attribute needs BOTH a literal and expression form, not just one
7. **Not Cascading Fixes**: Fixing a gap in one version but forgetting to apply the same fix to all later versions
8. **Confusing Path Keys with Model Attributes**: The `name` attribute on credentials and redirect-rewrite-rules is a management resource address key — it doesn't support expressions and doesn't need expression coverage
9. **Agent False Positives in Coverage Checks**: Automated agents scanning XML for coverage can miss entries that exist in different parts of the file. Always verify a reported gap by manually reading the XML before acting on it. Confirmed gaps will also exist identically in all versions derived from the one with the gap.

## Example: Adding a New Element

When a new element is added to a schema:

1. Identify which complex type(s) it belongs to
2. Add the element to ALL test files for versions that include it
3. Place it in the correct position (run tests to verify)
4. Use a representative test value
5. Run tests and adjust ordering if needed
6. Verify all tests pass

## References

- XSD Schemas: `src/main/resources/schema/`
- Test Files: `src/test/resources/org/wildfly/extension/{subsystem}/`
- Test Class: `src/test/java/org/wildfly/extension/{subsystem}/{Subsystem}SubsystemTestCase.java`
- Test Framework: `org.jboss.as.subsystem.test.AbstractSubsystemSchemaTest`