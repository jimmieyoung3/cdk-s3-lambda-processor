# cdk-s3-lambda-processor

An AWS CDK (Python) project that provisions an S3 bucket, an explicit
bucket policy denying non-TLS requests, and a Python 3.14 Lambda function
that parses single-line files uploaded to the bucket and emits structured
JSON logs to CloudWatch Logs.

## Architecture

![Architecture diagram](docs/architecture.png)

### Flow

1. A client uploads a single-line file to the S3 bucket over HTTPS.
   Non-TLS requests are rejected by the bucket policy with HTTP 403.
2. S3 emits an `s3:ObjectCreated:*` event notification to the Lambda
   function asynchronously.
3. The Lambda function fetches the object with `s3:GetObject`, decodes
   it as UTF-8, parses the first line as comma-separated values, and
   emits a structured JSON log entry containing the parsed fields.
4. CloudWatch Logs retains the log group for 14 days.

### Components

| Component | Purpose |
|---|---|
| `AWS::S3::Bucket` (`ProcessorBucket`) | Stores uploaded files. Block Public Access, SSE-S3 encryption, versioning. |
| `AWS::S3::BucketPolicy` | Explicit `Deny` for any request where `aws:SecureTransport` is `false`. |
| `AWS::Lambda::Function` (`ProcessorFunction`) | Python 3.14, 256 MB, 30 s timeout. Parses the uploaded file. |
| `AWS::IAM::Role` (for the Lambda) | Scoped to `s3:GetObject` on `ProcessorBucket` plus CloudWatch Logs. |
| `AWS::Logs::LogGroup` | 14-day retention for the Lambda's logs. |

## Project structure

```
cdk-s3-lambda-processor/
├── app.py                              # CDK app entry point
├── cdk.json                            # CDK config and feature flags
├── requirements.txt                    # CDK runtime dependencies
├── requirements-dev.txt                # Test dependencies (extends runtime)
├── cdk_s3_lambda_processor/
│   ├── __init__.py
│   └── stack.py                        # The stack definition
├── lambda_src/
│   └── handler.py                      # Lambda handler code
├── tests/
│   ├── conftest.py                     # Adds lambda_src to sys.path
│   ├── test_handler.py                 # Pure-Python tests for parse_line
│   └── test_stack.py                   # CDK template assertions
├── docs/
│   ├── README.md                       # Diagram contents and how to produce
│   ├── architecture.png                # Embedded in this README
│   └── architecture.excalidraw         # Excalidraw source (optional)
├── .github/workflows/ci.yml            # PR + main: pytest + cdk synth
├── .github/workflows/deploy.yml        # Push to main: cdk deploy via OIDC
├── .gitignore
├── LICENSE
├── CLAUDE.md                           # Instructions for Claude Code
└── README.md                           # This file
```

## Prerequisites

You need the following installed locally before deploying:

- **AWS account and credentials.** Configure with `aws configure` or
  `aws sso login`. The credentials must have permission to create
  S3 buckets, Lambda functions, IAM roles, and CloudFormation stacks.
- **Python 3.14.** Match the Lambda runtime.
- **Node.js 20 or newer.** Required only to install the CDK CLI.
- **AWS CDK CLI v2.** Install globally:
  ```bash
  npm install -g aws-cdk@2
  ```
- **CDK bootstrap.** Your target account and region must be CDK-bootstrapped
  once. Skip this if the account is already bootstrapped:
  ```bash
  cdk bootstrap aws://ACCOUNT-NUMBER/REGION
  ```

## Setup

Clone the repo and create a virtual environment:

```bash
git clone https://github.com/YOUR-USERNAME/cdk-s3-lambda-processor.git
cd cdk-s3-lambda-processor

python3.14 -m venv .venv
source .venv/bin/activate          # macOS / Linux
# .venv\Scripts\activate           # Windows PowerShell

pip install --upgrade pip
pip install -r requirements-dev.txt
```

## Deploy

```bash
cdk synth                          # optional: render the CloudFormation template
cdk deploy
```

CDK prints the bucket name, function name, and log group name as stack
outputs once deployment is complete. Capture them for the next step.

## Test the deployed stack

Create a sample single-line file and upload it to the bucket:

```bash
echo "alpha,bravo,charlie,delta" > sample.txt

aws s3 cp sample.txt s3://<BUCKET_NAME_FROM_OUTPUT>/sample.txt
```

View the Lambda's log output:

```bash
aws logs tail <LOG_GROUP_NAME_FROM_OUTPUT> --follow
```

You should see structured JSON entries similar to:

```json
{"event": "invocation_start", "record_count": 1}
{"event": "object_received", "bucket": "...", "key": "sample.txt"}
{"event": "object_parsed", "bucket": "...", "key": "sample.txt", "field_count": 4, "fields": {"field_1": "alpha", "field_2": "bravo", "field_3": "charlie", "field_4": "delta"}}
{"event": "invocation_complete", "processed": 1}
```

To verify the TLS-deny bucket policy, attempt an unencrypted request and
confirm it is denied:

```bash
aws s3api get-object \
    --bucket <BUCKET_NAME_FROM_OUTPUT> \
    --key sample.txt \
    --endpoint-url http://s3.amazonaws.com \
    /tmp/out
# Expected: An error occurred (AccessDenied) when calling the GetObject operation
```

## Run unit tests

```bash
pytest tests/ -v
```

Two test modules run:

- `tests/test_handler.py` covers the `parse_line` function with seven cases
  including quoted fields, whitespace preservation, and empty input.
