:tocdepth: 1

.. sectnum::

Abstract
========

Regularly updating dependencies for SQuaRE services is the highest-impact security intervention.
It also reduces the operational burden in the long term by making frequent, controlled, incremental updates rather than large and hard-to-debug flag days.
This document explores options for dependency management and proposes a hybrid system using Dependabot and WhiteSource Renovate.

Problem statement
=================

Security reviews of both SQuaRE internal services (`SQR-037`_) and the Science Platform (`SQR-041`_) identified security patching and upgrades of third-party software as one of the highest security risks and most impactful mitigations.
Determining whether a given update of third-party software fixes a security issue can be challenging; where possible, keeping all dependencies current is the ideal approach.
Regularly updating all dependencies spreads out upgrade work more smoothly, results in fewer flag days, and avoids getting stuck on an outdated, unsupported version when a security vulnerability is discovered.
For more discussion of this approach, see `Software dependencies and keeping them up-to-date <https://www.netcetera.com/home/stories/expertise/20170406-software-updates-inside-it.html>`__, among other articles.

.. _SQR-041: https://sqr-041.lsst.io/
.. _SQR-037: https://sqr-037.lsst.io/

Continuous dependency updates would be challenging or infeasible for the stack code underlying astronomical data processing for Vera C. Rubin Observatory.
Updating that code requires extensive validation and therefore must be carefully planned.
Thankfully, however, most of that software is not part of the security attack surface of SQuaRE or Science Platform services, and many services run by SQuaRE do not have the same challenges.
We therefore intend to regularly update all third-party software dependencies for all SQuaRE-managed services by default, making exceptions only where this would interfere with reproducible astronomical data processing or cause instability problems.

Anything done regularly should be automated.
That automation should come in the form of GitHub pull requests, which can then trigger normal test suites for individual repositories to catch incompatibilities.
To begin with, all pull requests will be manually reviewed.
In the longer term, some pull requests should be merged automatically if tests pass (updates of test-only dependencies such as ``pytest`` or ``mypy``, for example).

Types of dependencies
---------------------

The following types of external dependencies are present in SQuaRE services:

- Python PyPI packages
- JavaScript NPM packages
- Ruby gems
- Helm charts (both third-party and SQuaRE-internal charts)
- Kustomize external resources
- Docker images referenced in Helm charts and Kustomize resources
- Docker images used as a base in ``Dockerfile`` files
- Docker images referenced in ``docker-compose`` configuration files
- GitHub Actions referenced in GitHub Workflows
- pre-commit plugins for Python packages

Other desired features
----------------------

- Free for public repositories on GitHub.
- Bundle multiple low-risk updates into a single pull request.
- Support frozen requirements files for Python applications, which requires updating the frozen transitive dependencies as well.
- Cloud service preferred over self-hosting.

Survey of available software
============================

A short survey via Google uncovered the following services for automated dependency management:

- `Dependabot <https://dependabot.com/>`__
- `Depfu <https://depfu.com/>`__
- `Snyk <https://snyk.io/>`__
- `WhiteSource Renovate <https://renovate.whitesourcesoftware.com/>`__

Depfu only advertises support for Ruby, JavaScript, and Elixir on their web site, which doesn't meet the requirements.
I experimented with the remaining three services.

Dependabot
----------

Dependabot has been acquired by GitHub and is free for public repositories.
GitHub makes its security updates easily available through GitHub repository settings, but it can also send PRs for every out-of-date dependency.
It supports Python, JavaScript, Ruby, ``Dockerfile`` files, and GitHub Actions, but not the other types of dependencies of interest to SQuaRE.
Every dependency update is sent as a separate PR, so its flood of PRs can feel a bit overwhelming.

Dependabot can handle the PIP frozen requirements files, although it updates each dependency (including transitive dependencies) separately.
Since frozen dependencies require updating the entire dependency chain, this could potentially result in more than a PR per day per repository.

If packages use the new ``.github/dependabot.yml`` configuration mechanism, Dependabot results are available directly in the GitHub repository UI.

Dependabot is the least likely to disappear in the future, since it is a GitHub first-party service now.
For the same reason, it's likely to get continued development.
Dependabot is hosted by GitHub as a cloud service.

Snyk
----

Snyk is primarily a security scanner rather than a dependency update automation.
It attempts to find security vulnerabilities in a package, including via dependencies, and reports on them.
It can generate pull requests to fix dependencies with security vulnerabilities, but at least in the free version this process seems fairly manual.
Multiple dependency updates are batched into a single pull request.
However, the generated PR only pins dependencies to greater than the last security vulnerability, rather than to a specific version.
It does support frozen PIP dependency files, but does not detect out of date dependencies with no security implications.

