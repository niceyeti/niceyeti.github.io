# KMS

### Encryption Review

* In-flight: protects against MITM attacks, since CN of x509 cert must match target.
* Server-side encryption at rest: server receives and encrypts data before storing it, and decrypted before returning it
    1) data key stored in encrypted form, with data
    2) key-mgt service manages the data keys
* Client-side encryption: client encrypts all data, server will never be able to view or decrypt it. Server does not need to know any encryption parameters (per how the data was encrypted).
  * Envelope encryption

### KMS: key mgt service

* AWS manages the encryption keys for you
* Fully integrated with IAM
* Able to audit all api calls to use keys, via CloudTrail
* Seamlessly integrated into most AWS services
* Support through api calls: cli, sdk
  * encrypted keys can be stored in code, etc

Key types:

* symmetric (AES-256): single key for encryption and decryption
  * AWS services integrated with KMS use symmetric keys
  * You can never access the KMS key unencrypted; instead use api's to use the key
* Asymmetric: RSA and ECC
  * public (encrypt) and private (decrypt) pair
  * used for encrypt/decrypt and sign/verify operations
  * public key is downloadable

Other:

* AWS owned keys (free): SSE-S3, SSE-SQS, SSE-DDB.
  * These keys do not need to be created in KMS before using these services; AWS manages them.
* AWS Managed key (free): aws/service-name
* Customer managed keys: created in KMS, $1/month
  * IAM rules for administrators of keys
  * IAM rules for usage of keys
* Customer managed keys imported

Automatic Key rotation

* AWS managed KMS key: automatic every year
* Customer managed KMS key: (must be enable) automatic every year
* Imported KMS key: only manual rotation is possible

Regions: KMS keys are scoped **per region**.

* This is important for encrypted db snapshots: key-A encrypts a db snapshot in region-1. That snapshot is re-encrypted using key-B in region-2, and then restored using key-B.

Key Policies:

* control access to KMS keys, "similar" to S3
* Difference from S3: you cannot control access without them

1) Default KMS Key Policy: created if you don't provide a KMS Key Policy. Complete access to the key to the root user implies complete control of entire AWS account.
2) Custom KMS key policy: define user, roles that can access the KMS key.
    * Define who can administer the key
    * Useful for cross-account access: copy encrypted snapshot across accounts

Use case: copy an encrypted snapshots.

1) create a snapshot encrypted with your own KMS key-A (customer managed)
2) Attach a KMS Key Policy to authorize cross-account access
3) Share the encrypted snapshots
4) Create a copy of the snapshot, encrypt it with a CMK in the destination account
5) Create a volume from the snapshot

### API Decrypt and Encrypt

Encrypt: encrypt api has a 4kb limit; proxies RBAC checks to IAM/
Decrypt: ditto

Envelope Encryption: since the encrypt/decrypt apis have a 4kb limit, use Envelope Encryption for larger sizes using the GenerateDataKey api to encrypt/decrypt locally.

1) Given a 10MB file, call GenerateDataKey, which returns a plaintext DEK (data encryption key)
2) Encrypt the file client-side
3) Add an envelope around file by appending the encrypted DEK.
Decryption is then the inverse:
1) Given envelope of encrypted file and DEK
2) Call the Decrypt api on the encrypted DEK to get the plaintext DEK
3) Decrypt the file using the decrypted DEK

All encryption/decryption occurs client side.
The SDK provides a complete implementation and cli tool.

Data key caching: reuse data keys to reduce the number of calls to KMS. Uses a LocalCryptoMaterialsCache.

Exam: remember these core api terms.

* Encrypt: encrypt up to 4k of data through KMS.
* GenerateDataKey: generates a unique symmetric keys and returns it as plaintext, AND a copy that is encrypted under CMK that you specify (ie envelope encryption)
* GenerateDataKeyWithoutPlaintext: generate a DEK that you plan to use at some point; DEK is encrypted under CMK that you specify.
* Decrypt: decrypt up to 4kb of data (including DEKs for envelope encryption)
* GenerateRandom: returns a random byte string.

### CLI

These commands allow encrypting/decrypting a large file locally, to work around the KMS 4kb size limit.

* use to encrypt:
    `aws-encryption-cli --encrypt --input test.txt --master-keys key=$key --metadata-output metadata/ --output out/`
  * This encrypts data and generates metadata file detailing the data key, encryption context, etc.
  * UNCLEAR: do these commands run api calls? Require network?
* inverse: `aws-encryption-cli --decrypt ...`

### KMS Limits

* ThrottlingExceptions: use exponential backoff in response.
* Cyptographic operations have a quote: any service on our behalf uses this KMS quota (SSE-KMS).
* For GenerateDataKey, consider using DEK caching from the encryptiong SDK:
* Otherwise, request a quota increase

### KMS Lambda Integration

Encrypting environment variables in lambdas with KMS, and decrypting them at run time in code:

* Use env vars to pass secrets; encrypt them using a KMS key.
* Encrypted env vars are decrypted in the code, using boto3 kms client decrypt calls.
  * The console 'encrypt' options will auto-generate decryption code snippets for you to put in your lambda code
  * Note that decryption will increase the require lambda run time
  * Modify policy to allow reading the KMS key

The control mechanism means that data visibility can be controlled through KMS and users/policies.

### KMS S3 Integration

SSE-KMS encryption type reduces cost, api calls, using data keys and an S3 bucket key.

