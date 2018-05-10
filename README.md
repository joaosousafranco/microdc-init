[![Build Status](https://travis-ci.org/microdc/microdc-init.svg)](https://travis-ci.org/microdc/microdc-init)

# ee-microdc-init
A python package to manage a pick 'n' mix Kubernetes infrastructure on AWS

### Install
Requirements:

* Python 3.x (ideally 3.6.x). It may work with python 2.7.x but that is not tested at present.

Recommended utilities:

* [virtualenv](https://virtualenv.pypa.io/en/stable/) https://virtualenv.pypa.io/en/stable/ which ensures you avoid conflicts with system python packages.

If installing within a virtualenv create the virtualenv (only once) and activate it

`virtualenv -p <path to python 3> <name of virtualenv>` only once, doing again will simply inform the user that the virtualenv already exists.

`source <name of virtualenv>/bin/activate`

Install the package

`pip install git+https://github.com/microdc/microdc-init`

### Usage
`microdc --help`

### Tests
Running the tests with the following script requires `virtualenv`

`$ ./test.sh`

## Bring up a MicroDC environment
This will bring up kubernetes clusters in AWS accounts.  In this example we will spin up a single cluster called dev.

### Prerequisites
 * Two accounts with AWS - prod and nonprod (you can also go with one)
 * Delegate a DNS domain or sub domain
 * Generate a domain certificate from ACM and save the arn for use in the config file below - [howto](https://github.com/microdc/microdc-init/blob/master/docs/configure_acm_cert.md)
 * We have found raising the following AWS limits helpful (based on a environment setup of dev, test, stage, prod) :
   - Raise the Ec2 instance limit to 100
   - Increase elastic IPs to 20
   - Raise the VPC limit to 20
 * Install terraform - Download the relevant binary for your operating system [here](https://www.terraform.io/downloads.html). Currently 0.10.8.
 * Install kops - Download the relevant binary for your operating system [here](https://github.com/kubernetes/kops/releases/tag/1.8.0)
 * Install kubectl - Follow [these](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-binary-via-curl) steps
 * Install AWS CLI - Instructions [here](https://docs.aws.amazon.com/cli/latest/userguide/installing.html)
 * Install envsubst

### Then follow these steps

1. First we setup our environment

   Using the [test config](https://github.com/microdc/microdc-init/blob/master/tests/good_config.yaml) as a template, fill out the relevant details.
   ```
   microdc --workdir ~/.microdc --config config.yml --setup up
   microdc --workdir ~/.microdc --config config.yml --setup up | sh
   ```

2. We run the global setup (GLOBAL terraform)
```
microdc --workdir ~/.microdc --config config.yml --account nonprod --stack global --tool terraform up --bootstrap
microdc --workdir ~/.microdc --config config.yml --account nonprod --stack global --tool terraform up --bootstrap | sh
```

3. Initial cluster setup
```
microdc --workdir ~/.microdc --config config.yml --account nonprod --tool kops up --env dev
microdc --workdir ~/.microdc --config config.yml --account nonprod --tool kops up --env dev | sh
```

4. Complete setup around the edges. (SERVICE Stack - this is per env)
```
microdc --workdir ~/.microdc --config config.yml --account nonprod --stack service --tool terraform up --env dev
microdc --workdir ~/.microdc --config config.yml --account nonprod --stack service --tool terraform up --env dev | sh
```

5. Validate our setup.
 Update the NS records if using a delegated Route 53 DNS subdomain.
 The (super) DNS domain should be updated to use the NS servers of the new, terraform-created, subdomain.
```
# Copy these from the output of the kops command above
export AWS_DEFAULT_REGION=eu-west-1
export AWS_PROFILE=test-nonprod
export KOPS_STATE_STOR=s3://test-nonprod-kops
kops validate cluster
```
6. Deploy kubernetes level components - telemetry etc
```
microdc --workdir ~/.microdc --config config.yml --account nonprod --tool kubectl up --env dev
microdc --workdir ~/.microdc --config config.yml --account nonprod --tool kubectl up --env dev | sh
```

### Extras - create developer and deployment users for the apps namespace
[https://github.com/microdc/apps-namespace](https://github.com/microdc/apps-namespace)
