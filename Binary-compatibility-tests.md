This document gives an overview of the binary compatibility tests: how they are built and when they run in our continuous integration (CI) pipeline. This is meant to be a technical document describing the implementation details. There is no need to run these tests manually -- the idea is that they should run automatically during CI builds.

## Purpose

The purpose of the tests is to ensure that data compatibility is not broken by any commits that are made. Data compatibility can break when, for example, updating our dependencies on compression or serialization libraries, but we also want to make sure it doesn't happen for an unforeseen reason.

## Overview

In our CI pipeline there are two jobs corresponding to the compatibility tests, which run one after the other. They are:

* The `test-binary` job, which runs the tests using a cached, previous build as well as a current build to make sure the new code being tested by CI has not broken data compatibility. This is run as a standalone job, which currently runs parallel with the rest of the test suite.
* The `build-binary` job, which caches the current build for the next run of `test-binary`. Note that this should only run after all tests have succeeded. This job overwrites the previous cached binary, so we want to minimize the possibility that CI needs to be restarted.

## `test-binary`

Source: [scripts/test-binary](https://github.com/maidsafe/safe_client_libs/blob/master/scripts/test-binary)

Steps:

1. This job requires that a compatibility test binary has already built. It will check the CI cache to grab the previous binary; if there is none, this script will exit with success as there is nothing to compare against.

1. After grabbing the previous binary, this job unsets the `SAFE_MOCK_IN_MEMORY_STORAGE` env var and sets `SAFE_MOCK_VAULT_PATH=$HOME/tmp`. This is necessary so that data written by one binary can be read by the other, using the same vault.

1. Next, the current binary, built on the latest code, runs the `serialisation_write_data` test, writing some data to the mock vault. The previous cached binary then runs `serialisation_read_data` to try to read the same data from the mock vault. If this succeeds, it means that data compatibility has been preserved.

1. The previous step is run in reverse: the previous cached binary writes some data, and the current build tries to read it.

**Note:** the `serialisation_write_data` and `serialisation_read_data` tests are marked `#[ignore]` so they only run during compatibility tests and never during normal testing.

## `build-binary`

Source: [scripts/build-binary](https://github.com/maidsafe/safe_client_libs/blob/master/scripts/build-binary)

Steps:

1. This job runs the `cargo test` command with the `--no-run` option, which builds the tests without running them. Note that this step should complete instantaneously, as this job occurs late in the CI pipeline when the tests have already been built.

1. Next we use a complicated bash command to find the location of the latest binary in `target/`. `chmod +x` is run just in case.

1. The latest binary is copied to the `master/` directory in the CI cache (full path `${HOME}/.cache/master/`, filename `tests`). This path and filename can also be passed in as arguments to the script, as is the case when running from Docker.