In Client Libs we use **Continuous Integration**, or **CI**, to ensure the correctness and robustness of our released builds. You can think of CI as a pipeline of automated tasks that occur when a Pull Request is [raised or modified](https://github.com/maidsafe/safe_client_libs/wiki/Guide-to-contributing#submitting-patches-pull-requests) or when code is pushed to master. At the core of this pipeline is a full [build of the code](https://github.com/maidsafe/safe_client_libs/wiki/Building-Client-Libs) which makes sure that committed code is able to build without error. We also perform, among other things, a comprehensive [run of the test suite](https://github.com/maidsafe/safe_client_libs/wiki/Testing-Client-Libs) to make sure the new code did not break any tests.

## Benefits

The benefits of CI are manifold:

* CI provides automated and reproducible builds.
    * Team members are notified when a build fails.
    * Logs are kept so errors can be investigated.
    * The builds can run on a variety of environments, machines, and/or operating systems.
* Code does not enter *production* (the stage in the development cycle where the end-user actually runs the software) without all checks and tests having passed.
    * These checks add reliability and robustness to releases.
    * Developers do not need to remember to run formatting or linting tools; these can be done automatically.
* The final code is *deployed* (released to the public) automatically, reducing human error.

All of these points apply to Client Libs.

## Cloud CI

We use [Travis CI](https://travis-ci.com/) and [Appveyor](https://www.appveyor.com/) as cloud providers of Continuous Integration. Not maintaining our own dedicated CI server (which would entail handling updates, troubleshooting of outages, and so on) frees us up to focus on development. Both Travis and Appveyor provide their services *gratis* for open source projects such as the SAFE Network.

We use Travis for building and testing on Linux and OSX, and we use Appveyor for the equivalent on Windows. With Travis recently releasing experimental support for Rust builds on Windows, we are likely to drop Appveyor sometime soon, simplifying our CI process.

## The Travis file

Using Travis is quite simple, and only requires that you create an account with Travis CI and link it to your GitHub account or organization. Once you have done so, Travis will start checking incoming Pull Requests for your public repositories. If the committed code includes a `.travis.yml` file, then Travis will initiate a build of your code, following the provided settings in this file.

It's recommended that you have read and understood [this page](https://docs.travis-ci.com/user/for-beginners/) before continuing. Specifically, it's important to understand these terms:

> *phase* - the sequential steps of a job. For example, the `install` phase, comes before the `script` phase, which comes before the optional `deploy` phase.
>
> *job* - an automated process that clones your repository into a virtual environment and then carries out a series of phases such as compiling your code, running tests, etc. A job fails if the return code of the `script` phase is non zero.
>
> *build* - a group of jobs. For example, a build might have two jobs, each of which tests a project with a different version of a programming language. A build finishes when all of its jobs are finished.
>
> *stage* - a group of jobs that run in parallel as part of sequential build process composed of multiple stages.

Please also consult [the official docs](https://docs.travis-ci.com/).

## Travis in Client Libs

Due to the size and complexity of Client Libs, [our Travis file](https://github.com/maidsafe/safe_client_libs/blob/master/.travis.yml) is more involved than what you'd typically see for CI. Let's go through the structure of this file, examining the configuration sections, the sections defining the phases, and the different stages.

### Configuration

Our `.travis.yml` file contains the following sections for configuring the builds:

* `env`: This section defines global environment variables that Travis will set at the beginning of all jobs. We set the `rustc` optimization level here as well as enable backtraces for easier debugging of any errors that occur.
* `language`: By specifying `rust` here, we make sure that `rustc` and `rustup` are present in the environment.
* `rust`: The Rust version to use.
* `stages`: This section enumerates the build stages (see below).
* `jobs`: In this section we specify each job explicitly, including which stage it should run at, what OS it should run on, and what commands should run during its `script` phase (see below). In simple Travis files the jobs can be implicitly inferred from other settings (see the [Build matrix](https://docs.travis-ci.com/user/build-matrix/) page), but we specify them explicitly for more control.
* `cache`: We enable caching for dependencies that are built so that Travis can reuse them between builds. Due to the size and number of the artifacts, we double the amount of time allocated for Travis to fetch them.

### Phases

Our Travis file also contains the following sections which define the [*phases* that each job goes through](https://docs.travis-ci.com/user/job-lifecycle#the-job-lifecycle). Some phases, such as `deploy`, are only triggered under certain conditions, since they should not run for every job.

We specify the following custom phases, which run in this order:

* `before_script`: Before doing anything else, this phase will make sure that `cargo_install.sh` is present, as well as `cargo-prune` (required for the `before_cache` phase). Also, this phase checks that the last commit message is properly formed for version update PRs.
* `script`: This phase is defined separately for each job in the job include matrix (as opposed to the other phases, which are defined for *all* jobs). This allows each job to run different commands; for example, jobs in the `tests` stage will run different commands than jobs in the `deploy` stage. The actual commands are all found in scripts within the `scripts/` subdirectory, making the Travis file a bit shorter.
* `before_cache`: Before caching anything, we run `cargo prune` which removes some old versions of dependencies and prevents the build cache from growing too much in size. See [this repository](https://github.com/ustulation/cargo-prune) for more information.
* `before_deploy`: This phase gets the mock and non-mock build artifacts ready for deployment (release). It does so by calling the `package.rs` Rust script. This phase will only run if `deploy` runs, which itself only runs under certain conditions.
* `deploy`: Once the builds are ready, this phase pushes them to a specified bucket in AWS S3. This phase only runs on pushes to the `master` branch of the repository, and only if the `$RUN_DEPLOY` environment variable is set.
* `after_deploy`: This phase cleans up after the other deployment phases.
* `after_script`: Finally, this phase runs on pull requests to check if Cargo.lock should have been updated, and returns an error if it was not.

### Stages

We currently build in four stages, which run after each other -- that is, stage 2 begins after all jobs in stage 1 have finished successfully.

1. `warmup`: This stage warms up the cache by running real and mock builds. This helps subsequent stages take less time and prevents jobs from exceeding 60-minute running times, upon which they would timeout. You can also think of this as the *build* stage.
1. `tests`: This stage builds and runs as many [tests for the repository](https://github.com/maidsafe/safe_client_libs/wiki/Testing-Client-Libs) as is feasible, including tests which use the [mock network](https://github.com/maidsafe/safe_client_libs/wiki/Mock-vs.-non-mock) as well as some integration tests. Tests are run for both Linux and OSX. The reason we do not run *all* tests is because some tests require a connection to the real network. This stage also runs the `test-binary` portion of the [Binary compatibility tests](https://github.com/maidsafe/safe_client_libs/wiki/Binary-compatibility-tests), which tests binary compatibility between this and the last successful CI build. We only run the binary test on Linux.
1. `tests-binary-compat`: Next, we run the `build-binary` portion of the [Binary compatibility tests](https://github.com/maidsafe/safe_client_libs/wiki/Binary-compatibility-tests). This single job will cache the current build for the next time that `test-binary` is run, so it should run as late as possible, once other tests have completed successfully and the chances of the rest of the CI build failing are relatively low.
1. `deploy`: This stage deploys the build artifacts. It only runs on pushes to `master`, and not on PRs. This prevents code from being deployed until it has been officially released. This stage contains two jobs: one to deploy releases for Linux, the other, for OSX.

## The Appveyor file

Appveyor works similarly to Travis. Simply get an account, link it to your GitHub account or organization, and commit an `appveyor.yml` file.

## Appveyor in Client Libs

[Our Appveyor file](https://github.com/maidsafe/safe_client_libs/blob/master/appveyor.yml) is somewhat simpler than our Travis file. We do not have to repeat some of the heavy lifting that is already performed in Travis, such as running binary compatibility tests. All we have to do here is build for Windows, test the builds, and deploy the Windows releases. Much of this works in much the same way as Travis, except that there are no stages because we have not yet run into problems with jobs timing out and needing to be split up.

It is likely that we will move Windows builds to Travis, in which case this file will become obsolete.
