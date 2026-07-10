# Introduce Shared Auth Module for AWS Connectors

- Authors
  - Dimuthu Madushan
- Reviewed by
  - Danesh Kuruppu
- Created date 
  - 2026-07-10
- Updated date
  - 2026-07-10
- Issue
  - [#1456](https://github.com/ballerina-platform/ballerina-spec/issues/1456)
- State
  - Submitted

## Summary

Every Ballerina AWS connector currently implements AWS Signature Version 4 (SigV4) signing and credential handling independently. This proposal introduces a shared package, **ballerinax/aws.auth**, that provides common authentication, credential management, SigV4 signing, and region/endpoint resolution. Existing and future AWS connectors will reuse this module instead of maintaining their own implementations, following the shared-core architecture used by official AWS SDKs.

## Motivation

The AWS connectors have evolved independently, leading to several issues:

1. **Inconsistent authentication:** Most connectors only support static access keys, with varying credential support across connectors.  
2. **Duplicated signing logic**: SigV4 signing is implemented separately in multiple connectors, making bug fixes repetitive and error-prone.  
3. **Inconsistent configuration:** Common settings use different field names and formats, resulting in an inconsistent developer experience.  
4. **Limited endpoint support:** Connectors hardcode standard AWS endpoints, preventing support for China regions, FIPS, dual-stack, custom endpoints, and LocalStack.  
5. **High maintenance cost**: Every authentication improvement or new feature must be implemented separately for each connector.

All official AWS SDKs solve these problems by sharing a common authentication and signing core across all service clients. Ballerina already follows this approach for OAuth2 and authentication, so AWS connectors should adopt the same shared foundation.

## Goals

* Provide a single, shared implementation of AWS SigV4 request signing.  
* Provide a long-lived `CredentialProvider` that caches resolved credentials and transparently refreshes expiring ones.  
* Provide an endpoint resolution utility that correctly handles all AWS partitions.

## Non-Goals

- **Not an end-user-facing auth API.** End users continue to configure each connector's `ConnectionConfig` as today.  
- **No interactive login flows.** SSO support consumes an existing session created with `aws sso login`: no browser or device-code flow is implemented.  
- **No SigV4A (asymmetric, multi-region) signing** in the initial release. The API leaves room to add it later.

## Design

### 1. Module overview

`The ballerinax/aws.auth` sits behind the connectors. End users keep configuring each connector's `ConnectionConfig` and connectors internally delegate credential resolution, signing, and endpoint construction to the shared module. The credential layer wraps the official AWS SDK v2 artifacts rather than reimplementing the provider chain.

### 2. Configuration types

#### 2.1 The `AuthConfig` union

One union covers every standardized AWS credential source — six explicitly configurable sources plus the default provider chain:

```ballerina
# Represents the authentication configuration for AWS
public type AuthConfig StaticAuthConfig      // explicit keys
    | ProfileAuthConfig                      // ~/.aws/credentials profile
    | AssumeRoleConfig                       // STS assume-role; composable via recursive sourceCredentials
    | WebIdentityConfig                      // EKS IRSA, CI OIDC
    | SsoAuthConfig                          // IAM Identity Center
    | ProcessAuthConfig                      // credential_process / Roles Anywhere
    | DEFAULT_CREDENTIALS;                   // full default provider chain

# Instructs the credential provider to resolve credentials via the AWS default
# credential provider chain
public const DEFAULT_CREDENTIALS = "DEFAULT_CREDENTIALS";
```

#### 2.2 Credential source records

```ballerina
# Represents static AWS credentials.
public type StaticAuthConfig record {|
    # AWS access key ID
    string accessKeyId;
    # AWS secret access key
    string secretAccessKey;
    # AWS session token, required only for temporary credentials
    string sessionToken?;
|};


# Represents AWS credentials loaded from a named profile in a local AWS
# credentials file (as created by `aws configure`).
public type ProfileAuthConfig record {|
    # The named profile to use
    string profileName = "default";
    # Path to the AWS credentials file
    string credentialsFilePath = "~/.aws/credentials";
|};


# Represents temporary credentials obtained by assuming an IAM role via AWS STS.
# The base identity used to call STS is itself an `AuthConfig`, so a role can
# be assumed from any other credential source (including another assume-role).
public type AssumeRoleConfig record {|
    # ARN of the IAM role to assume
    string roleArn;
    # Identifier for the assumed-role session
    string roleSessionName = "ballerina-aws-auth";
    # External ID for third-party cross-account trust, if the role requires one
    string externalId?;
    # Validity period of each assumed-role session
    int durationSeconds = 3600;
    # Region of the STS endpoint to call
    string stsRegion = "us-east-1";
    # Credential source used to authenticate the AssumeRole call
    AuthConfig sourceCredentials = DEFAULT_CREDENTIALS;
|};


# Represents temporary credentials obtained by exchanging a web identity (OIDC)
# token for an IAM role via AWS STS
public type WebIdentityConfig record {|
    # ARN of the IAM role to assume
    string roleArn;
    # Path to the file containing the OIDC token (JWT)
    string webIdentityTokenFile;
    # Identifier for the assumed-role session
    string roleSessionName = "ballerina-aws-auth";
|};


# Represents credentials obtained from an AWS IAM Identity Center (SSO) session.
# Requires an active session created out-of-band with `aws sso login`.
public type SsoAuthConfig record {|
    # The Identity Center start URL
    string ssoStartUrl;
    # Region in which IAM Identity Center is configured
    string ssoRegion;
    # AWS account ID to get credentials for
    string accountId;
    # Permission-set role name to get credentials for
    string roleName;
|};


# Represents credentials supplied by an external process (the AWS
# `credential_process` contract). The command must print a JSON credential
# document to stdout.
public type ProcessAuthConfig record {|
    # The command to execute
    string command;
|};
```

#### 2.3 Resolved credentials

```ballerina
# Represents resolved AWS credentials, as returned by the `CredentialProvider`
public type Credentials record {
    # AWS access key ID
    string accessKeyId;
    # AWS secret access key
    string secretAccessKey;
    # Session token, present only for temporary credentials
    string sessionToken?;
};
```

### 3. The `CredentialProvider`

A long-lived provider object wrapping the corresponding AWS SDK v2 credential provider for the given `AuthConfig`. The SDK layer caches resolved credentials and refreshes expiring ones (STS, SSO, instance profile) automatically.

```ballerina
# A long-lived AWS credential provider with caching and automatic refresh.
public isolated class CredentialProvider {

    # Initializes the credential provider for the given configuration.
    #
    # + config - The credential source configuration
    # + return - An `auth:Error` if the configuration is invalid, or `()`
    public isolated function init(AuthConfig config) returns Error?;

    # Returns currently-valid credentials for signing a request.
    #
    # + return - The resolved `Credentials`, or an `auth:Error` if the
    #            configured source cannot supply credentials
    public isolated function getCredentials() returns Credentials|Error;
}
```

### 4. SigV4 signing

#### 4.1 `SignatureRequest`

```ballerina
# Describes an HTTP request to be signed with AWS Signature Version 4.
public type SignatureRequest record {
    # HTTP method in upper case
    string method;
    # Target host
    string host;
    # Absolute request path, unencoded
    string path = "/";
    # Query parameters, unencoded names and values
    map<string> queryParams = {};
    # Additional headers to include in the signature (e.g. `content-type`).
    # `host` and `x-amz-date` are always signed and need not be given
    map<string> headers = {};
    # Request body: use an empty array for bodiless requests
    byte[] payload = [];
    # When `true`, signs the payload as `UNSIGNED-PAYLOAD` (S3 streaming)
    boolean unsignedPayload = false;
    # When `true`, uses S3 path semantics: path segments are URI-encoded once
    # and not normalized. All other services require double-encoding (the default)
    boolean s3PathMode = false;
    # When `true`, `x-amz-security-token` is included in the canonical (signed)
    # headers. Most services expect the token added after signing (the default)
    boolean signSessionToken = false;
};
```

The three boolean flags expose the officially documented per-service variations of SigV4 (S3's non-normalized single-encoded paths, `UNSIGNED-PAYLOAD` streaming, and session-token placement) without service-specific code paths in the connectors.

#### 4.2 Signing functions

```ballerina
# Signs a request with AWS Signature Version 4 and returns the headers to set
# on the outbound HTTP request: `Authorization`, `X-Amz-Date`
#
# + req - The request to sign
# + credentials - Credentials obtained from a `CredentialProvider` or statically
# + region - Target region code, e.g. `us-east-1`
# + serviceName - Signing name of the target service, e.g. `s3`, `sns`, `dynamodb`
# + return - Headers to set on the outbound request, or a `SigningError`
public isolated function getSignedHeaders(SignatureRequest req, Credentials credentials,
        string region, string serviceName) returns map<string>|SigningError;
```

The returned map contains exactly the headers the caller must set on the outbound request.

### 5. Region type and endpoint resolution

#### 5.1 `Region`

One shared region enum.

```ballerina
# AWS regions. Shared by all AWS connectors so the region is typed
# consistently across the ecosystem.
public enum Region {
    AF_SOUTH_1 = "af-south-1",
    AP_EAST_1 = "ap-east-1",
    // ... all current AWS regions, including cn-* and us-gov-* ...
    US_EAST_1 = "us-east-1",
    US_WEST_2 = "us-west-2"
}
```

Every API that takes a region accepts `Region|string`, so regions launched after a given release of the module remain usable without a library update.

#### 5.2 Endpoint resolution

```ballerina
# Endpoint resolution options.
public type EndpointConfig record {
    # Use the FIPS 140-validated endpoint variant
    boolean fips = false;
    # Use the dualstack (IPv4/IPv6) endpoint variant
    boolean dualstack = false;
    # Full endpoint URL override
    string customEndpoint?;
};

# Resolves the HTTPS endpoint URL for an AWS service in a region
public isolated function resolveEndpoint(string serviceName, Region|string region,
        EndpointConfig config = {}) returns string;

# Resolves only the host part of the endpoint, for callers that construct URLs
# themselves or need the host for request signing,
public isolated function resolveEndpointHost(string serviceName, Region|string region,
        EndpointConfig config = {}) returns string;
```

### 6. Errors

```ballerina
# Base error type
public type Error distinct error;

# Returned when the configured credential source fails to yield credentials
public type CredentialResolutionError distinct Error;

# Returned when a request cannot be signed.
public type SigningError distinct Error;
```

### 7. Usage

#### 7.1 The `ConnectionConfig` record

The `ConnectionConfig` record will be updated with following fields in each connector.

```ballerina
import ballerinax/aws.auth;

public type ConnectionConfig record {
    auth:AuthConfig authConfig;
    auth:Region|string region;
    auth:EndpointConfig endpoint?;
};
```

#### 7.2 Inside an `http:Client`-based connector

An `http:Client`-based connector replaces its internal signing with the shared provider, signer, and endpoint resolver:

```ballerina
import ballerinax/aws.auth;

public isolated client class Client {
    private final http:Client httpClient;
    private final auth:CredentialProvider credentialProvider;
    private final string region;

    public isolated function init(ConnectionConfig config) returns Error? {
        // One provider per client; resolution + refresh handled by aws.auth
        self.credentialProvider = check new (config.auth);
        self.region = config.region;
        string endpoint = auth:resolveEndpoint("sns", config.region, config.endpoint ?: {});
        self.httpClient = check new (endpoint);
    }

    remote isolated function publish(...) returns ...|Error {
        // Fetch per request — cached while fresh, refreshed when expiring
        auth:Credentials creds = check self.credentialProvider.getCredentials();
        map<string> signedHeaders = check auth:getSignedHeaders({
            method: "POST",
            host: auth:resolveEndpointHost("sns", self.region),
            path: "/",
            headers: {"content-type": "application/x-www-form-urlencoded"},
            payload
        }, creds, self.region, "sns");
        // set signedHeaders on the outbound request and send ...
    }
}
```

#### 7.3 Inside an SDK-client-based connector

A connector that wraps an AWS SDK service client (e.g. `SqsClient`) integrates one level lower. The SDK client takes an `AwsCredentialsProvider` **object** at build time and internally calls `resolveCredentials()` on every request before signing with its own `AwsV4HttpSigner`.

```java
// Connector native code
AwsCredentialsProvider provider = ProviderFactory.buildProvider(bAuthConfig); // from aws.auth
SqsClient sqs = SqsClient.builder()
        .credentialsProvider(provider)
        .region(Region.of(region))
        .build();
```

Because the *provider object* lives inside the SDK client for the client's lifetime, credential caching and auto-refresh (STS, SSO, instance profile) will handle itself.

#### 7.4 The resulting end-user experience (via connectors)

After migration, every connector's `ConnectionConfig` exposes the same `auth` field typed by the shared union.

```ballerina
// Production on EKS/EC2 — no keys anywhere
s3:Client s3 = check new ({auth: s3:DEFAULT_CREDENTIALS, region: "us-east-1"});

// Cross-account access via STS assume-role — same shape in every connector
sqs:Client sqs = check new ({
    auth: {
        roleArn: "arn:aws:iam::123456789012:role/reader",
        sourceCredentials: sqs:DEFAULT_CREDENTIALS
    },
    region: "us-east-1"
});

// Local development against LocalStack
dynamodb:Client db = check new ({
    auth: {accessKeyId: "test", secretAccessKey: "test"},
    region: "us-east-1",
    endpoint: {customEndpoint: "http://localhost:4566"}
});
```

```shell
# Developer laptop
[myapp.awsAuth]
profileName = "dev"

# Production — nothing: DEFAULT_CREDENTIALS picks up the EKS/EC2 role
```

## Alternatives

* **Revamp each connector in place, keeping auth internal but consistent.** Rejected: consistency by copy-paste decays, it must be re-enforced by discipline on every change, and It also contradicts the architecture of every official AWS SDK.  
* **End-user-facing auth API as the primary interface** (user imports `aws.auth` and signs requests manually). Rejected: a two-step flow with no representation in WSO2 Integrator's connection-creation experience; would regress the low-code story.

## Testing

Following unit tests will be added to the auth module which verifies the credential provide functionalities, signer and endpoint resolution.

1. Credential provider unit tests
2. SigV4 test
3. Endpoint resolution tests

## Dependencies

* AWS SDK for Java v2

## References

* [AWS Signature Version 4 signing process](https://docs.aws.amazon.com/IAM/latest/UserGuide/create-signed-request.html)  
* [AWS credential provider chain](https://docs.aws.amazon.com/sdkref/latest/guide/standardized-credentials.html)  
* [AWS SDK for Java v2 — credential providers](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/credentials-chain.html)  
* [Ballerina Central — AWS connectors](https://central.ballerina.io/search?q=aws)
