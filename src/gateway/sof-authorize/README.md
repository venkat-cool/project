# Apigee SMART on FHIR Proxy - sof-authorize

This Apigee Proxy validates JWT tokens Or Opaque Access tokens from the configured Issuer depending upon configuration.
After token validation, the proxy calls configured FHIR Store with appropriate headers as prescribed by the FHIR Store and the response from the FHIR Store is returned back to the caller.

## Setup Instructions

[Step 1: Pre-requisites](#Pre-requisites)  
[Step 2: Configuration](#Configuration)  
[Step 3: Install Process](#Install-process)  
[Step 4: Usage](#Usage)

## Pre-requisites

1. Java 8 or later version should be installed (required for maven installation)  
1. Maven 3.2.0 or greater should be installed

## Configuration

### A) KVM Configurations

Following Key Value Maps hold configuration parameters.  
These KVM entries are present in 'config/env/{envName}/kvms.json'.  
Depending upon your environments you might have to rename these folders under config/env directory and use corresponding <apigee.profile> in 'profile' tag in pom.xml. More on apigee maven plugin and configurations can be found [here](https://github.com/apigee/apigee-deploy-maven-plugin#note-1)

#### KVM - SoF_Config

1. VERIFY_ACCESS_TOKEN  
Access token to the proxy may be a JWT or opaque token. Accepts one of 3 values: "JWT" (default), "introspect", "userinfo". Set to either "introspect" or "userinfo" for opaque access tokens depending on where the IDP provides the appropriate claims for the proxy to include. e.g.  
``
VERIFY_ACCESS_TOKEN: introspect
``

1. ALLOWED_PATH_PREFIX  
A comma-delimited set of Healthcare API paths to proxy. This provides the ability to limit which projects, datasets etc. with which the proxy may be used. Each entry must start with "/" e.g.  
``
ALLOWED_PATH_PREFIX: /v1/projects/p1/locations/us-central1/datasets/a/fhirStores/f1, /v1/projects/p1/locations/us-central1/datasets/a/fhirStores/f2,/v1beta1/projects/healthapix/locations/us-central1/datasets/hc-dataset-dev/fhirStores/r4
``

1. AUDIENCE  
A string that represents the expected audience for the access token. It may be set to the same string as the Token Issuer's client ID depending on how the Token Issuer populates the claim. e.g.  
``
AUDIENCE: some-client-id-of-customer
``

1. DISABLE_AUDIT_LOG  
Do not emit additional audit log information that is extracted from the SMART-on-FHIR token or the token issuer's endpoints. Default is FALSE. Set to TRUE when wanting to reduce the PII/PHI related to the user, scopes, etc related to requests. e.g.
``
DISABLE_AUDIT_LOG: false
``

1. FHIR_ISSUER  
The "iss" claim expected in the SMARTonFHIR token. e.g.  
``
FHIR_ISSUER: https://healthapix-dev-dev.apigee.net/v1/r4/R4_bd836872-4fac-425e-ac3d-d88c333214a2
``

1. FHIR_ISSUER_CLIENT_ID  
The Relying Party client ID for the proxy as allocated by the token issuer IDP. This may be used when the proxy calls OIDC endpoints. Corresponding secret has been registered in the encrypted KVM SoF_Secrets. e.g.  
``
FHIR_ISSUER_CLIENT_ID: `<client-id>`
``

1. IS_CLIENT_REGISTRATION_ENABLED  
When "true". expects clients of the proxy to be registered. Client ID and Client Secret headers must be present and will be validated by the proxy. If not "true" and a Client ID and/orClient-Secret headers are present, the proxy does not return an error. e.g.  
``
IS_CLIENT_REGISTRATION_ENABLED: false
``

1. JWKS_CACHE_TIMEOUT_SECONDS  
In case of JWT token, the JWKS response is cached for this number of seconds. e.g.  
``
JWKS_CACHE_TIMEOUT_SECONDS: 900
``

1. MAX_HEADER_SIZE_BYTES  
Limits the length of the SMARTonFHIR headers passed along to the FHIR Server to not exceed this limit, otherwise truncation will occur. e.g.  
``
MAX_HEADER_SIZE_BYTES: 1024
``

1. MAX_TOKEN_CACHE_SECONDS  
Enables caching successful token status response (JWT or Opaque) in the proxy and responds with same status response withing cache expiry time. Caching time could be less than the configured time if the token expiry is less. e.g.  
``
MAX_TOKEN_CACHE_SECONDS: 600
``

1. REMOVE_SCOPES  
A comma-delimited set of scopes to remove from the X-Authorization-Scope header that the proxy includes in its requests to downstream servers. e.g.  
``
REMOVE_SCOPES: openid,profile,email
``

1. SERVICE_ACCOUNT_NAME  
The service account name for the proxy to use when calling the Target FHIR endpoints. Default is "fhirproxy". e.g.  
``
SERVICE_ACCOUNT_NAME: fhirproxy
``

1. SOF_CLAIM_CONFIG  

- CLAIM_TOKEN_ID:  
Name of the OPTIONAL token identifier claim,
- CLAIM_FHIRUSER: name of the OPTIONAL principal user claim,
- CLAIM_EMAIL: name of the OPTIONAL principal's email claim,
- CLAIM_SUBJECT: name of the REQUIRED subject claim,
- CLAIM_PROFILE: name of the OPTIONAL profile claim,
- CLAIM_PATIENT: name of the OPTIONAL patient claim,
- CLAIM_SCOPE: name of the REQUIRED scope claim)  
e.g.  
``
SOF_CLAIM_CONFIG: {"CLAIM_PATIENT":"patient", "CLAIM_SCOPE":"scope", "CLAIM_FHIRUSER":"fhirUser", "CLAIM_EMAIL":"email", "CLAIM_SUBJECT":"subject", "CLAIM_TOKEN_ID":"id", "CLAIM_PROFILE":"profile"}
``

14. SPIKEARREST_RATELIMIT  
Rate limits configuration. e.g.  
``
SPIKEARREST_RATELIMIT: 20ps
``

1. KVM - SoF_Secrets  
(values need to be entered manually)

- FHIR_ISSUER_CLIENT_SECRET: Client Secret for ClientID mentioned in the SoF_Config->FHIR_ISSUER_CLIENT_ID entry.
``Note: Needs to be entered manually in the KVM after maven deployment``.

- FHIR_ISSUER_CLIENT_SECRET: `<client-secret-corresponding-to-client-id>`

16. fhirproxy  
Name of the service account as mentioned in the SoF_Config->SERVICE_ACCOUNT_NAME entry. It holds the info in service account key file to connect to GCP FHIR Store.
``Note: Needs to be configured manually in the KVM after maven deployment`` e.g.  

``
fhirproxy: `<content of service_account_key json file>`
``

### B) Manual KVM Configurations

The values in KVM SoF_Secrets need to be entered manually to avoid storage of Keys Or ClientSecrets in code base.
Maven deployment creates the keys in the KVM but uses dummy values. These dummy values need to be replaced with actual Client_Secret in case of key 'FHIR_ISSUER_CLIENT_SECRET' and service account key file details in case of 'fhirproxy' key.

## Install process

### A) Login to a machine where Java and maven is installed as per pre-requisites section above. Instructions mentioned below are baselined to Debian 9 on a GCP VM

```bash
# Install Git if needed
git --version

sudo apt-get -y install git

cd ~

# Clone this (native-sof) git repository to your local machine. 
# You may need manually generated credentials.
# You may generate and store your Git credentials using Google Cloud Source Repositories.

# Change current directory to git repo home
cd ~/native-sof

# Change to sof-authorize folder where pom.xml is present
cd src/gateway/sof-authorize

# Run maven command 
mvn install -P<profile-name> -Dusername=<your-apigee-username> -Dpassword=<your-apigee-password> -Dapigee.config.dir=<path-to-config-dir> -Dapigee.config.options=create

#Example when running from ~/native-sof/src/gateway/sof-authorize
mvn install -P<profile-name> -Dusername=<your-apigee-username> -Dpassword=<your-apigee-password> -Dapigee.config.dir=../../config -Dapigee.config.options=create
```

## Usage

The proxy can be used with JWT Tokens and Opaque Access Tokens.
These tokens are issued by the Issuer as configured in the KVM entry -> FHIR_ISSUER. The tokens then are used in the Authorization header to this native-sof proxy. Caller must use Issue specified mechanism to obtain these tokens.  
Examples of the API calls are given below, one each when a JWT token is used and Opaque token is used.

- Native sof proxy call with JWT token

```bash
curl -X POST \
  https://healthapix-dev-dev.apigee.net/sof/v1beta1/projects/dacfhir/locations/us-central1/datasets/hc-dataset-dev/fhirStores/r4/fhir/Patient/cc6f4f3b-dd80-423c-aa2a-09f78b1eae4e \
  -H 'Authorization: Bearer eyJqa3UiOiJodHRwczpcL1wvaGVhbHRoYXBpeC1kZXYtZGV2LmFwaWdlZS5uZXRcL29hdXRoXC92MlwvLndlbGwta25vd25cL2p3a3MuanNvbiIsImtpZCI6InJ1bm1jbGUxZ2g3bW1zNGVwaCIsInR5cCI6IkpXVCIsImFsZyI6IlJTMzg0In0.eyJzdWIiOiJzaGFzaGFuay5taXNocmEwMUBnbWFpbC5jb20iLCJhdWQiOiJodHRwczpcL1wvaGVhbHRoYXBpeC1kZXYtZGV2LmFwaWdlZS5uZXRcL3YxXC9yNFwvUjRfYmQ4MzY4NzItNGZh' \
 
 ```

- Native sof proxy call with Opaque access token

 ```bash
 curl -X POST \
  https://healthapix-dev-dev.apigee.net/sof/v1beta1/projects/healthapix/locations/us-central1/datasets/hc-dataset-dev/fhirStores/r4/fhir/Patient/cc6f4f3b-dd80-423c-aa2a-09f78b1eae4e \
  -H 'Authorization: Bearer omBvHGTffZnYgggGGva1IxomJkVj'
  
  ```
  