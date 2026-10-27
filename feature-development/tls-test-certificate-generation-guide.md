# Generating TLS test certificates and stores

## Overview

Generate ordinary TLS test material during setup rather than committing certificates or stores that expire. Keep the trust relationships visible: which CA signs the server, which CA signs the client, and which peer trusts each CA.

For Elytron Java tests, prefer Elytron's certificate builders with JCA store creation. This choice follows the Elytron project preference recorded in [Vault library #98](https://github.com/wildfly-security/wildfly-elytron-hashicorp-vault/issues/98) and [agent-notes #17](https://github.com/wildfly-security-incubator/agent-notes/issues/17); it is guidance for these projects, not a formally agreed ecosystem-wide standard.

## Pre-generation Checklist

- Identify the actual connection host, including any Testcontainers host override. A DNS SAN and an IP SAN are different certificate identities.
- Decide whether the test needs server TLS, transport mutual TLS, application certificate authentication, or a combination.
- Select the store type expected by the consuming API and explicit store/key passwords.
- Assign each fixture its own directory and cleanup owner, including setup and startup failures.

| Need | Approach |
| --- | --- |
| Self-signed identity in an Elytron Java test | `SelfSignedX509CertificateAndSigningKey` |
| CA-signed server/client identities, SANs and extensions | The self-signed builder for the CA; `X509CertificateBuilder` for leaves |
| Simple stores needed before tests execute | Maven `keytool-maven-plugin`, as used in [WildFly's shared tests](https://github.com/wildfly/wildfly/blob/2ae2943bc295f68933f30604423a2ff5bbca6dc2/testsuite/shared/pom.xml) |
| Existing Elytron tests using a reusable CA hierarchy | Consider `CAGenerationTool`; check test-artifact availability and defaults first |
| Specialist CRL/OCSP protocol fixtures | Follow the relevant project's revocation-test utilities; additional libraries such as Bouncy Castle may be justified |

SmallRye Certs previously generated the Vault fixtures. The migration uses Elytron to match the project and remove that extra generator dependency. Ordinary certificate generation and PEM encoding do not require Bouncy Castle.

## Generation Process

### Dependency and checked example

The following example uses released `org.wildfly.security:wildfly-elytron-x500-cert:2.8.3.Final`, in test scope. Use the version managed by your project when available. The example was compiled with `--release 17` and executed on Java 25 against that released JAR, including signature verification and JKS reload. See also WildFly's [CA-signed keystore-realm fixtures](https://github.com/wildfly/wildfly/blob/2ae2943bc295f68933f30604423a2ff5bbca6dc2/testsuite/integration/elytron/src/test/java/org/wildfly/test/integration/elytron/certs/realm/KeystoreRealmTestCase.java).

This fragment belongs in a test method declaring `throws Exception`. Imports are from `java.security`, `java.security.cert`, `java.nio.file`, `java.time`, `java.math`, `java.util`, `javax.security.auth.x500`, `org.wildfly.security.x500`, and `org.wildfly.security.x500.cert`.

```java
Path testDirectory = Files.createDirectories(Path.of("target", "tls-fixtures"));
Path directory = Files.createTempDirectory(testDirectory, "tls-");
char[] password = "test-password".toCharArray();
ZonedDateTime now = ZonedDateTime.now();
SelfSignedX509CertificateAndSigningKey ca =
        SelfSignedX509CertificateAndSigningKey.builder()
                .setDn(new X500Principal("CN=Test CA"))
                .setKeyAlgorithmName("RSA").setKeySize(2048)
                .setSignatureAlgorithmName("SHA256withRSA")
                .setNotValidBefore(now.minusDays(1))
                .setNotValidAfter(now.plusDays(30))
                .addExtension(new BasicConstraintsExtension(true, true, 0))
                .addExtension(new KeyUsageExtension(KeyUsage.keyCertSign, KeyUsage.cRLSign))
                .build();
KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA");
generator.initialize(2048);
KeyPair keys = generator.generateKeyPair();
X509Certificate server = new X509CertificateBuilder()
        .setIssuerDn(ca.getSelfSignedCertificate().getSubjectX500Principal())
        .setSubjectDn(new X500Principal("CN=Test Server"))
        .setSerialNumber(new BigInteger(159, new SecureRandom()).add(BigInteger.ONE))
        .setNotValidBefore(now.minusDays(1)).setNotValidAfter(now.plusDays(30))
        .setSigningKey(ca.getSigningKey()).setPublicKey(keys.getPublic())
        .setSignatureAlgorithmName("SHA256withRSA")
        .addExtension(new BasicConstraintsExtension(true, false, -1))
        .addExtension(new KeyUsageExtension(KeyUsage.digitalSignature, KeyUsage.keyEncipherment))
        .addExtension(new ExtendedKeyUsageExtension(false, List.of("1.3.6.1.5.5.7.3.1")))
        .addExtension(new SubjectAlternativeNamesExtension(false, List.of(
                new GeneralName.DNSName("localhost"), new GeneralName.IPAddress("127.0.0.1"))))
        .build();
KeyStore identity = KeyStore.getInstance("JKS");
identity.load(null, password);
identity.setKeyEntry("server", keys.getPrivate(), password,
        new X509Certificate[] { server, ca.getSelfSignedCertificate() });
try (var out = Files.newOutputStream(directory.resolve("identity.jks"))) {
    identity.store(out, password);
}
KeyStore trust = KeyStore.getInstance("JKS");
trust.load(null, password);
trust.setCertificateEntry("ca", ca.getSelfSignedCertificate());
try (var out = Files.newOutputStream(directory.resolve("trust.jks"))) {
    trust.store(out, password);
}
```

Keep generated material under the module's `target/` directory so Maven clean removes any leftovers; remove each fixture's `directory` recursively in teardown or `finally` as well. The SANs above suit localhost connections. Add the actual container host as a DNS or IP SAN when it differs. JKS is explicit because the Vault library initialization path requests JKS. Other consumers may use PKCS12; choose and test the format at that API boundary.

For a client identity, generate another key pair and leaf with client-auth EKU `1.3.6.1.5.5.7.3.2`. Give the server the issuing client CA certificate as its client trust material. A key store chain starts with the leaf and continues through its issuer certificates. Mark CAs with CA constraints and certificate-signing usage; mark leaves as non-CAs with appropriate usage. Use generation-relative validity and positive, distinct serials per issuer.

### Matching PEM for a container

Reuse the same keys and certificates as the JKS outputs. A small encoder can wrap `Base64.getMimeEncoder(64, new byte[] { '\n' }).encodeToString(encoded)` between PEM BEGIN/END lines. Use `CERTIFICATE` for `certificate.getEncoded()` and `PRIVATE KEY` for a generated JCA private key's PKCS#8 `getEncoded()`. Put the leaf before its issuer certificates in a PEM chain; write the CA certificate separately for trust.

The Vault #98 change supplies `TlsTestMaterial.java` and `VaultContainerHttps.java` under `src/test/java/org/wildfly/security/hashicorp/vault/`. These keep paths and identity material together, emit PEM and password-protected JKS from the same identity, and give each container its own server/client CAs. Untrusted-certificate controls generate independent material. Avoid static certificate fields shared by live containers. Keep host temporary directories private and ensure copied container files are readable by the container's runtime user.

### When to consider CAGenerationTool

In [Elytron 2.8.3.Final](https://github.com/wildfly-security/wildfly-elytron/blob/2.8.3.Final/tests/common/src/test/java/org/wildfly/security/ssl/test/util/CAGenerationTool.java), `CAGenerationTool` is test support, not part of the certificate-builder artifact. Its [module POM](https://github.com/wildfly-security/wildfly-elytron/blob/2.8.3.Final/tests/common/pom.xml) builds `org.wildfly.security:wildfly-elytron-tests-common` with a `tests` classifier. That establishes its source packaging, not availability in a particular artifact repository.

Consider it when tests already depend on that support and need its predefined hierarchy. Check artifact resolution and transitive test dependencies before introducing it elsewhere. Its default stores are PKCS12; do not assume JKS or automatic recursive cleanup. The direct builder approach above fits the small Vault fixtures without coupling them to a separate project's test support.

## Verification Steps

✅ Complete the TLS configuration builder before use. In the Vault Java driver, setting `SslConfig` fields is insufficient: call `build()` to construct the context and apply verification. The #98 property tests exposed an omitted call that left verification disabled and client keys unused.

✅ Exercise the configured client API with trust verification enabled. Container readiness checks may use a separate insecure probe, but that does not verify the client under test.

✅ Perform real remote reads after create and update, using fresh credential-store instances or an independent Vault read. A retrieve on the writing instance may hit a credential cache. Verify remote absence after deletion too.

✅ Prove mTLS enforcement: the same listener and trusted server CA must reject a real request when the client key store is omitted. Also retain untrusted-server and untrusted-client controls.

✅ Distinguish transport mTLS from Vault cert login. Token-authenticated requests with a client certificate isolate transport mTLS; token-free tests prove the separate application login mechanism.

✅ Test wrong key-store and trust-store passwords separately using real generated files. Assert the exception category, stable load-failure message and preserved cause, without requiring provider-specific text. These failures need no running server.

The accompanying #98 `HashicorpVaultCredentialStoreTlsPropertiesTestCase` exercises the public path/password properties without supplying an SSLContext. Existing connector and credential-store TLS tests retain token-free authentication and negative trust coverage.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Hostname verification fails | Actual connection host, DNS versus IP SAN, container host overrides |
| Valid-looking client certificate is rejected | Client-auth usage, chain order, and the CA configured on the server |
| Container cannot read its key | Copied file and directory permissions for the runtime user |
| Tests pass despite bad transport setup | Cached reads, missing real requests, supplied SSLContext, disabled verification |
| Intermittent identity mismatches | Static/shared certificate fields or premature directory cleanup |

## Summary Checklist

- [ ] Generate only the identities the test needs; share each identity across its PEM and store outputs.
- [ ] Make roles, trust, SANs, validity, serials, formats and passwords explicit.
- [ ] Clean up on both successful and failed setup/startup.
- [ ] Demonstrate positive TLS behavior and relevant failure controls with real requests.
- [ ] Keep expiring generated files out of Git and verify on the project's supported JDKs.
