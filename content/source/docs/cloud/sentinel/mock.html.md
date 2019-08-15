---
layout: "cloud"
page_title: "Mocking Terraform Sentinel Data - Sentinel - Terraform Cloud"
---

# Mocking Terraform Sentinel Data

We recommend that you test your Sentinel policies extensively before deploying
them within Terraform Cloud. An important part of this process is mocking
the data that you wish your policies to operate on.

Due to the highly variable structure of data that can be produced by an
individual Terraform configuration, Terraform Cloud provides the ability to
generate mock data from existing configurations. This can be used to create
sample data for a new policy, or data to reproduce issues in an existing one.

Testing policies is done using the [Sentinel
Simulator](https://docs.hashicorp.com/sentinel/commands/). More general
information on testing Sentinel policies can be found in the [Testing
section](https://docs.hashicorp.com/sentinel/writing/testing) of the [Sentinel
runtime documentation](https://docs.hashicorp.com/sentinel).

~> **Be careful!** Mock data generated by Terraform Cloud directly exposes
any and all data within the configuration, plan, and state, including any
sensitive data. Treat this data with care, and avoid generating mocks with live
sensitive data if at all possible. To help secure access to this information,
[write
permission](/docs/cloud/users-teams-organizations/permissions.html#write)
on the workspace is necessary to generate mock data.

## Generating Mock Data Using the UI

Mock data can be generated using the UI by expanding the plan status section of
the run page, and clicking on the **Download Sentinel mocks** button.

![sentinel mock generate ui](/assets/images/guides/sentinel/download-mocks.png)

For more information on creating a run, see [Running
Terraform](/docs/cloud/getting-started/runs.html) in the [Getting
Started](/docs/cloud/getting-started/index.html) guide.

If the button is not visible, then the plan is ineligible for mock generation or
the user doesn't have the necessary permissions. See [Mock Data
Availability](#mock-data-availability) for more details.

## Generating Mock Data Using the API

Mock data can also be created with the [Plan Export
API](/docs/cloud/api/plan-exports.html).

Multiple steps are required for mock generation. The export process is
asynchronous, so you must monitor the request to know when the data is generated
and available for download.

1. Get the plan ID for the run that you want to generate the mock for by
   [getting the run details](/docs/cloud/api/run.html#get-run-details).
   Look for the `id` of the `plan` object within the `relationships` section of
   the return data.
1. [Request a plan
  export](/docs/cloud/api/plan-exports.html#create-a-plan-export) using the
  discovered plan ID. Supply the Sentinel export type `sentinel-mock-bundle-v0`.
1. Monitor the export request by [viewing the plan
  export](/docs/cloud/api/plan-exports.html#show-a-plan-export). When the
  status is `finished`, the data is ready for download.
1. Finally, [download the export
   data](/docs/cloud/api/plan-exports.html#download-exported-plan-data).
   You have up to an hour from the completion of the export request - after
   that, the mock data expires and must be re-generated.

## Using Mock Data

Mock data is supplied as a bundled tarball, containing 3 files:

```
mock-tfconfig.sentinel # tfconfig mock data
mock-tfplan.sentinel   # tfplan mock data
mock-tfstate.sentinel  # tfstate mock data
```

The recommended placement of the files is in a subdirectory of the repository
holding your policies, so they don't interfere with `sentinel test`. While the
test data is Sentinel code, it's not a policy and will produce errors if
evaluated like one.

```
.
├── foo.sentinel
├── sentinel.json
├── test
│   └── foo
│       ├── fail.json
│       └── pass.json
└── testdata
    ├── mock-tfconfig.sentinel
    ├── mock-tfplan.sentinel
    └── mock-tfstate.sentinel
```

Each configuration that needs access to the mock should reference the mock data
files within the `mock` block in the Sentinel configuration file.

For `sentinel apply`, this path is relative to the working directory. Assuming
you always run this command from the repository root, the `sentinel.json`
configuration file would look like:

```
{
  "mock": {
    "tfconfig": "testdata/mock-tfconfig.sentinel",
    "tfplan": "testdata/mock-tfplan.sentinel",
    "tfstate": "testdata/mock-tfstate.sentinel"
  }
}
```

For `sentinel test`, the paths are relative to the specific test configuration
file. For example, the contents of `pass.json`, asserting that the result of the
`main` rule was `true`, would be:

```
{
  "mock": {
    "tfconfig": "../../testdata/mock-tfconfig.sentinel",
    "tfplan": "../../testdata/mock-tfplan.sentinel",
    "tfstate": "../../testdata/mock-tfstate.sentinel"
  },
  "test": {
    "main": true
  }
}
```

## Mock Data Availability

The following factors can affect your ability to generate mock data:

* You do not have [write
  access](/docs/cloud/users-teams-organizations/permissions.html#write) to
  the workspace. This is to protect the possibly sensitive data which can be
  produced via mock generation.
* The run has not progressed past the planning stage, or did not create a plan
  successfully.
* The run has been in a terminal state, such as applied or discarded, longer
  than seven days. At this point, the data necessary to generate the mocks is no
  longer available.

If a plan cannot have its mock data exported due to any of these reasons, the
**Download Sentinel mocks** button within the plan status section of the UI will
not be visible.

-> **Note:** Only the plan needs to be successful for a run to be eligible for
mock generation - if the apply or the policy checks fail, the data can still be
generated, as long as it's within the 7 day expiration period.