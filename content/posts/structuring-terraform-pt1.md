+++
author = "nivaldo melo"
title = "Structuring a Terraform project pt1"
date = "2024-08-13"
description = "A guide about how to structure a terraform project inside your team"
tags = [ "golang", "terraform", ]
+++

## Structuring a Terraform project Pt.1

As we know, Terraform is one of the most popular tools when we talk about declarative Infrastructure as Code (IAC). However, one aspect that often feels a bit unclear for me is how to structure projects effectively. While it's easy to find tutorials on best practices for naming resources, figuring out the right folder structure can be more challenging. Should you use modules? Workspaces? How much should you aggregate resources into a single .tf file?

To address these questions, I decided to create a structured module that I could use across my projects. This structure includes functionalities that I consider essential. The module is built on four key pillars:

- **Modules**: These are used to group related services that function as a unit, reducing code repetition and improving maintainability.
- **Examples**: This section contains use case examples to make it easier for teams to adopt this structure.
- **Unit Tests**: These help us detect failures early in the development process.
- **Documentation**: We'll aim to make the documentation process as seamless as possible, so it doesn't become a burden during development.

Let's start by exploring why these pillars are important.

----------

### Modules

As many of you know, a module in Terraform is a way to increase code reusability by aggregating resources that are often created together. By doing this, you can centralize configuration in a single file, making bulk changes easier and more efficient. It's important to remember that when you're writing modules, other people might depend on your code. Therefore, it's crucial to be mindful of this before making any breaking changes.

### Examples

All examples should have a practical use case, making it easier for readers to see how they can be applied in their own environments.

### Unit Tests

Testing infrastructure can be uncommon due to potential costs, but when done in a controlled environment, it greatly simplifies development and maintenance. Through tests, we can describe the most important scenarios that the module should cover, helping to prevent unwanted breaking changes. Another advantage is that by documenting these scenarios, we don't have to rely on someone remembering to manually test everything, which can lead to errors.

### Documentation

Last but not least, there's the documentation. Good documentation makes it easier for teams to understand and adopt your product. However, if done manually every time, it's easy for things to become outdated, leading to inaccurate and difficult-to-follow documents. That's why we aim to make the documentation process as seamless as possible, ensuring that it always stays up to date with the project's code.

## Final look

So to have an idea of what we're aiming to, this is the repo structure that we're aiming to have in the end. You may notice that we have some `.go` files, it's because we're gonna use a tool written in golang to write our tests

``` bash
terraform-modules-structure
├── examples/
├── go.mod
├── go.sum
├── modules/
├── README.md
└── test/
```

----------

## Writing your code

Before we start, let's define the kind of service we're going to write. For this example, we'll create a Cloud Function on GCP. In a Cloud Function, there are multiple ways to store your source code; in this example, we'll use a GCS bucket.

### Initializing the Repository

This part is straightforward; we just need to start a Golang project, which can be done as follows:

``` bash
❯ mkdir terraform-modules-structure && cd terraform-modules-structure
❯ go mod init terraform-modules-structure
```

Note that I've named my project `terraform-modules-structure`, but you can name it whatever you like.

### Examples

This is arguably the most important phase of the project. Here, we'll define the module's use cases, thinking like a final user. Therefore, the examples need to be clear, practical, and stable; otherwise, adoption of the project might be compromised.

As end users, we need to create a Cloud Function in GCP. Let's start by considering which parameters we should control and which ones we shouldn't.

Looking at this [example](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/cloudfunctions2_function#example-usage---cloudfunctions2-basic), we'll propose that the user can control the following parameters:
- function name
- region
- description
- runtime
- entry_point
- max_instance
- memory
- timeout

Other parameters can be defined within the module itself, including the connection to the bucket.

#### Terraform files

So let's start creating this files

``` bash
❯ mkdir -p examples/gcp/cloud-function-v2
❯ touch examples/gcp/cloud-function-v2/main.tf
```

In `main.tf` let's write:

``` hcl
module "function" {
  source = "../../modules/gcp/cloud-function-v2"

  name        = "my-function"
  description = "My function"
  region      = "us-central1"

  runtime     = "nodejs16"
  entry_point = "helloGET"

  source_path = "./src"

  max_instances = 1
  memory        = 256
  timeout       = 60
}
```

#### Deploying the source code for the function

Let's create the function so that we're able to deploy it. To do this, follow the example in the official Google documentation. To fit it into our structure, you just need to do the following:

``` bash
❯ mkdir examples/gcp/cloud-function-v2/src
❯ touch examples/gcp/cloud-function-v2/src/index.js
❯ touch examples/gcp/cloud-function-v2/src/package.json
```

and then fill the `index.js` with:
``` javascript
const functions = require('@google-cloud/functions-framework');

// Register an HTTP function with the Functions Framework that will be executed
// when you make an HTTP request to the deployed function's endpoint.
functions.http('helloGET', (req, res) => {
  res.send('Hello World!');
});
```

and `package.json` with:

``` json
{
  "name": "nodejs-docs-samples-functions-hello-world-get",
  "version": "0.0.1",
  "private": true,
  "license": "Apache-2.0",
  "author": "Google Inc.",
  "repository": {
	"type": "git",
	"url": "https://github.com/GoogleCloudPlatform/nodejs-docs-samples.git"
  },
  "engines": {
	"node": ">=16.0.0"
  },
  "scripts": {
	"test": "c8 mocha -p -j 2 test/*.test.js --timeout=6000 --exit"
  },
  "dependencies": {
	"@google-cloud/functions-framework": "^3.1.0"
  },
  "devDependencies": {
	"c8": "^8.0.0",
	"gaxios": "^6.0.0",
	"mocha": "^10.0.0",
	"wait-port": "^1.0.4"
  }
}
```

Great! Now we have a use case, so let's document it.

#### Documentation

Now that we have a use case, it's important to make it easy to adopt by providing a README that explains how to use it. For this, we'll use [terraform-docs](https://terraform-docs.io/user-guide/introduction/). After installing it, we'll set up the following:

First, create the `examples/.terraform-docs.yml` file, which will serve as the base structure for all the examples:

```` yaml
formatter: "markdown table"
header-from: header.md # Specifies the source for the README header content

content: |-
  {{ .Header }}

  ```hcl
  {{ include "main.tf" }}
  ```
````

Next, create the `examples/gcp/cloud-function-v2/header.md` file to define a header specific to our example. For this case, the content might be:

``` markdown
## Creating a Cloud Function

This example demonstrates how to create a Cloud Function in GCP using Terraform. The example will zip the code inside the `src/` folder and store it in a GCS bucket.
```

Finally, run the following command to generate your README:

``` bash
❯ cd examples/gcp/cloud-function-v2
❯ terraform-docs -c ../../.terraform-docs.yml .
```

With this we have a documented use case and provide a process to regenerate it based on your code. To automate this process, simply add it to your CI pipeline.

## Next steps

In the next part of the series, we’ll write a unit test based on our example.
