# Lesson Learnt: WFLYCTL0436 — Capability Leak on Resource Remove/Re-Add

## The Bug

When a WildFly management resource registers a **runtime API capability** (i.e. a doohickey-style `*-api` capability) in its add handler's `recordCapabilitiesAndRequirements`, and the remove handler is the standard `TrivialCapabilityServiceRemoveHandler` without any override, the API capability is **never deregistered** on remove.

This goes unnoticed as long as the resource name is only ever added once per server lifetime. But as soon as a test (or an operator) removes and re-adds the same resource, WildFly rejects the second add with:

```
WFLYCTL0436: Cannot register capability 'org.wildfly.security.key-store-api.foo'
in context 'org.wildfly.extension.elytron' because it is already registered there.
```

## How It Was Discovered

During WFCORE-7712 development, the full WildFly Core `allTests` suite was run on a VM after implementing doohickey support across the TLS resource chain. The suite failed with exactly one error:

```
SecurityCommandsTestCase.testInvalidEnableSSL:557 — WFLYCTL0436: Cannot register capability
'org.wildfly.security.key-store-api.foo' … already registered
```

The test exercises a scenario where `key-store` resources are added, removed, and added again during SSL configuration validation. Because `KeyStoreDefinition`'s add handler registered `KEY_STORE_API_CAPABILITY` but the remove handler was the plain service-only variant, the API capability was stuck in the global registry after the first removal. The Elytron subsystem tests (347 tests) all passed, because they never exercised the same-name add-remove-add cycle for a doohickey-adapted resource.

**The lesson: Elytron subsystem unit tests alone are insufficient to catch this class of bug. Full `allTests` — or a dedicated same-name remove/re-add test — is required.**

## Root Cause

`TrivialCapabilityServiceRemoveHandler` only calls `context.removeService(...)` and handles the normal service capability. It does not know about dynamically registered API capabilities. The add handler that calls `context.registerCapability(API_CAPABILITY, ...)` in `recordCapabilitiesAndRequirements` must be paired with a remove handler that calls `context.deregisterCapability(...)` for each such capability.

## The Fix

`DoohickeyAddHandler` gained a factory method:

```java
public AbstractRemoveStepHandler createRemoveHandler(RuntimeCapability<?>... apiCapabilities) {
    // Returns an anonymous TrivialCapabilityServiceRemoveHandler subclass that
    // deregisters each apiCapability in recordCapabilitiesAndRequirements before
    // delegating to the super implementation.
}
```

Every resource that uses `DoohickeyAddHandler` now calls:

```java
static final AbstractRemoveStepHandler REMOVE = ADD.createRemoveHandler(API_CAPABILITY);
```

For resources that extend `BaseAddHandler` directly (such as `KeyStoreDefinition` and `FilteringKeyStoreDefinition`), the inline remove handler was replaced with an anonymous subclass that explicitly deregisters the API capability.

Eight files were changed in WildFly Core to apply this fix consistently:
- `KeyStoreDefinition` (inline anonymous override)
- `FilteringKeyStoreDefinition` (inline anonymous override)
- `CredentialStoreResourceDefinition` (switched to `add.createRemoveHandler(...)`)
- `SecretKeyCredentialStoreResourceDefinition` (switched to `add.createRemoveHandler(...)`)
- `AggregateComponentDefinition` (api-capable overload switched)
- `SSLDefinitions` (key-manager, trust-manager, client-ssl-context builders)
- `ProviderDefinitions` (switched to `add.createRemoveHandler(...)`)

## The Rule

> **Any add handler that calls `context.registerCapability()` in `recordCapabilitiesAndRequirements` for an API capability must have a matching `context.deregisterCapability()` in its remove handler.**

This applies to:
- `DoohickeyAddHandler` subclasses — use `add.createRemoveHandler(API_CAPABILITY, ...)`
- `BaseAddHandler` subclasses that manually register an API capability — override the remove handler inline

The same rule applies to extensions outside WildFly Core. If the Vault feature pack adds a `VAULT_CREDENTIAL_STORE_API_CAPABILITY` for doohickey-style early access, its remove handler must deregister that capability.

## How to Verify

Add (or find) a test that:
1. Adds the resource.
2. Removes it.
3. Adds it again with the same name.
4. Asserts the second add succeeds with no `WFLYCTL0436` error.

In WildFly Core, `SecurityCommandsTestCase.testInvalidEnableSSL` exercises this path for key-store resources indirectly through the CLI `security enable-ssl-management` flow. A dedicated doohickey cycle test in `ElytronDoohickeyTestCase` is the clearest way to cover this for any new resource.

## Date

2026-09-28 — discovered during WildFly Core WFCORE-7712 full `allTests` run.
