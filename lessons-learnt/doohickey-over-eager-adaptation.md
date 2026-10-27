# Lesson Learnt: Over-Eager Early-Access Adaptation — When Not to Add a Doohickey

## Background

During WFCORE-7712, an attempt was made to adapt `ldap-key-store` and its supporting `dir-context` resource to participate in the early TLS construction path. The intent was to make LDAP-backed key stores available as early as plain file-backed key stores, so an `ssl-context` that references an LDAP key store could be initialized before service startup.

## What Went Wrong

### `dir-context` silently dropped security configuration

`DirContextDefinition`'s early-initialization path (`createImmediately`) was implemented using the doohickey API to resolve the directory server connection. However, two optional security configurations — the authentication context and the SSL context — were looked up via MSC service suppliers and were therefore unavailable in the management thread. The early path silently omitted them, creating an unauthenticated, unencrypted directory context even when the operator had configured credentials and TLS.

This is a subtle and dangerous failure mode: the resource was "available" early, but with different security behavior than the configured service would have. No error was surfaced.

### LDAP key-store restart left stale state

When the `LdapKeyStoreDoohickey` was used, API-first creation cached the initial `KeyStore` instance. If the underlying `DirContext` service was subsequently restarted (e.g. due to a configuration change), the doohickey's cached key store was stale — it still referred to the old connection. The restart path left a null cached wrapper and a stale underlying store, producing errors on subsequent use.

### Lesson

> **Do not add early-access API support to a resource if `createImmediately` cannot faithfully reproduce the full configured behavior.**

Partial or degraded early initialization is worse than no early initialization: operators get surprising runtime behavior with no error message pointing at the cause. A clean capability-not-found error (`WFLYCTL0436` for a missing API capability) is far better — it tells the operator that the configuration they are trying is not supported in the early boot path.

## The Resolution

`LdapKeyStoreDoohickey` was deleted. `DirContextDefinition` was restored to a service-only handler. `ldap-key-store` no longer advertises `KEY_STORE_API_CAPABILITY`. Configurations that require an LDAP-backed key store in the early TLS chain receive a clean capability-not-found error.

## The General Rule

Before adding doohickey support to a resource, verify that **all** of the following can be faithfully reconstructed in `createImmediately` from the management context alone (i.e. without MSC dependency injection):

1. All security-relevant configuration (credentials, TLS, authentication context) can be resolved through their own runtime APIs or the operation context.
2. The constructed object behaves identically to the service-path object for the consumer's purpose.
3. Restart semantics can be honored: `reset()` and a subsequent `createImmediately` will produce an equivalent fresh object.

If any of these cannot be guaranteed, the resource should remain service-only. The early TLS construction path will then fail with a capability-not-found error when an operator uses that resource type, which is the correct and honest failure mode.

## Date

2026-09-28 — root cause identified and resolved during WFCORE-7712 code review iterations.
