# Detecting and resolving dependency vulnerabilities in Gradle projects

This is a simple project demonstrating how to use the `dependency-submission` GitHub action to detect
vulnerable dependencies in a Gradle project, and various techniques to address these vulnerabilities.

You may find it useful to fork this repository, which will allow you to follow this guide, viewing and resolve Dependabot alerts.
Note that GitHub Actions workflows are not automatically enabled for repository forks. 
To start the process, you'll need to:
1. Fork the repository
2. Navigate to "Settings -> Code security and analysis" to enable Dependency graph and Dependabot alerts (see below)
3. Navigate to the "Actions" tab to enable GitHub Actions workflows
4. Push a commit to the 'main' branch in order to trigger the initial `dependency-submission` workflow to run. A change to the README will be sufficient.

# Setting up the repository to detect vulnerable dependencies

In order to receive alerts about any vulnerable dependencies for this repository:

1. **Dependency graph and Dependabot alerts are enabled**

<img width="800" alt="image" src="https://github.com/gradle/github-dependency-submission-demo/assets/179734/7ecf6cc6-aec2-4985-8331-b9e9db78d312">

2. **A simple `dependency-submission` workflow is configured to run on any push to the `main` branch**

https://github.com/gradle/github-dependency-submission-demo/blob/a353fcb9c6a17d937f5988d95d10e28f1994fe03/.github/workflows/dependency-submission.yml#L1-L30

