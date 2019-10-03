Want to contribute? Great! There are many ways to give back to the project, whether it be writing new code, fixing bugs, or just reporting errors. All forms of contribution are encouraged!

## Reporting issues

Notice something amiss? Have an idea for a new feature? Feel free to to write an Issue on the [GitHub repository](https://github.com/maidsafe/safe_client_libs/issues) about anything that you feel could be fixed or improved. Examples include:

- Unclear documentation
- Bugs, crashes
- Enhancement ideas
- ... and more

Please try to be as descriptive as possible. Provide as much information as you are able to (e.g. Rust version, operating system, Client Libs version). If you are seeing an error, paste as much of the log as you can. We don't currently have a template for Issues, but may add one in the future to help you structure your Issues and provide useful information.

Of course, after submitting an Issue you are free to tackle the problem yourself (see "Submitting patches" below), and encouraged to do so.

## Choosing an Issue

If you want to contribute code but don't know what to work on, we try to always keep some Issues marked [`help-wanted`](https://github.com/maidsafe/safe_client_libs/issues?q=is%3Aissue+label%3A%22help+wanted%22+is%3Aopen) or [`good-first-issue`](https://github.com/maidsafe/safe_client_libs/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22). We also have the tag [`easy`](https://github.com/maidsafe/safe_client_libs/issues?q=is%3Aissue+is%3Aopen+label%3Aeasy) for categorizing Issues that are fairly low-effort, which you can tackle if you are short on time or new to programming. We want everyone to be able to contribute!

## Development

We follow the common development process in the company. We use [git](https://git-scm.com/) as our [version control system](https://en.wikipedia.org/wiki/Version_control) (VCS). We develop new features in separate git branches, raise [Pull Requests](https://help.github.com/en/articles/about-pull-requests), put them under peer review, and we merge them only after they pass the QA checks and [continuous integration](https://en.wikipedia.org/wiki/Continuous_integration) (CI). We never commit directly to the `master` branch.

For useful resources, please see:

- [Git basics](https://git-scm.com/book/en/v1/Getting-Started-Git-Basics) for Git beginners
- [Git best practices](https://sethrobertson.github.io/GitBestPractices/)

### Code Style

We are following the company-wide code style guide that you can find in the [the Rust Style document](https://github.com/maidsafe/QA/blob/master/Documentation/Rust%20Style.md). You should install `rustfmt` and `clippy` and run them before each of your Git commits, and you should follow the common commit message style (for an example you can take a look at the list of commits in [our repository](https://github.com/maidsafe/safe_client_libs/commits/master).

In addition to the common code style, we have our own [coding convention](./FFI+calling+conventions) that we use for the FFI (functions interfacing with other languages).

## Submitting patches (Pull Requests)

If you are a complete newbie, click [here](https://github.com/firstcontributions/first-contributions) for an easy-to-follow guide (with pictures!)

We follow the standard procedure for submitting Pull Requests. Please refer to the [official GitHub documentation](https://help.github.com/articles/creating-a-pull-request/) if you are unfamiliar with the procedure. If you still need help, we are more than happy to guide you along!

**Note:** Before submitting code, please either choose an existing Issue, or write your own, describing the problem you are solving. We would like for every PR to have an accompanying Issue, so that we know concretely what the problem is and can track its resolution.

### Format

The PR title and git commit messages should follow the format specified in [the Rust Style document](https://github.com/maidsafe/QA/blob/master/Documentation/Rust%20Style.md#git-commit-messages). We don't have any hard-and-fast rules for the PR body or branch name, but try to look at existing PRs first to get a sense of how they are structured.

### Running tests (CI script)

Submitted PRs are expected to pass continuous integration (CI), which, among other things, runs a test suite on your PR to make sure that your code doesn't have any bugs.

Please refer to [Testing Client Libs](./Testing-Client-Libs) and ensure you have appropriately and comprehensively run tests on your branch before submitting.

### Code review

Your PR will be automatically assigned to the team member specified in the `codeowners` file, who may either review the PR himself or assign it to another team member. More often than not, a code submission will be met with review comments and changes requested. It's nothing personal, but nobody's perfect; we leave each other review comments all the time.

After addressing review comments, it is up to your discretion whether you make a new commit or amend the previous one and do a [force-push](https://estl.tech/a-gentler-force-push-on-git-force-with-lease-fb15701218df) to the PR branch. We encourage amending the last commit if there are only minor changes to be made. GitHub now shows diffs for force-pushed commits, making them easy to review, and it keeps the final commit history clean.

### Deployment

Deployment happens automatically once the test suite has passed and the Pull Request has been merged. Intermediate versions of the libraries are published on S3 for all the platforms we support as Tier 1 (Windows, Ubuntu 18.04, OS X, + Android & iOS).

### Version updates

There's a special kind of Pull Request and deployment trigger called *version publish*: we use it when we release a new version of SAFE Client Libs, and it happens when the Pull Request title follows a special pattern ([for example](https://github.com/maidsafe/safe_client_libs/pull/686)). We should always be sure to update respective *change logs* before doing a version publish (e.g. a [change log](https://github.com/maidsafe/safe_client_libs/blob/master/safe_app/CHANGELOG.md) for Safe App).