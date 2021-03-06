# Testing terraform-provider-vcd

## Table of contents

- [Running tests](#running-tests)
- [Tests split by feature set](#Tests-split-by-feature-set)
- [Adding new tests](#Adding-new-tests)
- [Binary testing](#Binary-testing)
- [Custom terraform scripts](#Custom-terraform-scripts)
- [Environment variables](#Environment-variables)

## Running tests

In order to test the provider, you can simply run `make test`.

```sh
$ make test
```

In order to run the full suite of Acceptance tests, run `make testacc`.

*Note:* Acceptance tests create real resources, and often cost money to run.

```sh
$ make testacc
```

The acceptance tests will run against your own vCloud Director setup, using the configuration in your file `./vcd/vcd_test_config.json`
See the file `./vcd/sample_vcd_test_config.json` for an example of which variables need to be defined.

Each test in the suite will write a Terraform configuration file inside `./vcd/test-artifacts`, named after the
tests. For example: `vcd.TestAccVcdNetworkDirect.tf`

The test suite will try to minimize the amount of resources to create. If no catalog and vApp
template (`catalogItem`) are defined in the configuration file, new ones will be created and removed at the end of
the test. You can choose to preserve catalog and vApp template across runs (use the `preserve` field in the
configuration file).


Both the (short) test and the acceptance test include a run of the `unit` tests, i.e. tests that check the correctness
of the code without need for a live vCD.

You can run the unit tests directly with

```sh
make testunit
```

## Tests split by feature set

The tests can run with several tags that define which components are tested.
Using the Makefile, you can run one of the following:

```bash
make testcatalog
make testnetwork
make testextnetwork
make testgateway
make testvapp
make testvm
```

For more options, you can run manually in `./vcd`
When running `go test` without tags, you'll get a list of tags that are available.

```
$ go test -v .
=== RUN   TestTags
 --- FAIL: TestTags (0.00s)
     api_test.go:87: # No tags were defined
     api_test.go:62:
         # -----------------------------------------------------
         # Tags are required to run the tests
         # -----------------------------------------------------

         At least one of the following tags should be defined:

            * ALL :       Runs all the tests
            * functional: Runs all the acceptance tests
            * unit:       Runs unit tests that don't need a live vCD

            * catalog:    Runs catalog related tests (also catalog_item, media)
            * disk:       Runs disk related tests
            * network:    Runs network related tests
            * gateway:    Runs edge gateway related tests
            * org:        Runs org related tests
            * vapp:       Runs vapp related tests
            * vdc:        Runs vdc related tests
            * vm:         Runs vm related tests

         Examples:

           go test -tags unit -v -timeout=45m .
           go test -tags functional -v -timeout=45m .
           go test -tags catalog -v -timeout=15m .
           go test -tags "org vdc" -v -timeout=5m .

         Tagged tests can also run using make
           make testunit
           make testacc
           make testcatalog
 FAIL
 FAIL	github.com/terraform-providers/terraform-provider-vcd/v2/vcd	0.017s
```

## Adding new tests

All tests need to have a build tag. The tag should be the first line of the file, followed by a blank line

```go
// +build functional featurename ALL

package vcd
```

Tests that integrate in the functional suite use the tag `functional`. Using that tag, we can run all functional tests
at once.
We define as `functional` the tests that need a live vCD to run.

1. The test should always define the `ALL` tag:

* ALL :       Runs all the tests

2. The test should also always define either the `unit` or `functional` tag:

* functional: Runs all the tests that use a live vCD (acceptance tests)
* unit:       Runs unit tests that do not need a live vCD

3. Finally, the test should always define the feature tag. For example:

* catalog:    Runs catalog related tests (also `catalog_item`, `media`)
* vapp:       Runs vapp related tests

The `ALL` tag includes tests that use a different framework. At the moment, this is useful to run a global compilation test.
Depending on which additional tests we will implement, we may change the dependency on the `ALL` tag if we detect
clashes between frameworks.

If the test file defines a new feature tag (i.e. one that has not been used before) the file should also implement an
`init` function that sets the tag in the global tag list.
This information is used by the main tag test in `api_test.go` to determine which tags were activated.

```go
func init() {
	testingTags["newtag"] = "filename_test.go"
}
```

**VERY IMPORTANT**: if we add a test that runs using a different tag (i.e. it is not included in `functional` tests), we need
to add such test to GNUMakefile under `make test` and `make testacc`. **The general principle is that `make test` and `make testacc` run all tests**. If this can't be
achieved by adding the new test to the `functional` tag (perhaps because we foresee framework conflicts), we need to add the
new test as a separate command.
For example, the unit test run as a command before the acceptance test:

```
testacc: testunit
	@sh -c "'$(CURDIR)/scripts/runtest.sh' acceptance"
```


## Binary testing

By *binary testing* we mean the tests that run using Terraform binary executable, as opposed to running the test through the Go framework.
This test runs the same tasks that run in the acceptance test, but instead of running them directly, they are fed to the
terraform tool through a shell script, and for every test we run

* `terraform init`
* `terraform plan`
* `terraform apply -auto-approve`
* `terraform destroy -auto-approve`

The test runs from GNUMakefile, using

```bash
make test-binary
```

All the tests run unattended, stopping only if there is an error.

It is possible to customise running of the binary tests by preparing them and then running the test script from the `tests-artifacts` directory:

```bash
make test-binary-prepare
[...]

cd ./vcd/test-artifacts
./test-binary.sh help

# OR
./test-binary.sh pause verbose

# OR
./test-binary.sh pause verbose tags "catalog gateway"
```

The "pause" option will stop the test after every call to the terraform tool, waiting for user input.

When the test runs unattended, it is possible to stop it gracefully by creating a file named `pause` inside the
`test-artifacts` directory. When such file exists, the test execution stops at the next `terraform` command, waiting
for user input.

## Custom terraform scripts

The commands `make test-binary-prepare` and `make test-binary` have the added benefit of compiling custom Terraform scripts located in `./vcd/test-templates`.
These tests are similar to the ones produced by the testing framework, but unlike the standard ones, they can be edited by users. And users can also remove and add files to suit their purposes.

The files in `test-templates` are not executable directly by `terraform`: they need to be processed (which happens during `make test-binary-prepare`) and their placeholders expanded to the values taken from the configuration file.

A **placeholder** is a label enclosed in double braces and prefixed by a dot, such as `{{.LabelName}}`.
The template processor will replace that label with the corresponding value taken from the test configuration file.
See `sample_vcd_test_config.json` for a description of all fields.

This is the list of placeholders that you can use in template files:

Label                        | Field in `vcd_test_config.json`
:----------------------------|:------------------------------------------
Org                          | vcd.org
Vdc                          | vcd.vdc
ProviderVdc                  | vcd.providerVdc.name
NetworkPool                  | vcd.providerVdc.networkPool
StorageProfile               | vcd.providerVdc.storageProfile
Catalog                      | vcd.catalog.name
CatalogItem                  | vcd.catalog.catalogItem
OvaPath                      | ova.OvaPath
MediaPath                    | media.MediaPath
MediaUploadPieceSize         | media.UploadPieceSize
MediaUploadProgress          | media.UploadProgress
OvaDownloadUrl               | ova.ovaDownloadUrl
OvaTestFileName              | ova.ovaTestFileName
OvaUploadProgress            | ova.uploadProgress
OvaUploadPieceSize           | ova.uploadPieceSize
OvaPreserve                  | ova.preserve
LoggingEnabled               | logging.enabled
LoggingFileName              | logging.logFileName
EdgeGateway                  | networking.edgeGateway
SharedSecret                 | networking.sharedSecret
ExternalNetwork              | networking.externalNetwork
ExternalNetworkPortGroup     | networking.externalNetworkPortGroup
ExternalNetworkPortGroupType | networking.externalNetworkPortGroupType
ExternalIp                   | networking.externalIp
InternalIp                   | networking.internalIp
Vcenter                      | networking.vcenter
LocalIp                      | networking.local.localIp
LocalGateway                 | networking.local.localSubnetGateway
PeerIp                       | networking.peer.peerIp
PeerGateway                  | networking.peer.peerSubnetGateway
MaxRetryTimeout              | provider.maxRetryTimeout
AllowInsecure                | provider.allowInsecure
ProviderSysOrg               | provider.sysOrg
ProviderUrl                  | provider.url
ProviderUser                 | provider.user
ProviderPassword             | provider.password


The files generated from `./vcd/test-templates` will end up in `./vcd/test-artifacts`, and you will recognize them because their name will start by `cust.` instead of `vcd.`, and they all use the tag `custom`.

Note that the template files should **not** have a `provider` section, as it is created by the template processor.
Inside the template, you can indicate the need for specific `terraform` options, by inserting one or more comments containing `init-options`, `plan-options`, `apply-options`, or `destroy-options`. The options, if indicated, will be added to the corresponding `terraform` command. For example:

```
# apply-options -no-color
# destroy-options -no-color
```
When running `terraform apply` and `terraform destroy`, the option `-no-color` will be added to the command line.

To run these tests, you go inside `test-artifacts` and execute:

```bash
./test-binary.sh names "cust*.tf" [options]

# or

./test-binary.sh names "*.tf" tags custom [options]

# or

./test-binary.sh names cust.specific-file-name.tf [options]
```

The execution then proceeds as explained in [Binary testing](#Binary-testing).


## Environment variables

There are several environment variables that can affect the tests:

* `TF_ACC=1` enables the acceptance tests. It is also set when you run `make testacc`.
* `GOVCD_DEBUG=1` enables debug output of the test suite
* `VCD_SKIP_TEMPLATE_WRITING=1` skips the production of test templates into `./vcd/test-artifacts`
* `ADD_PROVIDER=1` Adds the full provider definition to the snippets inside `./vcd/test-artifacts`.
   **WARNING**: the provider definition includes your vCloud Director credentials.
* `VCD_CONFIG=FileName` sets the file name for the test configuration file.
* `REMOVE_ORG_VDC_FROM_TEMPLATE` is a quick way of enabling an alternate testing mode:
When `REMOVE_ORG_VDC_FROM_TEMPLATE` is set, the terraform
templates will be changed on-the-fly, to comment out the definitions of org and vdc. This will force the test to
borrow org and vcd from the provider.
* `VCD_TEST_SUITE_CLEANUP=1` will clean up testing resources that were created in previous test runs.
* `TEST_VERBOSE=1` enables verbose output in some tests, such as the list of used tags, or the version
used in the documentation index.