* KMS Customer master key (CKM) is used to generate a rotating bucket key
* Bucket key is used to encrypt objects
* This reduces api calls by placing the key in S3, rather than forcing encryption through KMS.

### Key Policies

Who can access KMS key; default policy allows anyone within your account with IAM permissions to use key, int the policy "Principal" field:

* Specify AWS account and root user
* IAM Roles
* IAM Role Sessions
* IAM users
* Federated user sessions
* AWS services
* All Principals

### Cloud HSM

AWS manages both encryption software and HSM hardware, which supports both asymmetric and symmetric crypto.
Note that Coud HSM is its own service, with more features and security guarantees than KMS.

* Must use client software to use
* Good option to use when using with SSE-C encryption
* cluster based, HA (multi-AZ)
* FIPS 140, level 3
* single tenant (KSM is multitenant)
* KMS keys are per-region; Cloud HSM is VPC-scoped and allows crossing vpc's through peering

Unlike KMS, Cloud HSM software manages users and keys (whereas KMS uses IAM).

KMS integration: in KMS define Cloud HSM as a custom key store.
Thereof, encrypting an RDS instance will call into KMS, get keys from Cloud HSM, and generate auditable usage logs through CloudTrail.

### SSM Parameter Store

Secure storage for configuration and secrets; like a service for ConfigMaps, but with password/secret properties as well.
Serverless, scalable, easy SDK.
Data are versioned, auditable.
Secured through IAM, integrated with CloudFormation to use params in your stacks.
Params can be assigned TTLs, which integrate with EventBridge.
Using these patterns you can easily swith between dev and prod be referencing $DEV_OR_PROD in your parameter paths.

Params are stored in a path-based hierarchy, much like etcd keys:

* /my-department/my-app/dev
  * /db-url
  * /db-password

You can also access secrets-manager secrets through the parameter store on this path:

* /aws/reference/secretsmanager/...
* public params are also available, such as latest AMI key

Example: given a lambda function that has an IAM role it can access the param store.

Parameter policies:

* assign a TTL to force user to delete things like passwords
* also notify via EventBridge if a parameter does NOT change

CLI calls:

* aws ssm get-parameters --names /my-app/dev/db-url ...
* include "--with-decryption" to use KMS permission and decrypt values

SDK calls with boto3 (e.g., for lambda integration):

* configure lambda role with inline policy to allow reading parameters by path on resources/arns of those paths AND allow KMS decrypt
* ssm = boto3.client("ssm")
* db_password = ssm.get_parameters(Names=["/some/path"], WithDecryption=True)

### AWS Secrets Manager

Different from Parameter Store; allows forcing rotation of secrets, and generation of new ones.
Well-integrated with other services.
Mostly meant for RDS/Aurora integration.
Replicate secrets across regions, synced with primary secret.
Pay as you go.
Store credentials per specific target: RDS, Redshift cluster, DocumentDB, etc.

Invoke a lambda function on a rotation interval to generate new username, passwords, etc.

CloudFormation integration: cloud formation template can contain secrets manager params.

* ManageMasterUserPassword: automatically creates a password managed by and rotated by RDS
* Secondly, use a dynamic reference for a secret you manage yourself:
  * generate a secret in your resources
  * create an "instance attachment" (some glue)
  * reference the secret in other resources

### SSM Parameter Store vs. Secrets Manager

Secrets Manager:

* costlier
* automatic rotation of secrets with Lambda
* Lambda function is provided for RDS, Redshift, etc
* KMS encryption is mandatory
* integrates with CloudFormation

SSM Parameter Store:

* simple api
* cheaper
* no secret rotation
* KMS is optional
* integrated with CloudFormation
* can pull secrets from Secrets Manager using the SSM Parameter Store API
* rotation must be accomplished using another service and logic, such as CloudWatch chron events to invoke a lambda to update values

### CloudWatch integration

Encrypt CloudWatch logs with KMS, which is enabled at the Log Group Level.

* you cannot associate a CMK with a log group using CloudWatch console, only the cli api calls:
  * associate-kms-key
  * create-log-group
  * Configure Key-Policy to allow service' Log Group to use key
    * example:

    ```
    aws logs create-log-group --log-group-name /encrypted-test --kms-key-id arn:aws:kms:... --region us-gov-west-1

    ```

### CodeBuild Security

Recall the CodeBuild is outside of your VPC but can be launched within the run code builds.
Secrets:

1) env vars that reference SSM Parameter Store parameters
2) env vars can reference secrets manager secrets

CodeBuild will resolve vars at runtime via their appropriate source: parameter store or secrets manager. CodeBuild must have access to the params.

### Nitro Enclaves

Process highly-sensitive data in an isolated environment: PII, healthcare, etc.
Fully-isolated, hardened, highly-constrained

* not a container, a dedicated machine
* no persistent storage
* no interactive access or external networking

Reduces attack surface for sensitive data processing apps.

* Includes cryptographic attestation of code artifcat
Usecaes: private keys, credit card processing, etc

Setup:

1) launch a compatible Nitro-based EC2 instance with EnclaveOptions=true.
2) Use Nitro cli to convert your app to an Enclave Image
3) Use EIF to create enclave (vm) in EC2 instance
4) The enclave is its own hardened vm in the EC2 instance, with its own kernel, etc. Communicates via a secure local channel to host.
