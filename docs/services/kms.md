# KMS

**Protocol:** JSON 1.1 (`X-Amz-Target: TrentService.*`)
**Endpoint:** `POST http://localhost:4566/`

## Supported Actions

<!-- floci:actions:start -->
| Action | Description |
| --- | --- |
| `CreateKey` | Create a new KMS key |
| `GenerateRandom` | Generate random bytes |
| `GetPublicKey` | Get public key material for asymmetric keys |
| `DescribeKey` | Get key metadata |
| `ListKeys` | List all keys |
| `CreateGrant` | Create a grant for a KMS key |
| `ListGrants` | List grants for a KMS key |
| `ListRetirableGrants` | List grants retirable by a principal |
| `RevokeGrant` | Revoke (administratively delete) a grant |
| `RetireGrant` | Retire a grant (token- or key+grant-based) |
| `Encrypt` | Encrypt plaintext with a key |
| `Decrypt` | Decrypt ciphertext |
| `ReEncrypt` | Re-encrypt under a different key |
| `GenerateDataKey` | Generate a data key (plaintext + encrypted) |
| `GenerateDataKeyWithoutPlaintext` | Generate only the encrypted data key |
| `Sign` | Sign a message with an asymmetric key |
| `Verify` | Verify a signature |
| `GenerateMac` | Generate a MAC with an HMAC key |
| `VerifyMac` | Verify a MAC with an HMAC key |
| `CreateAlias` | Create a friendly name for a key |
| `UpdateAlias` | Repoint an alias at a different key |
| `DeleteAlias` | Remove an alias |
| `ListAliases` | List all aliases |
| `ScheduleKeyDeletion` | Mark a key for deletion |
| `CancelKeyDeletion` | Cancel pending deletion |
| `TagResource` | Tag a key |
| `UntagResource` | Remove tags |
| `ListResourceTags` | List tags |
| `GetKeyPolicy` | Get a key's resource policy |
| `PutKeyPolicy` | Update a key's resource policy |
| `ListKeyPolicies` | List a key's policy names (always the single `default` policy) |
| `UpdateKeyDescription` | Update a key's description |
| `GetKeyRotationStatus` | Check if automatic key rotation is enabled |
| `EnableKeyRotation` | Enable automatic key rotation (symmetric keys only) |
| `DisableKeyRotation` | Disable automatic key rotation |
| `EnableKey` | Enable a key |
| `DisableKey` | Disable a key |
| `RotateKeyOnDemand` | Rotate key material on demand (symmetric keys only) |
<!-- floci:actions:end -->

## Asymmetric Encryption (RSA)

RSA keys created with `KeyUsage=ENCRYPT_DECRYPT` (`RSA_2048`, `RSA_3072`, `RSA_4096`)
use real RSA OAEP encryption, matching AWS:

- `Encrypt` returns raw RSA OAEP ciphertext (one modulus long), exactly like AWS. The
  `EncryptionAlgorithm` request field accepts `RSAES_OAEP_SHA_256` (the default when
  omitted) and `RSAES_OAEP_SHA_1`, and is echoed back in the response.
- `Decrypt` accepts ciphertext produced by floci `Encrypt` and ciphertext produced
  externally with the public key returned by `GetPublicKey` (for example with
  `openssl pkeyutl` or the AWS Encryption SDK). When `KeyId` is omitted, floci tries
  the RSA keys in the region until one unwraps the ciphertext, mirroring AWS key
  resolution. When `EncryptionAlgorithm` is omitted, both OAEP digests are tried.
- An `EncryptionContext` is bound into RSA ciphertext as the OAEP label (the ASN.1
  DER encoding of the sorted context map, the same encoding the AWS Encryption SDK
  uses). Decryption with a different context fails with `InvalidCiphertextException`.
- `Encrypt`/`Decrypt` on an RSA `SIGN_VERIFY` key fail with `InvalidKeyUsageException`,
  and an RSA encryption algorithm on a symmetric key fails with
  `InvalidEncryptionAlgorithmException`.

Symmetric keys (`SYMMETRIC_DEFAULT`) keep floci's envelope ciphertext format: an
opaque, self-describing blob that binds the key id and encryption context. Values
encrypted with an floci symmetric key are not intended to interoperate with real AWS
ciphertext, but round-trip semantics (including encryption context and key state
checks) match AWS behavior.

See `scripts/test-kms-asymmetric-crypto` for an end-to-end demonstration using the
AWS CLI and `openssl`.

## Grant Support Scope

Grant lifecycle operations (`CreateGrant`, `ListGrants`, `ListRetirableGrants`, `RevokeGrant`, `RetireGrant`) are supported. However, grant lifecycle support **does not** imply grant-based authorization enforcement on cryptographic operations (`Encrypt`, `Decrypt`, `Sign`, `Verify`, `GenerateDataKey`, etc.). Grants are stored and queryable but are not evaluated during crypto operations.

## Configuration

| Variable | Default | Description |
|---|---|---|
| `FLOCI_SERVICES_KMS_ENABLED` | `true` | Enable or disable the service |

## Examples

```bash
export AWS_ENDPOINT_URL=http://localhost:4566

# Create a symmetric key
KEY_ID=$(aws kms create-key \
  --description "My encryption key" \
  --query KeyMetadata.KeyId --output text \
  --endpoint-url $AWS_ENDPOINT_URL)

# Create an alias
aws kms create-alias \
  --alias-name alias/my-key \
  --target-key-id $KEY_ID \
  --endpoint-url $AWS_ENDPOINT_URL

# Encrypt
CIPHER=$(aws kms encrypt \
  --key-id alias/my-key \
  --plaintext "Hello, World!" \
  --query CiphertextBlob --output text \
  --endpoint-url $AWS_ENDPOINT_URL)

# Decrypt
aws kms decrypt \
  --ciphertext-blob $CIPHER \
  --query Plaintext --output text \
  --endpoint-url $AWS_ENDPOINT_URL | base64 --decode

# Generate a data key (envelope encryption)
aws kms generate-data-key \
  --key-id alias/my-key \
  --key-spec AES_256 \
  --endpoint-url $AWS_ENDPOINT_URL

# Asymmetric: create an RSA key and encrypt with the public key locally
RSA_KEY_ID=$(aws kms create-key \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec RSA_2048 \
  --query KeyMetadata.KeyId --output text \
  --endpoint-url $AWS_ENDPOINT_URL)

aws kms get-public-key \
  --key-id $RSA_KEY_ID \
  --query PublicKey --output text \
  --endpoint-url $AWS_ENDPOINT_URL | base64 --decode > /tmp/kms-pub.der

openssl pkey -pubin -inform DER -in /tmp/kms-pub.der -out /tmp/kms-pub.pem
echo -n "encrypted outside floci" | openssl pkeyutl -encrypt \
  -pubin -inkey /tmp/kms-pub.pem \
  -pkeyopt rsa_padding_mode:oaep -pkeyopt rsa_oaep_md:sha256 \
  -out /tmp/kms-cipher.bin

# floci decrypts ciphertext that was encrypted with the public key
aws kms decrypt \
  --ciphertext-blob fileb:///tmp/kms-cipher.bin \
  --encryption-algorithm RSAES_OAEP_SHA_256 \
  --query Plaintext --output text \
  --endpoint-url $AWS_ENDPOINT_URL | base64 --decode
```
`CreateKey` also accepts a reserved creation-time tag key, `floci:override-id`, when tests need a deterministic `KeyId`. Floci uses the tag value as the created key id, strips the reserved tag from stored resource tags, and rejects attempts to add `floci:*` tags later via `TagResource`.
