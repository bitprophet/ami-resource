# AMI Update Resource

A Concourse CI resource to check for new Amazon Machine Images (AMI).

## Source Configuration

- `aws_access_key_id`: Your AWS access key ID.

- `aws_secret_access_key`: Your AWS secret access key. 

- `region`: *Required.* The AWS region to search for AMIs.

- `filters`: *Required.* A map of named filters to their values. Check the AWS CLI [describe-images](http://docs.aws.amazon.com/cli/latest/reference/ec2/describe-images.html) documentation for a complete list of acceptable filters and values.

If `aws_access_key_id` and `aws_secret_access_key` are both absent, AWS CLI will fall back to other authentication mechanisms. See [Configuration setting and precedence](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#config-settings-and-precedence)

## Behaviour

### `check`: Check for new AMIs

Searches for AMIs that match the provided source filters, ordered by their creation date. The AMI ID serves as the resulting version.

#### Parameters

*None.*

### `in`: Fetch the description of an AMI

Places the following files in the destination:

- `output.json`: The complete AMI description object in JSON format. Check the AWS CLI [describe-images](http://docs.aws.amazon.com/cli/latest/reference/ec2/describe-images.html#examples) documentation for examples.

- `id`: A plain text file containing the AMI ID, e.g. `ami-5731123e`

- `packer.json`: The AMI ID in Packer `var-file` input format, typically for use with [packer-resource](https://github.com/jdub/packer-resource), e.g.

  ```json
  {"source_ami": "ami-5731123e"}
  ```

#### Parameters

*None.*

### `out`: Signal that a job has created an AMI

You only really need this if you want Concourse to understand that a given job
has created a given AMI ID as a resource version. For example, if you run
tests against an AMI and want a final, fan-in build plan step that uses `get:
passed: [test, jobs, here]`. Or if you simply want to have useful records of
which Concourse jobs generated which AMIs.

Simple "trigger a job when a new AMI shows up" behavior doesn't require
using this; in that case you should be using the original resource this was
based upon.

#### How it works

It simply expects that some build task has placed an `id.json` in an `outputs`
directory matching the `put`'s configured `output_dir`. `id.json` should be of
the format `{"ami": "ami-d34db33f"}` and that ID value should be the one
created by the build task.

This JSON bob is then printed to stdout to communicate with the rest of
Concourse.

#### Parameters

- `output_dir`: The `outputs` entry from the appropriate preceding build task.

## Example

This pipeline will check for a new Ubuntu 14.04 LTS AMI in the Sydney region
every hour, triggering a build step which generates a new internal AMI from
that upstream one, and then emitting the new AMI ID as the new version.

```yaml
resource_types:
- name: ami
  type: docker-image
  source:
    repository: bitprophet/ami-resource

resources:
- name: ubuntu-ami
  type: ami
  check_every: 1h
  source:
    aws_access_key_id: "..."
    aws_secret_access_key: "..."
    region: ap-southeast-2
    filters:
      owner-id: "099720109477"
      is-public: true
      state: available
      name: ubuntu/images/hvm-ssd/ubuntu-trusty-*server*

jobs:
- name: my-ami
  plan:
  - get: ubuntu-ami
    trigger: true
  - task: build-fresh-image
    config:
      inputs: ubuntu-ami
      run: my-awesome-build-script
      outputs: built-ami
  - put: ubuntu-ami
    params:
      output_dir: built-ami
```
