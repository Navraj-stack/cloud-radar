<!-- PROJECT SHIELDS -->
<!--
*** I'm using markdown "reference style" links for readability.
*** Reference links are enclosed in brackets [ ] instead of parentheses ( ).
*** See the bottom of this document for the declaration of the reference variables
*** for contributors-url, forks-url, etc. This is an optional, concise syntax you may use.
*** https://www.markdownguide.org/basic-syntax/#reference-style-links
-->
<!-- [![Contributors][contributors-shield]][contributors-url]
[![Forks][forks-shield]][forks-url]
[![Stargazers][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url] -->

<!-- PROJECT LOGO -->
<br />
<p align="center">
  <!-- <a href="https://github.com/DontShaveTheYak/cloud-radar">
    <img src="images/logo.png" alt="Logo" width="80" height="80">
  </a> -->

  <h3 align="center">Cloud-Radar</h3>

<!-- TABLE OF CONTENTS -->
<details open="open">
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
      <ul>
        <li><a href="#built-with">Built With</a></li>
      </ul>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li><a href="#prerequisites">Prerequisites</a></li>
        <li><a href="#installation">Installation</a></li>
      </ul>
    </li>
    <li><a href="#usage">Usage</a></li>
    <li><a href="#roadmap">Roadmap</a></li>
    <li><a href="#contributing">Contributing</a></li>
    <li><a href="#license">License</a></li>
    <li><a href="#contact">Contact</a></li>
    <li><a href="#acknowledgements">Acknowledgements</a></li>
    <li><a href="#about-the-maintainer">About the Maintainer</a></li>
  </ol>
</details>

## About The Project

<!-- [![Product Name Screen Shot][product-screenshot]](https://example.com) -->

Cloud-Radar is a python module that allows testing of Cloudformation Templates/Stacks using Python.

### Unit Testing

You can now unit test the logic contained inside your Cloudformation template. Cloud-Radar takes your template, the desired region and some parameters. We render the template into its final state and pass it back to you.

You can Test:
* That Conditionals in your template evaluate to the correct value.
* Conditional resources were created or not.
* That resources have the correct properties.
* That resources are named as expected because of `!Sub`.

You can test all this locally without worrying about AWS Credentials.

### Functional Testing

This project is a wrapper around Taskcat. Taskcat is a great tool for ensuring your Cloudformation Template can be deployed in multiple AWS Regions. Cloud-Radar enhances Taskcat by making it easier to write more complete functional tests.

Here's How:
* You can interact with the deployed resources directly with tools you already know like boto3.
* You can control the lifecycle of the stack. This allows testing if resources were retained after the stacks were deleted.
* You can run tests without hardcoding them in a taskcat config file.

This project is new and it's possible not all features or functionality of Taskcat/Cloudformation are supported (see [Roadmap](#roadmap)). If you find something missing or have a use case that isn't covered then please let me know =)

### Built With

* [Taskcat](https://github.com/aws-quickstart/taskcat)
* [cfn_tools from cfn-flip](https://github.com/awslabs/aws-cfn-template-flip)

## Getting Started

Cloud-Radar is available as an easy to install pip package.

### Prerequisites

Cloud-Radar requires python >= 3.8

### Installation

1. Install with pip.
   ```sh
   pip install cloud-radar
   ```

## Usage
<details>
<summary>Unit Testing <span style='font-size: .67em'>(Click to expand)</span></summary>

Using Cloud-Radar starts by importing it into your test file or framework. We will use this [Template](./tests/templates/log_bucket/log_bucket.yaml) as an example.

```python
from pathlib import Path
from cloud_radar.cf.unit import Template

template_path = Path("tests/templates/log_bucket/log_bucket.yaml")

# template_path can be a str or a Path object
template = Template.from_yaml(template_path.resolve())

params = {"BucketPrefix": "testing", "KeepBucket": "TRUE"}

# parameters and region are optional arguments.
stack = template.create_stack(params, region="us-west-2")

stack.no_resource("LogsBucket")

bucket = stack.get_resource("RetainLogsBucket")

assert "DeletionPolicy" in bucket

assert bucket["DeletionPolicy"] == "Retain"

bucket_name = bucket.get_property_value("BucketName")

assert "us-west-2" in bucket_name
```

The AWS pseudo variables are all class attributes and can be modified before rendering a template.
```python
# The value of 'AWS::AccountId' in !Sub "My AccountId is ${AWS::AccountId}" can be changed:
Template.AccountId = '8675309'
```
_Note: Region should only be changed to change the default value. To change the region during testing pass the desired region to render(region='us-west-2')_

The default values for psedo variables:

| Name             | Default Value   |
| ---------------- | --------------- |
| AccountId        | "555555555555"  |
| NotificationARNs | []              |
| **NoValue**      | ""              |
| **Partition**    | "aws"           |
| Region           | "us-east-1"     |
| **StackId**      | ""              |
| **StackName**    | ""              |
| **URLSuffix**    | "amazonaws.com" |
_Note: Bold variables are not fully impletmented yet see the [Roadmap](#roadmap)_

A real unit testing example using Pytest can be seen [here](./tests/test_cf/test_examples/test_unit.py)

</details>

<details>
<summary>Functional Testing <span style='font-size: .67em'>(Click to expand)</span></summary>
Using Cloud-Radar starts by importing it into your test file or framework.

```python
from pathlib import Path

from cloud_radar.cf.e2e import Stack

# Stack is a context manager that makes sure your stacks are deleted after testing.
template_path = Path("tests/templates/log_bucket/log_bucket.yaml")
params = {"BucketPrefix": "testing", "KeepBucket": "False"}
regions = ['us-west-2']

# template_path can be a string or a Path object.
# params can be optional if all your template params have default values
# regions can be optional, default region is 'us-east-1'
with Stack(template_path, params, regions) as stacks:
    # Stacks will be created and returned as a list in the stacks variable.

    for stack in stacks:
        # stack will be an instance of Taskcat's Stack class.
        # It has all the expected properties like parameters, outputs and resources

        print(f"Testing {stack.name}")

        bucket_name = ""

        for output in stack.outputs:

            if output.key == "LogsBucketName":
                bucket_name = output.value
                break

        assert "logs" in bucket_name

        assert stack.region.name in bucket_name

        print(f"Created bucket: {bucket_name}")

# Once the test is over then all resources will be deleted from your AWS account.
```

You can use taskcat [tokens](https://aws.amazon.com/blogs/infrastructure-and-automation/a-deep-dive-into-testing-with-taskcat/) in your parameter values.

```python
parameters = {
  "BucketPrefix": "taskcat-$[taskcat_random-string]",
  "KeepBucket": "FALSE",
}
```

You can skip the context manager. Here is an example for `unittest`

```python
import unittest

from cloud-radar.cf.e2e import Stack

class TestLogBucket(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        template_path = Path("tests/templates/log_bucket/log_bucket.yaml")
        cls.test = Stack(template_path)
        cls.test.create()

    @classmethod
    def tearDownClass(cls):
        cls.test.delete()

    def test_bucket(self):
        stacks = self.__class__.test.stacks

        for stack in stacks:
            # Test
```

A real functional testing example using Pytest can be seen [here](./tests/test_cf/test_examples/test_functional.py)

</details>

## Roadmap

### Project
- Python 3.7 support
- Add Logging
- Add Logo
- Make it easier to interact with stack resources.
  * Getting a resource for testing should be as easy as `stack.Resources('MyResource)` or `template.Resources('MyResource')`
- Easier to pick regions for testing

### Unit
- Add full functionality to pseudo variables.
  * Variables like `Partition`, `URLSuffix` should change if the region changes.
  * Variables like `StackName` and `StackId` should have a better default than ""
- Handle References to resources that shouldn't exist.
  * It's currently possible that a `!Ref` to a Resource stays in the final template even if that resource is later removed because of a conditional.
    
### Functional
- Add the ability to update a stack instance to Taskcat.

## Contributing

Contributions are what make the open-source community such an amazing place to learn, inspire, and create. Any contributions you make are **greatly appreciated**.

This project uses poetry to manage dependencies and pre-commit to run formatting, linting and tests. You will need to have both installed to your system as well as python 3.9.

1. Fork the Project
2. Setup environment (`poetry install`)
3. Setup commit hooks (`pre-commit install`)
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## License

Distributed under the Apache-2.0 License. See [LICENSE.txt](./LICENSE.txt) for more information.

## Contact

NAVRAJ SINGH - navraj771s@gmail.com

<!-- ACKNOWLEDGEMENTS -->
## Acknowledgements
* [Taskcat](https://aws-quickstart.github.io/taskcat/)
* [Hypermodern Python](https://cjolowicz.github.io/posts/hypermodern-python-01-setup/)
* [Best-README-Template](https://github.com/othneildrew/Best-README-Template)

## About the Maintainer

This project is actively maintained by NAVRAJ SINGH. As a Cloud Engineer with 3 years of experience, NAVRAJ specializes in designing, deploying, and optimizing secure, scalable cloud infrastructures on AWS. With a strong background in CI/CD, Terraform, Kubernetes/EKS/AKS, and building high-availability environments, NAVRAJ is committed to ensuring the reliability, scalability, and continued development of Cloud-Radar.

Connect with NAVRAJ:
*   Email: navraj771s@gmail.com