Note that the workflow is configured to publish a [Build Scan®](https://scans.gradle.com/#gradle) on each run. 
This Build Scan will contain information about your project structure and dependencies, 
and will be useful later as we attempt to identify and resolve any vulnerable dependencies.

See the [full dependency-submission documentatio](https://github.com/gradle/actions/blob/main/dependency-submission/README.md)n for more details on adding a dependency-submission workflow.

# Reviewing vulnerabilities reported for this repository

After executing the `dependency-submission` workflow, the repository has 5 current Dependabot alerts for vulnerable dependencies. 
These are not publicly visible in the repo, but here is the list:

<img width="800" alt="image" src="https://github.com/gradle/github-dependency-submission-demo/assets/179734/1e9905c6-d1cf-40ea-bdc1-d42b70eafefe">

In the following sections we will step through the process of investigating, isolating and addressing different types of vulnerabilities.

- [Updating a direct dependency to non-vulnerable version](#updating-a-direct-dependency-to-non-vulnerable-version)
- [Updating a vulnerable transitive dependency by updating a direct dependency](#updating-a-vulnerable-transitive-dependency-by-updating-a-direct-dependency)
- [Updating a transitive dependency using a dependency constraint](#updating-a-transitive-dependency-using-a-dependency-constraint)
- [Updating a Plugin classpath dependency using a dependency constraint](#updating-a-plugin-classpath-dependency-using-a-dependency-constraint)

# Updating a direct dependency to non-vulnerable version

In this example repository, a dependency on `org.apache.commons:commons-text:1.9` results in the following
Dependabot alert:

<img width="800" alt="image" src="https://github.com/gradle/github-dependency-submission-demo/assets/179734/fab87384-d5c6-4f9a-8206-75b4256a82c3">

In this simple case, the vulnerable dependency is declared directly in the Gradle project,
and a newer, non-vulnerable version is available. 

The fix is as simple as bumping the version in the project.
[Here is a pull-request](https://github.com/gradle/github-dependency-submission-demo/pull/1/files) that will address this vulnerability.

<img width="800" alt="image" src="https://github.com/gradle/github-dependency-submission-demo/assets/179734/f6821ca1-e5fe-4894-bc5c-0944357cf6df">

# Updating a vulnerable transitive dependency by updating a direct dependency

We see 2 vulnerabilities reported for `org.apache.commons:commons-compress:1.24.0`, like this:

<img width="800" alt="image" src="https://github.com/bigdaz/dependency-submission-demo/assets/179734/6d5bc043-6ff6-46f3-9166-d46096fe6bac">

But there isn't anywhere in our Gradle project where we add depend on `commons-compress`, which means this vulnerability
must involve a _transitive_ dependency. The first step is to identify which direct dependency is responsible.

The easiest way to do this is with a Gradle Build Scan®, which is why our workflow is
configured to automatically publish a Build Scan for every dependency submission.
You'll find the build-scan link in the [summary for each workflow run](https://github.com/gradle/github-dependency-submission-demo/actions/runs/8557095342#summary-23448497383).

<img width="800" alt="image" src="https://github.com/gradle/github-dependency-submission-demo/assets/179734/e7aafb1c-6821-4f7f-8800-b5f3ff33ef0a">

By inspecting [the Build Scan for the latest submission on 'main'](https://scans.gradle.com/s/7r7ceoclg3kky/dependencies?dependencies=commons-compress&expandAll&focusedDependency=WzAsMSwyLFswLDAsWzBdXV0&focusedDependencyView=versions), we can see that `commons-compress` is required by `io.minio:minio:8.5.8`, which _is_ a direct dependency of our project.

<img width="800" alt="image" src="https://github.com/gradle/github-dependency-submission-demo/assets/179734/dea559d9-0d54-41df-8cf2-81625d62b0a9">


Searching for newer versions reveals that there's an updated version of `minio` available (`8.5.9`),
and when we update our project to this version, [the commons-compress library is updated to `1.26.0`](https://scans.gradle.com/s/bfkkg66rwrbbw/dependencies?dependencies=commons-compress&expandAll&focusedDependency=WzAsMSw3LFswLDAsWzBdXV0&focusedDependencyView=versions).

Here's [the pull-request](https://github.com/gradle/github-dependency-submission-demo/pull/2/files) that will update the version of `minio`, 
resolving the 2 Dependabot alerts triggered by `org.apache.commons:commons-compress:1.24.0`.

<img width="800" alt="image" src="https://github.com/gradle/github-dependency-submission-demo/assets/179734/56e1290f-b92c-4f35-80cc-5797e56b036b">

# Updating a transitive dependency using a dependency constraint

At times when you won't be able to update a direct dependency to resolve a transitive dependency vulnerability.
This can occur if there isn't a newer version of the direct dependency available, or perhaps your project isn't
compatible with the newer version.

In this case, you can directly control the transitive dependency version using a [dependency constraint](https://docs.gradle.org/current/userguide/dependency_constraints.html).
A dependency contraint allows you to specify a transitive dependency version to use, without adding a direct dependency on that version.

[This pull-request](https://github.com/gradle/github-dependency-submission-demo/pull/3/files) adds a dependency constraint that causes the build 
to use a newer version of `commons-compress`, thereby resolving the Dependabot alert. 

<img width="800" alt="image" src="https://github.com/gradle/github-dependency-submission-demo/assets/179734/c05e16d7-32e6-4b08-987b-9cf1215006e5">

If you inspect [the resulting Build Scan](https://scans.gradle.com/s/feschz3ywyb4c/dependencies?dependencies=commons-compress&expandAll&focusedDependency=WzAsMSw1LFswLDAsWzFdXV0&focusedDependencyView=versions), 
you can see that the version of `commons-compress` is updated, but the version of `minio` is not changed.

<img width="800" alt="image" src="https://github.com/gradle/github-dependency-submission-demo/assets/179734/4b153141-b67c-4852-a2a0-f466cc07fa5e">

# Updating a Plugin classpath dependency using a dependency constraint

In addition to things you declare in a `dependencies` block, any Gradle plugins that you apply to your build
will likely have library dependencies. 
Vulnerabilities in these plugin dependencies are detected by the `dependency-submission` action.

The final 2 Dependabot alerts in this project are due to `com.squareup.okio:okio-jvm:3.2.0` and `com.squareup.okio:okio:3.2.0`. 

<img width="800" alt="image" src="https://github.com/bigdaz/dependency-submission-demo/assets/179734/c9ca7b7b-2f07-4647-9201-7f00a0c167ce">

When inspecting the Build Scan for these vulnerable versions, [we note that they are not listed in the 
"Dependencies" section](https://scans.gradle.com/s/2hwggowm3vyts/dependencies?dependencies=okio&expandAll).
This is because these vulnerable versions are actually brought in by the `com.github.ben-manes.versions` plugin: 
you can see this by [searching for 'okio' in the "Build Dependencies" section of the Build Scan](https://scans.gradle.com/s/2hwggowm3vyts/build-dependencies?dependencies=okio&expandAll&focusedDependency=WzAsMCwzMCxbMCwwLFszXV1d&focusedDependencyView=versions).

<img width="800" alt="image" src="https://github.com/gradle/github-dependency-submission-demo/assets/179734/1286e7b4-fdc1-4ed1-b694-4dbf26293afd">

Although vulnerable plugin dependencies like this can be trickier to identify, they can be resolved in much the same way as
regular transitive dependencies, either by updating to a new version of the plugin, or by using a dependency constraint.

In this case there is no newer version of the plugin available, so we must add a dependency constraint to force a newer version of `okio` to be used.

[This pull request](https://github.com/gradle/github-dependency-submission-demo/pull/4/files) adds a dependency constraint to the `buildscript` classpath, w
hich fixes the security vulnerablitity in `okio` that is introduced by the `com.github.ben-manes.versions` plugin.

<img width="800" alt="image" src="https://github.com/gradle/github-dependency-submission-demo/assets/179734/f1b3420b-668c-4d24-8e73-7f35e998479e">

The [resulting Build Scan demonstrates](https://scans.gradle.com/s/2hwggowm3vyts/build-dependencies?dependencies=okio&expandAll) that the build dependencies no longer include the vulnerable dependency versions.

<img width="800" alt="image" src="https://github.com/gradle/github-dependency-submission-demo/assets/179734/4dbb6680-3256-42d4-bd65-a0f587b29b4a">

