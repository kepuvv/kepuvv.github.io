## Why?

Recently, Oracle updated the list of resources available for the Oracle Cloud Free Tier Always Free program (of course, reducing available value).

However, OCI still remains an attractive option for using cloud computing resources.

Clicking through the UI to create three virtual machines included in the free tier is not particularly time-consuming, but as a DevOps engineer, I am always looking for ways to automate tasks and minimize the risk of manual errors.
For infrastructure deployment, I chose Terraform and Ansible, which are widely used by engineers.

## Requirements

For this deployment, you will need:

- Terraform
- OCI CLI
- Git
- Oracle Cloud account
- Ansible (optional)
- jq

We will use the Terraform project template:

https://github.com/kepuvv/oci-free-tier-terraform

## Install and configure OCI CLI

On macOS, install it using Homebrew:

```bash
brew install oci-cli
```

Installation instructions for other environments are available in the official documentation:

https://docs.oracle.com/en-us/iaas/private-cloud-appliance/pca/installing-the-oci-cli.htm

After installation, configure access using API keys.

The latest instructions for adding API signing keys can be found in the official Oracle Cloud documentation:

https://docs.oracle.com/en-us/iaas/Content/API/Concepts/apisigningkey.htm

As a result, you should have a minimal configuration in the `~/.oci/config` file:

```ini
[DEFAULT]
user=ocid1.user.oc1..<your-value-here>
fingerprint=<your-value-here>
tenancy=ocid1.tenancy.oc1..<your-value-here>
region=<your-value-here>
key_file=<your-value-here>
```

Now with configured access to your Oracle Cloud account we can retrieve the `compartment-id`, which will be needed later:

```bash
oci iam user list | jq -r '.data[] | ."compartment-id"'
```

## Terraform project customisation

Clone the Terraform project template:

```bash
git clone git@github.com:kepuvv/oci-free-tier-terraform.git
cd oci-free-tier-terraform
```

Create the variables file from the example file:

```bash
cp terraform.tfvars.example terraform.tfvars
```

`availability_domain_name` — to determine the appropriate Availability Domain, run:

```
export OCI_COMPARTMENT_OCID=$(cat ~/.oci/config | grep 'tenancy' | cut -d= -f2)
```

```bash
for ad in $(oci iam availability-domain list --compartment-id "$OCI_COMPARTMENT_OCID" --query 'data[].name' | jq -r '.[]'); do
  echo "=== $ad ===\n"
  oci compute shape list \
    --compartment-id "$OCI_COMPARTMENT_OCID" \
    --availability-domain "$ad" \
    --query 'data[?"shape"==`VM.Standard.E2.1.Micro`]' \
    | jq '.[].shape'
done
```

It will output something like:

```text
=== VluR:EU-FRANKFURT-1-AD-1 ===

=== VluR:EU-FRANKFURT-1-AD-2 ===

"VM.Standard.E2.1.Micro"
=== VluR:EU-FRANKFURT-1-AD-3 ===
```

The Availability Domain `VluR:EU-FRANKFURT-1-AD-2` that contains the required `VM.Standard.E2.1.Micro` shape will be the value for `availability_domain_name`.

`tenancy_ocid` — is equal to the `compartment-id` that we retrieved earlier.

Next, create an S3 bucket to store the Terraform state file so it is accessible to all project members.

For convenience, define the following environment variables (`OCI_COMPARTMENT_OCID` is the compartment-id, which is also the tenancy-ocid that we obtained earlier; don't forget to replace the values with your own):

```bash
export OCI_COMPARTMENT_OCID="ocid1.tenancy.oc1..<your-value-here>"
export TF_STATE_BUCKET="bucket-name-<your-value-here>"
export OCI_NAMESPACE="$(oci os ns get --query 'data' --raw-output)"
```

Create the S3 bucket using OCI CLI:

```bash
oci os bucket create \
  --compartment-id "$OCI_COMPARTMENT_OCID" \
  --namespace-name "$OCI_NAMESPACE" \
  --name "$TF_STATE_BUCKET" \
  --public-access-type NoPublicAccess \
  --storage-tier Standard
```

Enable versioning (OCI does not appear to allow enabling versioning during bucket creation, or I have not found a way to do it. If someone knows how to optimize bucket creation and enable versioning in a single command, I would appreciate it):

```bash
oci os bucket update \
  --namespace-name "$OCI_NAMESPACE" \
  --name "$TF_STATE_BUCKET" \
  --versioning Enabled
```

## Configure access to Terraform S3 Backend

OCI's Terraform S3 backend uses the S3-compatible Object Storage API.

Create a Customer Secret Key for your OCI user:

```bash
oci iam customer-secret-key create --display-name display-name --user-id ocid1.user.oc1..<your-value-here>
```

Copy the ID (`AWS_ACCESS_KEY_ID`) and the key (`AWS_SECRET_ACCESS_KEY`) to a secure location.

Add them to `~/.aws/credentials`:

```ini
[oracle]
aws_access_key_id = <secret-key-id>
aws_secret_access_key = <secret-key-secret>
```

Reference:

https://docs.oracle.com/en/learn/ocios-s3-api-cpp/#task-2-determine-your-tenancy-namespace-and-s3-api-compartment

## Enable S3 bucket state

Return to the Terraform repository.

Create your backend configuration from the example:

```bash
cp backend.s3.tfbackend.example backend.s3.tfbackend
```

Uncomment in `main.tf`:

```terraform
backend "s3" {}
```

To avoid the following error caused by ignoring `skip_s3_checksum = true` in `backend.s3.tfbackend`:

```text
Error:
│ "s3" backend:
│ failed to upload state: operation error S3: PutObject,
│ https response error StatusCode: 501,
│ api error NotImplemented:
│ AWS chunked encoding not supported.
```

Set the following environment variables:

```bash
export AWS_REQUEST_CHECKSUM_CALCULATION=when_required
export AWS_RESPONSE_CHECKSUM_VALIDATION=when_required
```

**Do not commit `backend.s3.tfbackend` if it contains real bucket or account details.**

Everything is now ready to provision the infrastructure using the prepared Terraform configuration:

```bash
terraform init -backend-config=backend.s3.tfbackend
terraform plan
terraform apply
```

After the deployment completes, Terraform automatically generates an Ansible inventory file that can be used to manage the newly created virtual machines with Ansible.
