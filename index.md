Intended to define the policy that will be followed for vendored node modules and how it will be verified.

### Non-goals
Current explicit non-goals of this policy include:

* Full code review of all modules added to node_modules in mozilla-central.
* Covering node usage in repositories outside of mozilla-central and the integration branches, even if they may be used to generate artifacts that eventually land in mozilla-central.

### Policy
* Node modules may only be used at build time or before (e.g. automated scripts and linting tools).
  **
<details><summary>Details</summary>
Ultimately, we’d like to support node_modules packages for automated testing, and potentially for runtime inclusion as well, but to limit the scope of the initial vendoring project, the initial license compatibility list will be chosen with building (and testing?) in mind. This policy will be documented in the in-tree reviewer documentation, and emitted as part of the “mach vendor node” command.
</details>

### Why vendoring?
Builds work even when not connected to the network (i.e. in Continuous Integration).
Support build-team work towards fully-reproducible builds for security reasons
Avoid developer debugging pain because we get deterministic node package versions for a given checkout without having to remember to `npm install`
Keep CI (and local, to some degree) build resource usage down by avoiding mandatory package install step.]

Introduction or updates of vendored modules require review by a nodejs peer.
Any changes to the vendored node modules should be landed as a standalone changeset also containing any changes required to keep the tree building with an appropriate description describing the need and review results. 
[Aside: The intent here is to maintain a working tree across all related commits so that VCS bisect functionality continues to work..]
All vendored node modules (and the entire dependency tree) must be licensed under acceptable licenses based on the licensing runbook.
Any introduction of modules including cryptographic code requires additional review from a cryptographic expert.
Modules including binary code will only be approved in special circumstances.
[Aside: The primary intent here is to avoid the implementation complexity needed for multiple platform-specific binaries in the vendored tree.  This will be handled by having `mach vendor node` pass `--ignore-scripts` to `npm`.  Note that the failure modes of that switch are package-dependent, which could lead to unexpected build behaviors/failures.  We may wish to have the vendoring code and reviewer docs emit a message suggestion manually inspecting the ignore scripts to avoid this.
A bit of relevant discussion here: 
<ted> I had to jump through some hoops to deal with a similar issue with Python modules: https://bugzilla.mozilla.org/show_bug.cgi?id=1481612]
<gps> And to add what Ted said, we did end up vendoring a binary Python wheel to work around clients not being able to compile Python C extensions (especially on Windows, where Python 2.7 requires an ancient MSVC toolchain).
So we can vendor binary files in some limited circumstances. But it really scares me and should be done sparingly.
If we can compile Node modules easily enough, doing that wouldn't be impossible. But the devil with compiling is likely in the unvendored source dependencies. That leaves us with a bootstrap problem or we bite the bullet and vendor all 3rd party software dependencies so people can compile everything. That would make mozilla-central more hermetic, which I fully support. But we've been unwilling to cross that bridge for various reasons. Partial clones in VCS land will make a significant blocker go away. So we can revisit this, I reckon.]
Avoid importing tools with their own build-systems that don’t expose their build Directed Acyclic Graphs (i.e. the graph of all dependencies from inputs to outputs).

[Aside: The Firefox build system is moving towards a world where as much of the build dependency graph as possible is well-defined and deterministic, as well as efficient for incremental builds.  As such,  we’d like to try as hard as possible to avoid introducing tools that provide their own build-systems that don’t provide a way to export their DAG to be part of the larger Firefox DAG.  

This is something we’ll need to rely on both submitters and reviewers to catch, though (per Ted), BTup (now tier 1 in Treeherder) is likely to break if node_scripts do something wrong here.

Existing moz.build script expects called node scripts to output dependencies one per line, with “dep:” at the beginning of such lines.  Other schemes could be considered.]
Vulnerability response
When Mozilla are made aware of a vulnerability in a vendored node module either via public announcement or private disclosure it is generally expected that the team using the module in question will be responsible for determining the best solution available. The NodeJS module peers will generally act in an advisory role helping where necessary with understanding the threats presented by a vulnerability. If the feature in question is unowned then the NodeJS peers may take a more active role in finding a solution.
[Aside: the NodeJS peers should not be considered as owning all of the vendored node module code.]
Package selection
While not fixed requirements these are a list of things to consider when choosing a module to vendor:
Size of module and dependency tree.
Update frequency.
Test coverage
Responsiveness to bugs and security vulnerabilities.
Random sample to evaluate likely code quality.
Does this inject any code into the final build or have any other inherent security risks?
Any general opinions of the module found in the Node community.
Automated tools
A set of automated systems will be employed to verify that all vendored modules continue to meet the policy.
mach lint node: Primarily intended for CI, runs checks that no modules or dependencies use a disallowed license or have known security vulnerabilities. Verifies that package-lock.json is valid (does not pull code from unexpected registries).
[Aside:  For security vulnerabilities this will rely on npm audit since snyk requires an account and we shouldn’t expect developers to need to sign up to run basic checks.]
Automated jobs run on treeherder for new phabricator revisions that affect node files and run periodically to verify existing modules.
[Aside:  This will include the tests from the above lint checks as well as using snyk (which means there will have to be an account that our CI can use).]
mach vendor node: Adds a new module to the devDependencies of package.json and validates it and its dependency tree as above.

Herald rule to automatically add nodejs-peers as a blocking reviewer to any revisions that modify node_modules, package.json or package-lock.json.
