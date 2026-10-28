# Logging and user-facing messages

## Overview

These requirements apply to all code changes, including bug fixes, refactoring, maintenance and new features.

Use JBoss Logging's typed message interfaces for exceptions and other messages that may reach end users, including operational messages logged at **INFO, WARN or ERROR**. Use inline messages for **DEBUG and TRACE** diagnostics. If a message could be emitted for every request, prefer **TRACE**.

This guide records the project conventions requested in [agent-notes #18](https://github.com/wildfly-security-incubator/agent-notes/issues/18). Existing code contains exceptions to these conventions; follow the guidance here for new work rather than copying every existing declaration.

### Two roles of typed messages

1. **Unique identifiers:** a project code and numeric message ID give a diagnostic a searchable identity, independent of its translated wording. For example, `projectCode = "ELY"`, `length = 5` and `id = 19004` produce the prefix `ELY19004:`.
2. **Translation:** annotated message templates allow translations to be supplied through JBoss Logging Tools. Keeping complete sentences in the interface makes them available for translation; it does not itself provide translated text.

These roles apply even when no log entry is written. A logger interface can return an exception or a formatted `String` as well as emit a log record. Its name, often `ElytronMessages`, does not restrict it to logging.

## Pre-change Checklist

- Find the module's existing typed message interface, logger category and project code.
- Decide whether the text is an end-user diagnostic, UI text or developer-only logging.
- For logging, consider frequency as well as severity: could this path run for every request?
- Check the allocated range and existing IDs before adding an identified message.
- Preserve a failure's cause when wrapping it in a typed exception.

## Message Selection Process

| Use | Pattern | Message ID |
| --- | --- | --- |
| Exception whose message may reach an end user | `@Message` method returning the exception type | Normally required |
| Other user-facing diagnostic text | `@Message` method returning `String` | Normally required |
| Operational INFO, WARN or ERROR log | `@LogMessage` plus `@Message` on a `void` method | Required |
| UI label, prompt or help text where a prefix would be inappropriate | `@Message(id = Message.NONE, ...)` returning `String` | Intentionally omitted; still translatable |
| DEBUG or TRACE diagnostic log | Inline `debugf`, `tracef` or equivalent | No ID or annotated message declaration |

### Exceptions and formatted messages

Put the complete message template in the existing interface and pass changing values as parameters. Avoid assembling a sentence from separately translated fragments or appending English text at the call site.

This fragment is from Elytron's encryption module:

```java
@Message(id = 19004, value = "Unable to decode Base64 token.")
GeneralSecurityException unableToDecodeBase64Token(@Cause Throwable cause);
```

At the failure site, call the factory and throw the returned exception:

```java
throw log.unableToDecodeBase64Token(cause);
```

`@Cause` preserves the underlying failure as the exception cause. Calling this method constructs an exception; it does **not** log it. A `String`-returning method similarly formats text without logging it. Avoid logging a failure at every layer when the caller already reports it.

### INFO, WARN and ERROR logs

Define an identified, translatable message and declare its level explicitly. For example, Elytron's token realm declares:

```java
@LogMessage(level = WARN)
@Message(id = 1126, value = "Jwt-based token realm not configured with a list of valid issuers. Ignoring issuer verification.")
void tokenRealmJwtWarnNoIssuerIgnoringIssuerCheck();
```

Calling the method emits the log record. Use the same annotation pattern for INFO and ERROR. Select the level according to what the operator needs to know: normal significant events at INFO, conditions requiring attention at WARN, and failures at ERROR. Use `@Cause` when a logged exception's stack trace is needed.

The numeric IDs in these examples are **already allocated**. Do not copy them for a new message.

### UI strings without an ID

An identifier prefix is often inappropriate for labels, help text and prompts. Keep these strings in the typed interface for translation and explicitly use `Message.NONE`.

For example, `ElytronToolMessages` imports `org.jboss.logging.annotations.Message.NONE` and declares:

```java
@Message(id = NONE, value = "Password for credential store")
String cmdLineCredentialStorePassword();
```

The caller displays the returned text in the UI. Omitting its diagnostic prefix does not mean hard-coding the text at the display site. Use this exception where the presentation warrants it; retain IDs for ordinary operational diagnostics.

### DEBUG, TRACE and request frequency

Use the existing logger's inherited `BasicLogger` methods with an inline template:

```java
log.debugf("Initializing authentication mechanism %s", mechanismName);
log.tracef("Evaluating request with authentication mechanism %s", mechanismName);
```

Do not add `@Message` / `@LogMessage` declarations or allocate IDs for these diagnostic statements. Guard expensive argument construction with `isDebugEnabled()` or `isTraceEnabled()`; Java evaluates method arguments even when the level is disabled.

**If a diagnostic could occur per request, prefer TRACE rather than DEBUG.** This includes routine authentication attempts, authorization decisions, credential lookups and protocol progress. Consider the busiest deployment, including rejected requests, rather than the frequency seen in a small test. A failure outcome alone does not make each request an operational WARN or ERROR. Report an actionable service or configuration failure at the appropriate higher level without repeating it for every request.

Keep credentials, tokens and other secrets out of messages at every level, including TRACE. Dedicated security audit events have a separate purpose; this diagnostic logging guidance does not replace audit requirements.

## Elytron Message ID Allocation Process

### Project code, category and range

Elytron's module loggers share the project code `ELY`, normally with five numeric digits. The logger category controls filtering and routing; it does not give messages a separate ID namespace. Two logger interfaces with different categories still need distinct new IDs when they share the project code.

For example, the encryption module declares:

```java
@MessageLogger(projectCode = "ELY", length = 5)
@ValidIdRanges({
    @ValidIdRange(min = 19000, max = 19999)
})
public interface ElytronMessages extends BasicLogger {
    // Message methods use this module's allocated IDs.
}
```

Follow the target project's existing `Logger.getMessageLogger(...)` construction and category. Related repositories can have different project codes: Elytron EE's newer messages use `ELYEE`, and the HashiCorp Vault integration uses `ELYHCVT`. Some Elytron EE interfaces retain legacy `ELY` IDs. Check the actual interface before choosing a namespace or range.

### Adding an ID or reserving a range

1. Read **[`ELY_Messages.txt`](https://github.com/wildfly-security/wildfly-elytron/blob/8ed8a15ba2667f4ced355060d34a5acf8a0b2f5a/ELY_Messages.txt)** at the Elytron repository root on the branch being changed. It records existing allocations and reservations. Inspect the live file, not just the linked snapshot, before choosing a range.
2. For an existing logger, choose an unused ID within its allocation. Check both its `@ValidIdRange` / `@ValidIdRanges` declarations and the existing message methods. A gap in one interface does not prove an ID is free across the project.
3. For a new allocation, reserve an unused range by recording the owning module in `ELY_Messages.txt`. The file requests new ranges in **blocks of 1,000** and also contains branch-specific reservations; do not treat those as unallocated space. Coordinate the reservation with maintainers and concurrent work.
4. Add or extend the logger's range annotations to match the allocation, then add message methods with distinct IDs. Include the reservation and annotation changes with the feature, or reference a separately agreed reservation change.
5. Preserve existing IDs and their meanings. Do not renumber messages or recycle a removed message's ID for an unrelated diagnostic. Historical allocations include overlaps from moved or deprecated code; they are not a pattern for assigning new IDs.

The root text file is a reservation record; it does not replace the annotations or the compiler's annotation processing. Likewise, a successful module compilation does not establish that a proposed range is free in every module or branch.

## Verification Steps

✅ Review each new message against the selection table, including exceptions that are returned or thrown without logging.

✅ Check new IDs against the target branch's allocation record and logger interfaces. Confirm that the project code, numeric width and range match the module.

✅ Run the affected module's normal build with its existing JBoss Logging annotation processor enabled. Check for duplicate or out-of-range IDs, invalid method signatures and format/parameter mismatches. Use the project's existing dependency and processor configuration.

✅ Exercise the affected failure or output path where appropriate: confirm the exception type and cause, diagnostic prefix, or intentionally prefix-free UI text. Do not rely on exact translated wording as a stable identifier.

✅ Review call sites for per-request frequency and secret disclosure. Check that DEBUG/TRACE text is inline and routine request diagnostics use TRACE.

## PR Review Guidelines

- Are user-facing exceptions and INFO/WARN/ERROR messages declared in the typed interface?
- Does each new diagnostic have a distinct ID in the correct namespace and reserved range?
- Is `Message.NONE` justified by the way text is displayed, such as a UI prompt or help string?
- Can a translator work with a complete template and correctly matched placeholders?
- Are causes retained without logging the same exception repeatedly?
- Could any message repeat per request, and should it be TRACE?

## Source Examples

These sources show the mechanisms used by the guide. Existing call sites may use different levels or annotate DEBUG/TRACE messages; those details do not override the convention above.

| Source | What to inspect |
| --- | --- |
| [Elytron encryption messages](https://github.com/wildfly-security/wildfly-elytron/blob/8ed8a15ba2667f4ced355060d34a5acf8a0b2f5a/encryption/src/main/java/org/wildfly/security/encryption/ElytronMessages.java) | Shared `ELY` code, allocated range and typed exception with `@Cause` |
| [Elytron token realm messages](https://github.com/wildfly-security/wildfly-elytron/blob/8ed8a15ba2667f4ced355060d34a5acf8a0b2f5a/auth/realm/token/src/main/java/org/wildfly/security/auth/realm/token/_private/ElytronMessages.java) | Operational WARN declarations and exception factories |
| [Elytron tool messages](https://github.com/wildfly-security/wildfly-elytron/blob/8ed8a15ba2667f4ced355060d34a5acf8a0b2f5a/tool/src/main/java/org/wildfly/security/tool/ElytronToolMessages.java) | `String` methods using `Message.NONE` for CLI presentation |
| [HashiCorp Vault logger](https://github.com/wildfly-security/wildfly-elytron-hashicorp-vault/blob/7ca556dba6e85ca452874dff4861679562ab6ccc/src/main/java/org/wildfly/security/hashicorp/vault/_private/HashiCorpVaultLogger.java) | Separate project code and width, typed exceptions and logged causes |

## Summary Checklist

- [ ] Use typed messages for end-user diagnostics, exceptions and INFO/WARN/ERROR logs.
- [ ] Support both identifiable diagnostics and translation; omit IDs deliberately for suitable UI strings.
- [ ] Use inline DEBUG/TRACE messages and prefer TRACE for per-request diagnostics.
- [ ] Check project-wide allocations, reserve new ranges in `ELY_Messages.txt`, and preserve existing IDs.
- [ ] Verify annotation processing, output behavior, causes and message frequency.
