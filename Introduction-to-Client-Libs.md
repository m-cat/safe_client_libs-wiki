Welcome to SAFE Client Libs!

We are:

- [Marcin Swieczkowski](https://github.com/m-cat)
- [Lionel Faber](https://github.com/lionel1704/)
- [Nikita Baksalyar](https://github.com/nbaksalyar)
- [Yogeshwar Murugan](https://github.com/Yoga07/)

If you have any questions about any of the libraries, we are happy to help!

## About

SAFE Client Libs is a set of libraries providing a way for developers to consume and use the SAFE Network facilities. The libraries communicate with [Vaults](https://github.com/maidsafe/safe_vault) and build upon the foundation of fundamental network components, such as [Crust](https://github.com/maidsafe/crust) and [Routing](https://github.com/maidsafe/routing), to provide higher-level network abstractions like files or directories. 

Because it's one of our user-facing components, it's very important for us to make it as understandable and developer-friendly as possible. The goal of this document is to provide an introduction understandable by most people, even with little prior knowledge.

### Components

The Client Libs are written entirely in Rust. The main components are the following:

- [SAFE Core](https://github.com/maidsafe/safe_client_libs/tree/master/safe_core) [![](http://meritbadge.herokuapp.com/safe_core)](https://crates.io/crates/safe_core) -- This library provides high-level building blocks that are based on the network primitives. For example, Routing does not deal with user accounts or file systems on the network -- that is what SAFE Core provides.
- [SAFE App](https://github.com/maidsafe/safe_client_libs/tree/master/safe_app) [![](http://meritbadge.herokuapp.com/safe_app)](https://crates.io/crates/safe_app) -- This is a library that apps running on the SAFE Network use. It provides essential IPC communication with the Authenticator, API functions to work with the network, and also a foreign interface for other languages.
- [SAFE Authenticator](https://github.com/maidsafe/safe_client_libs/tree/master/safe_authenticator) [![](http://meritbadge.herokuapp.com/safe_authenticator)](https://crates.io/crates/safe_authenticator) -- This library handles the Authenticator functions, which include users account registration, log in, apps registration, apps revocation, and communication with apps through the IPC mechanism.

The [main repository](https://github.com/maidsafe/safe_client_libs) contains all three of the main crates split into their own subdirectories. It follows a common pattern found in Rust projects called a [workspace](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html). We use this project structure because the crates of SAFE Client Libs are inter-dependent: both SAFE App and SAFE Authenticator depend on SAFE Core, and SAFE App depends on SAFE Authenticator in the testing mode.

It is also important to be aware of differences between the [mock and non-mock](./Mock-vs.-non-mock) versions of the libraries.

To build the project, you can follow the instructions from the [Building Client Libs](./Building-Client-Libs) page.

### Auxiliary libraries

There are also several auxiliary libraries that belong to the family of SAFE Client Libs:

- [FFI Utils](https://github.com/maidsafe/ffi_utils): Provides helpers to work with foreign function interface more easily. Also contains many helpful testing tools that depend on our [FFI calling conventions](./FFI-calling-conventions).
- [SAFE Bindgen](https://github.com/maidsafe/safe_bindgen): Automatically generates language bindings for C, Java, and C# based on the user API.
- [System URI](https://github.com/maidsafe/system_uri): Helps to setup custom URI handlers on different operating systems (Windows, Mac, Linux). Required for a proper functioning of the [IPC mechanism](./IPC-mechanism).

## Project management and communication

- We use [GitHub Issues](https://github.com/maidsafe/safe_client_libs/issues) and [project boards](https://github.com/maidsafe/safe_client_libs/projects) to track and manage the tasks we're working on.
- We post Dev Updates every Thursday on the [main forums](https://safenetforum.org/). These updates recap the work we had been doing since the last update and the primary purpose of them is to communicate with our community and SAFE app developers.

## Getting started

Client Libs can feel overwhelming and very complex at first, but the good news is that most of the complexity comes from the number of components, not the underlying concepts.

Here are some things you might want to know before you start:

### Building, configuring, and testing SAFE Client Libs

To learn about building your own version of SAFE Client Libs, please follow the relevant page, [Building Client Libs](./Building-Client-Libs). It contains thorough build instructions and explains the difference between various build features. You should get several outputs as a result of compilation:

- The *dynamic C libraries* (`libsafe_app.so` and `libsafe_authenticator.so`), which are used by JavaScript, Java, C\#, and other languages to interact with the SAFE Network.
- Rust libraries, which can be used to create new examples, tests, and SAFE apps in Rust.

There are also multiple [different configuration](./Configuring-Client-Libs) options for Client Libs that you should be aware of.

### Documentation

For an essential documentation, check out the other wiki pages in the sidebar, which describe the high-level abstractions and concepts in Client Libs.

We also have API-level documentation for our libraries, found on docs.rs:

- [SAFE Core](https://docs.rs/safe_core/)
- [SAFE App](https://docs.rs/safe_app/)
- [SAFE Authenticator](https://docs.rs/safe_authenticator/)

You can also find some [examples in the `safe_app`](https://github.com/maidsafe/safe_client_libs/tree/master/safe_app/examples) project that can give you some taste & feel of our API.

## Development guide

We follow the common development process in the company. We develop new features in separate git branches, raise pull requests, put them under peer review, and we merge them only after they pass the QA checks and continuous integration. We never commit directly to the `master` branch.

The deployment happens automatically once a pull request is merged. Intermediate versions of the libraries are published on S3 for all the platforms we support as Tier 1 (Windows, Ubuntu 18.04, OS X, + Android & iOS). There's a special kind of pull request and deployment trigger called *version publish*: we use it when we release a new version of SAFE Client Libs, and it happens when the pull request title follows a special pattern ([for example](https://github.com/maidsafe/safe_client_libs/pull/686)). We should always be sure to update respective *change logs* before doing a version publish (e.g. a [change log](https://github.com/maidsafe/safe_client_libs/blob/master/safe_app/CHANGELOG.md) for safe\_app).

We strive to have full test coverage of all Client Libs features. To help with this goal, we created a testing framework and multiple custom-made testing utilities for SAFE Client Libs. You can learn more about it from the [Testing framework](./Testing-framework) page.

## Contributing

For full instructions on contributing, including code style details, please see the [Guide to contributing](./Guide-to-contributing) page.

## Useful resources

In Client Libs we heavily rely on some of systems development concepts, so it might be very useful to know more about memory allocation, operating systems, and the C programming language. This section lists some of the resources on these topics you might find helpful.

### Articles

- [Linked Data: The Story so Far](http://tomheath.com/papers/bizer-heath-berners-lee-ijswis-linked-data.pdf): a good introduction to RDF and Linked Data concepts, from Sir Tim Berners-Lee himself. Please read this before delving more deeply into the RDF-related topics.

### Presentations

- [Memory Management in Unsafe Rust](https://www.bytedude.com/files/unsafe-rust.html), slides for the full talk given by Marcin within the company.

### Books

- [The C Programming Language 2e (K&amp;R)](https://www.amazon.co.uk/C-Programming-Language-2nd/dp/0131103628): one of the best books to learn the C language from. Very relevant to the FFI aspect of Client Libs (as it conforms to the C ABI).
- [The Linux Programming Interface](http://man7.org/tlpi/): a (big) book that describes most of the Linux system calls, starting from the very basics. It's useful way beyond just Linux.
- [Language Implementation Patterns](https://pragprog.com/book/tpdsl/language-implementation-patterns): a practical book about building simple compilers (for configuration file parsing, source translators, etc.). Very relevant to what we do for Safe-Bindgen.
- [Learning SPARQL](http://shop.oreilly.com/product/0636920020547.do): a great book that covers a lot of topics in RDF, linked data, and SPARQL.
