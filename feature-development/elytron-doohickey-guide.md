# ElytronDoohickey Guide

## Overview

`ElytronDoohickey<T>` coordinates the creation of one Elytron runtime value when the value can be requested through either an MSC service or a management runtime API. `DoohickeyAddHandler<T>` registers both access paths for a resource. Both classes are package-private helpers in the WildFly Core Elytron subsystem; other extensions cannot subclass or instantiate them directly.

Use this pattern when a resource may be needed during management operations or expression resolution **before its MSC service is available**, while other consumers still need the normal service capability. A resource used only after its service starts does not need a doohickey.

The source examples are [`ElytronDoohickey.java`](https://github.com/wildfly/wildfly-core/blob/main/elytron/src/main/java/org/wildfly/extension/elytron/ElytronDoohickey.java), [`DoohickeyAddHandler.java`](https://github.com/wildfly/wildfly-core/blob/main/elytron/src/main/java/org/wildfly/extension/elytron/DoohickeyAddHandler.java), and the [Elytron resource definitions](https://github.com/wildfly/wildfly-core/tree/main/elytron/src/main/java/org/wildfly/extension/elytron).

## Why Elytron Needs It

A credential reference can ask for a credential store while resolving a management operation. The store might itself need a credential reference or a configured provider loader. At that point, waiting for the credential store's MSC service can be too late: service dependencies have not necessarily been installed or started. [`CredentialReference.getCredentialSource()`](https://github.com/wildfly/wildfly-core/blob/main/controller/src/main/java/org/jboss/as/controller/security/CredentialReference.java) therefore obtains the credential store runtime API and calls `apply(context)`.

The doohickey provides two ways to construct the same value:

| Path | Entry point | Dependency access |
|------|-------------|-------------------|
| Management runtime API | `apply(OperationContext)` | On first initialization, read the resource model and obtain other resources through their runtime APIs or the supplied context. MSC dependency suppliers are unavailable. |
| MSC service start | `get()` | On first initialization, use the suppliers wired by `prepareServiceSupplier()` on the `CapabilityServiceBuilder`. |

The first path to initialize the doohickey stores the value; the other path returns that value. This avoids independently constructing a service instance and an API instance for the same resource. The implementation synchronizes initialization and checks for recursive doohickey calls, reporting the resource address chain when it detects a cycle. This check covers recursion through doohickeys; it does not replace MSC dependency validation.

## Use Across Extensions

An extension outside Elytron can implement the **pattern** for its own resources using the public WildFly Core capability and service APIs. It cannot use `ElytronDoohickey` or `DoohickeyAddHandler` as a library: their package visibility and Elytron-specific dependencies make them implementation details. If a shared helper is needed, it would need to be deliberately extracted as a public API with a supported contract.

**Yes, an early initialization path in another extension can call Elytron's early initialization path.** Its own doohickey does not need a reference to the `ElytronDoohickey` class. From `createImmediately(OperationContext)`, it can look up the named Elytron runtime API as the public `ExceptionFunction` interface and invoke `apply(context)`:

```java
ExceptionFunction<OperationContext, CredentialStore, OperationFailedException> storeApi =
        context.getCapabilityRuntimeAPI("org.wildfly.security.credential-store-api",
                storeName, ExceptionFunction.class);
CredentialStore store = storeApi.apply(context);
```

If Elytron has not initialized that store yet, `apply(context)` reads its model and creates it immediately. This is the same mechanism Elytron's credential-store doohickey uses to obtain a provider loader from another doohickey. It works after capability registration, in a runtime management context where the named Elytron capability is visible; it does not require either MSC service to have started. The consuming module must be able to see the public types in the API, such as `ExceptionFunction` and `CredentialStore`; it does not need access to the doohickey class. The other extension's service-start path should separately wire a normal MSC dependency. Elytron's cycle tracker only covers `ElytronDoohickey` instances, so a separate implementation must consider cycles that cross extension boundaries.

A runtime API lookup does not register a model dependency by itself. For example, Elytron's credential-store `providers` attribute registers a requirement on the `org.wildfly.security.providers` capability, while its immediate construction path calls `org.wildfly.security.providers-api`. A resource that refers to an Elytron resource must record the appropriate capability requirement during the MODEL stage; the early lookup then occurs in the RUNTIME stage. See [`CredentialStoreResourceDefinition.java`](https://github.com/wildfly/wildfly-core/blob/main/elytron/src/main/java/org/wildfly/extension/elytron/CredentialStoreResourceDefinition.java) for both sides of this pattern.

Other extensions can consume the published Elytron service capabilities. The [capability registry](https://github.com/wildfly/wildfly-capabilities/blob/main/registry.txt) lists [`org.wildfly.security.credential-store`](https://github.com/wildfly/wildfly-capabilities/blob/main/org/wildfly/security/credential-store/capability.adoc), which provides `CredentialStore`, and [`org.wildfly.security.providers`](https://github.com/wildfly/wildfly-capabilities/blob/main/org/wildfly/security/providers/capability.adoc), which provides `Provider[]`. Both are dynamic and public. A consumer should declare the appropriate capability requirement and obtain the service name through `OperationContext` in the runtime stage, following the [WildFly extension capability guidance](https://docs.wildfly.org/38/Extending_WildFly.html#_basics_of_using_other_capabilities).

The Elytron add handler registers `org.wildfly.security.credential-store-api` and `org.wildfly.security.providers-api` as dynamic runtime APIs; WildFly Core's `CredentialReference` uses the former. These `*-api` names are **not listed in the published capability registry**. Therefore, unlike the service capabilities, they have no documented stable cross-extension contract; treat them as internal implementation details. An external extension that needs this early access should coordinate a supported API contract with Elytron. The service capability itself does not expose the doohickey runtime API.

## How the Add Handler Works

`DoohickeyAddHandler<T>` extends `BaseAddHandler` and takes the normal runtime capability plus a separate, dynamically named API capability. During capability registration it creates a doohickey for the resource address and registers it as an `ExceptionFunction<OperationContext, T, OperationFailedException>` runtime API. This registration occurs only when `requiresRuntime(context)` is true.

In `performRuntime()`, the handler retrieves that **same doohickey**, calls `resolveRuntime(context)`, lets it prepare the service supplier and dependencies, then installs an active `TrivialService<T>` whose value supplier calls `doohickey.get()`. An early `apply(context)` may initialize the value before service start; otherwise the service start initializes it.

`ElytronDoohickey<T>` delegates resource-specific work to three methods:

| Method | Responsibility |
|--------|----------------|
| `resolveRuntime(ModelNode, OperationContext)` | Resolve and validate model attributes, including expressions, into state needed by either path. Do not create the runtime value here. |
| `prepareServiceSupplier(OperationContext, CapabilityServiceBuilder<?>)` | Declare MSC dependencies with the builder and return a supplier that constructs the value at service start. Do not call dependency suppliers while preparing the service. |
| `createImmediately(OperationContext)` | Construct the value from a management context without depending on MSC injection. Resolve referenced doohickey resources through their runtime APIs where appropriate. |

The doohickey's `resolveRuntime(OperationContext)` method owns synchronization and reads the model from the resource address. Call it through the base class; do not call the protected `resolveRuntime(ModelNode, OperationContext)` method directly from an add handler.

## End-to-End Lifecycle

1. **MODEL stage:** The resource definition's normal service capability is registered. `DoohickeyAddHandler` also registers the named runtime API capability containing the doohickey when runtime support is required. Attribute definitions register requirements for configured references. Runtime API lookup is not available in the MODEL stage.
2. **Early request:** After capability resolution, a runtime management step can obtain the API capability and call `apply(context)` even if the MSC service has not started. If the value is absent, the doohickey reads its resource model, resolves runtime attributes, and calls `createImmediately(context)`. That method can call another doohickey's API in the same way. The resulting non-null value is cached in the doohickey.
3. **Service installation and start:** The resource's `performRuntime()` obtains that same doohickey, calls `resolveRuntime(context)` and `prepareService(...)`, and installs a `TrivialService` wired to `doohickey.get()`. The service dependencies are still registered even if the value was created early. When MSC starts the service, `get()` returns the cached value and the service publishes it. Early API initialization alone does not mean the MSC service has started or that its dependencies are satisfied.
4. **Service-first case:** If MSC starts the service before any API request, `get()` creates and caches the value using the prepared service supplier. A later `apply(context)` returns that same value without creating another one.

An initial `apply()` needs a non-null `OperationContext`; `apply(null)` works only after the value has been initialized. The API capability must be registered and visible in the caller's capability scope. A caller should not assume that an early value makes its service available to MSC dependents.

## Adapting an Elytron Resource

### 1. Decide Whether Both Access Paths Are Required

Find the call that needs the resource before its service is available, such as expression resolution or a runtime management operation. Keep the existing service capability for normal service consumers. Define a separate API capability name for callers that need early access. Both capability names must use the same resource name when looked up dynamically.

### 2. Use `DoohickeyAddHandler<T>`

Pass the runtime capability and API capability name to the handler. Override `createDoohickey(PathAddress)` to return one `ElytronDoohickey<T>` for that resource. Preserve any resource-specific `populateModel()`, rollback, remove, and attribute-write behavior from the old handler.

The credential-store resource illustrates the handler change in [`CredentialStoreResourceDefinition.java`](https://github.com/wildfly/wildfly-core/blob/main/elytron/src/main/java/org/wildfly/extension/elytron/CredentialStoreResourceDefinition.java); the smaller provider-loader example is in [`ProviderDefinitions.java`](https://github.com/wildfly/wildfly-core/blob/main/elytron/src/main/java/org/wildfly/extension/elytron/ProviderDefinitions.java).

### 3. Implement the Shared Model Resolution and Both Construction Paths

Resolve model attributes once in `resolveRuntime(ModelNode, OperationContext)`. Keep the construction logic equivalent across paths, preferably by sharing a private factory method after their dependencies have been obtained. The paths differ in **how** dependencies are obtained:

```java
// Simplified shape of AggregateApiComponentAddHandler in AggregateComponentDefinition.
protected void resolveRuntime(ModelNode model, OperationContext context)
        throws OperationFailedException {
    references = aggregateReferences.unwrap(context, model);
}

protected ExceptionSupplier<T, StartException> prepareServiceSupplier(
        OperationContext context, CapabilityServiceBuilder<?> builder) {
    List<Supplier<T>> dependencies = new ArrayList<>();
    for (String name : references) {
        dependencies.add(builder.requires(serviceNameFor(name)));
    }
    return () -> aggregateValues(dependencies.stream().map(Supplier::get).toList());
}

protected T createImmediately(OperationContext context) throws OperationFailedException {
    List<T> values = new ArrayList<>();
    for (String name : references) {
        ExceptionFunction<OperationContext, T, OperationFailedException> api =
                context.getCapabilityRuntimeAPI(apiCapabilityName, name, ExceptionFunction.class);
        values.add(api.apply(context));
    }
    return aggregateValues(values);
}
```

This is a shape example, not a drop-in implementation: use the resource's actual capability names, service type, and error handling. In particular, `createImmediately()` cannot rely on suppliers returned by `builder.requires(...)`. For file paths, the provider-loader and credential-store implementations show both the MSC `PathManager` dependency and immediate resolution through `resolveRelativeToImmediately()`.

### 4. Update Early Callers Within the Internal API Contract

An early caller obtains the named API capability and passes its `OperationContext` to `apply(context)`:

```java
ExceptionFunction<OperationContext, CredentialStore, OperationFailedException> api =
        context.getCapabilityRuntimeAPI(CREDENTIAL_STORE_API_CAPABILITY, storeName,
                ExceptionFunction.class);
CredentialStore store = api.apply(context);
```

This example is from WildFly Core's internal Elytron integration. The same lookup works from another extension's runtime context, subject to the cross-extension contract caveat above. The context is needed if this call performs first initialization. `apply(null)` is only valid after the value has already been initialized; the helper treats earlier calls without a context as a programming error. Keep ordinary service consumers on the public service capability.

## Verification Steps

- [ ] An API call before service start builds the value using the management context.
- [ ] An API call before the resource's `performRuntime()` can initialize it; later service installation still wires its dependencies.
- [ ] Service start without an earlier API call builds the value using wired MSC dependencies.
- [ ] When both paths run, they obtain the same runtime value rather than separate instances.
- [ ] Configured references register model-stage capability requirements; the early runtime lookup does not substitute for them.
- [ ] Referenced doohickey resources can initialize through their APIs without relying on unstarted services.
- [ ] Early initialization followed by service start publishes the same value, while MSC still enforces service dependencies.
- [ ] Recursive references among Elytron doohickeys fail with a useful resource address chain; cross-extension cycles are checked separately.
- [ ] Existing add, remove, reload, and runtime operations still behave as required for the resource.

The Elytron [credential-store tests](https://github.com/wildfly/wildfly-core/blob/main/elytron/src/test/java/org/wildfly/extension/elytron/CredentialStoreTestCase.java) and [expression-resolution tests](https://github.com/wildfly/wildfly-core/blob/main/elytron/src/test/java/org/wildfly/extension/elytron/ExpressionResolutionTestCase.java) are useful starting points when adapting a credential-store-related resource.

## Summary Checklist

- [ ] Confirm early runtime access is needed.
- [ ] Keep the normal service capability and add a named runtime API capability.
- [ ] Have `DoohickeyAddHandler<T>` create and register the doohickey.
- [ ] Resolve model attributes once; implement service and immediate construction paths.
- [ ] Make early callers use the API capability with an `OperationContext`.
- [ ] Verify early access, service-first access, and recursive references.

## Revision History

- **2026-09-26**: Initial guide based on the WildFly Core Elytron implementation.