Synk can analyze Kubernetes resources and Helm charts, but appears to only look for potential security issues, not for out-of-date dependencies.

The security scanning capabilities of Synk look useful, but it doesn't have the right set of features for automated dependency management.

WhiteSource Renovate
--------------------

Renovate is a freemium service also available as an `open source version <https://github.com/renovatebot/renovate>`__.
It can be run as a command-line tool or as a Docker image (with Kubernetes support), or is available as a `GitHub App <https://github.com/marketplace/renovate>`__ managed by WhiteSource.

Compared to the other alternatives, it has excellent Kubernetes support.
It checks Helm chart dependencies and Docker image dependencies in ``values.yaml`` files.
It also has excellent Docker support, checking for dependencies in ``Dockerfile`` and ``docker-compose.yaml`` files.
It advertises Kustomize support as well, but unfortunately the current release as of this writing appears not to properly support Kustomize resources where the ``kustomize.yaml`` file is not in the root directory of the repository.

Renovate appears to not support ``pyproject.toml`` or ``setup.cfg`` dependencies.
It has experimental support for ``setup.py`` dependencies.
Dependabot supports all three.
Unfortunately, Renovate also does not support PIP ``requirements.txt`` files with pinned hashes, which is our standard for Python applications.

Unlike Dependabot, Renovate can be configured to batch dependency updates into a single PR.
This is highly configurable and can be done by language, path, package name, or other criteria.
Renovate can also be configured to automatically merge some updates based on criteria such as package name or major versus minor version update.

Renovate supports almost everything that Dependabot supports except for GitHub Actions.
Neither supports pre-commit hook dependencies.
I did not test Renovate support for NPM or Ruby gem dependencies.

Since Renovate is a freemium product from a startup, there is some risk that WhiteSource will remove or limit their free tier in the future or cease open source development, particularly given that they are now competing against a first-party GitHub service.
However, the open source version could potentially be maintained by the community.

Recommendations
===============

We have two main options:

#. Use a single system and live with the features that it's missing.
#. Use some mix of multiple systems.

We can also supplement either approach with code we write and maintain ourselves.

Renovate is the closest to being the single system that could do everything desired, since it has Kubernetes support.
It also allows grouping of PRs, which is a major feature, and allows some PRs to be automatically merged.
However, the lack of support for frozen PIP requirements files is a huge gap, since we expect to use that mechanism going forward for all SQuaRE microservices
There is also a bug in the handling of Kustomize manifests.
We could contribute that support, but that would require doing significant JavaScript development.

Dependabot is attractive because of the strong GitHub integration and first-party support, and because it handles frozen PIP requirements files.
It's also very simple to set up and configure.
Renovate isn't too bad, but it's a bit more complex.
However, it doesn't support Kubernetes, which is a significant enough gap that we would need to supplement it with our own code.
Its inability to group updates into a single PR for packages with frozen PIP dependencies is also likely to be overwhelming.

The combination of Dependabot and Renovate covers all of the types of dependencies except for Kustomize resources, pre-commit plugins.
However, that would mean using Dependabot for Python dependencies for applications with frozen PIP requirements files and dealing with large numbers of PRs.

The best approach therefore seems to be to use Dependabot for Python library and GitHub Actions dependencies, Renovate for Kubernetes and Docker dependencies, and custom code for frozen PIP requirements, Kustomize resources, and pre-commit hooks.
This requires configuring two systems, but we can template the configuration files.
They shouldn't require much maintenance work.
We can choose which of the two systems to use for the dependencies that both systems support based on how well they handle them.
Given the current implementations, Renovate is the best default choice because of the ability to configure automatic merging and to group multiple updates into a single PR.

This unfortunately requires us to maintain our own additional code to handle Python applications, Kustomize, and pre-commit dependencies (although we could skip management of pre-commit dependencies without much risk if we wanted to reduce the scope of that code).
This isn't a desirable option, but the alternatives (very frequent PRs from Dependabot or significant JavaScript development on Renovate) seem worse.
We can drop that code in the future if Dependabot adds support for batched PRs or Renovate adds better Python and Kustomize support.
Fixing Kustomize support is probably the easiest Renovate contribution, if we decide to start trying to contribute to the project.

Therefore, the recommendation is:

- Use Dependabot for Python library dependencies, GitHub Actions, and other language dependencies Renovate doesn't support.
- Use Renovate for all other dependencies except Python applications with frozen PIP requirements files, Kustomize resources, and pre-commit hooks.
- Write our own code to handle Python applications, Kustomize resources, and pre-commit hooks, with an eye to phasing it out if either of the other systems improve.