- `tests/test_stack.py` synthesizes the stack and asserts the CloudFormation
  template contains the expected resources, runtime, encryption,
  public-access block, and bucket policy statement.

## Teardown

```bash
cdk destroy
```

The bucket is configured with `auto_delete_objects=True` and
`RemovalPolicy.DESTROY` so `cdk destroy` removes the bucket and its
contents cleanly. **This is appropriate for an assessment and would be
inverted (`RETAIN`) for production.**

## Design decisions

### Why CDK and not raw CloudFormation

The assessment prompt asks for CDK in the latest supported version of
Python. CDK synthesizes to CloudFormation, so the deliverable is both:
the CDK source in this repo and the CloudFormation template produced by
`cdk synth`. CDK reduces boilerplate, encodes best-practice defaults
(for example, the L2 `Bucket` construct automatically adds an
`AWS::S3::BucketPolicy` whenever any policy statement is attached),
and is easier to evolve than hand-authored YAML.

### Why an explicit bucket policy instead of `enforce_ssl=True`

The CDK `Bucket` construct exposes an `enforce_ssl=True` flag that adds
a TLS-deny statement automatically. The assessment prompt calls out
"a bucket policy" as a distinct deliverable, so this project adds the
statement explicitly via `bucket.add_to_resource_policy()`. Reviewers
can see the policy clearly in `stack.py` rather than inferring it from
a boolean flag.

### Why CSV positional parsing

The prompt says "parse the contents of a single line file" without
specifying a format. CSV via the stdlib `csv` module is the broadest
defensible interpretation: it handles delimited input, correctly parses
quoted fields containing commas, and does not assume a schema. Field
names are positional (`field_1`, `field_2`, ...) because the prompt
provides no domain vocabulary. The parse function is isolated and easy
to replace if the production format turns out to be JSON, key=value,
or fixed-width.

### Why Python 3.14

Python 3.14 is the latest runtime supported by AWS Lambda
([announced November 2025](https://aws.amazon.com/blogs/compute/python-3-14-runtime-now-available-in-aws-lambda/)).
The prompt asks for the latest supported Python version, so 3.14 is
used for both the CDK app and the Lambda runtime. The managed runtime
does not support disabling the GIL or enabling the experimental JIT,
which is not relevant to this workload.

### Why `auto_delete_objects=True`

Lets the reviewer run `cdk destroy` without first emptying the bucket
manually. CDK implements this with a small custom-resource Lambda that
runs at delete time. In production, `auto_delete_objects` would be
`False` and the bucket's `RemovalPolicy` would be `RETAIN` to prevent
accidental data loss.

### Why a single stack

The workload is small and self-contained. Splitting into multiple stacks
(for example, a separate `BucketStack` and `LambdaStack`) would add
cross-stack references and deployment ordering complexity without
material benefit at this scale.

## Security

What is enforced today:

- All public access to the bucket is blocked at the bucket level
  (`BlockPublicAccess.BLOCK_ALL`).
- SSE-S3 (AES-256) server-side encryption is applied to every object.
- All non-TLS requests to the bucket are denied by the bucket policy.
- The Lambda's IAM role grants `s3:GetObject` on the bucket's ARN only.
  No wildcard resources, no cross-account principals.
- CloudWatch log retention is set to 14 days so logs do not accumulate
  indefinitely.

## Cost

For the volumes typical of an assessment review (a few uploads, a few
Lambda invocations), this stack stays well within the AWS Free Tier
across S3, Lambda, and CloudWatch Logs. There are no provisioned
resources that incur idle cost; everything is pay-per-use or
free-tier-covered.

## Future improvements

Items deliberately omitted to keep the assessment scoped. Each is a
talking point rather than a missing requirement.

- **Dead-letter queue.** Add an SQS DLQ to capture Lambda invocations
  that fail all retries.
- **EventBridge fan-out.** Replace direct S3 to Lambda notifications
  with S3 to EventBridge, allowing multiple consumers and richer event
  filtering without re-deploying the bucket.
- **SSE-KMS encryption.** Customer-managed KMS key for additional access
  control and auditability, at the cost of KMS API charges.
- **cdk-nag.** Add `cdk-nag` to the CDK app and the CI pipeline to
  enforce AWS Well-Architected security rules at synth time.
- **Multi-environment configuration.** Parameterize the stack on
  environment (dev / staging / prod) via `cdk.json` context or CDK
  `Stage` constructs.
- **Lambda DLQ destination and on-failure destination.** Distinct from
  function-level DLQ. Configure async invocation failure destinations.
- **AWS X-Ray tracing.** Enable active tracing on the Lambda for
  request-level latency breakdown.
- **Structured logging library.** Adopt `aws-lambda-powertools` for
  structured logging, tracing, and metrics in one library.

## Troubleshooting

**`cdk: command not found`.** Install the CDK CLI with
`npm install -g aws-cdk@2`.

**`This stack uses assets, so the toolkit stack must be deployed`.**
Run `cdk bootstrap aws://ACCOUNT/REGION` once for the target account
and region.

**Lambda invocation fails with `AccessDenied` on `GetObject`.** Confirm
the Lambda's IAM role was deployed with the latest stack version. The
`bucket.grant_read(processor)` call attaches the policy; if you renamed
the bucket logical ID, redeploy.

**Bucket policy blocks an upload.** All requests must be HTTPS. The AWS
CLI uses HTTPS by default; tools and SDKs that force HTTP will be
denied. This is expected behavior.

**`cdk destroy` fails to remove the bucket.** Only happens if
`auto_delete_objects` was changed to `False` after deploy. Empty the
bucket manually (`aws s3 rm s3://BUCKET --recursive`) and retry.
