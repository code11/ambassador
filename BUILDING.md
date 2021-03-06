Building Ambassador
===================

If you just want to **use** Ambassador, check out https://www.getambassador.io/! You don't need to build anything, and in fact you shouldn't.

TL;DR
-----

```
git clone https://github.com/datawire/ambassador
# Development should be on branches based on the `develop` branch
git checkout develop
git checkout -b dev/my-branch-here
cd ambassador
# best to activate a Python 3 virtualenv here!
pip install -r dev-requirements.txt
make DOCKER_REGISTRY=-
```

That will build an Ambassador Docker image for you but not push it anywhere. To actually push the image into a registry so that you can use it for Kubernetes, set `DOCKER_REGISTRY` to a registry that you have permissions to push to.

**It is important to use `make` rather than trying to just do a `docker build`.** Actually assembling a Docker image for Ambassador involves quite a few steps before the image can be built.

Branching
---------

1. `master` is the branch from which Ambassador releases are cut. There should be no other activity on `master`.

2. `develop` is the default branch for the repo, and is the base from which all development should happen. 

3. **All development should be on branches cut from `develop`.** When you have things working and tested, submit a pull request back to `develop`.

This implies that `master` must _always_ be shippable and tagged, and `develop` _should_ always be shippable.

Unit Tests
----------

Unit tests for Ambassador live in the `ambassador/tests` directory. Within that directory, you'll find directories with names that start with three digits (`000-default`, `001-broader-v0`, etc); each of these directories describes a test environment. Tests run on every build of Ambassador, and test Ambassador's ability to transform a known Ambassador configuration into a known intermediate configuration and a known Envoy configuration.

**You are strongly encouraged to add tests when you add features.** Adding a new test is simply a matter of adding a new test directory under `ambassador/tests`. Start its name with three digits; the rest of the name should be a brief description of what you're testing, with no spaces.

Call the test directory (e.g. `ambassador/tests/002-tls-module-1`) `$TESTDIR`. Then:

- The test MUST contain an Ambassador configuration.
   - For every test except `000-default`, the configuration is the set of YAML files in `$TESTDIR/config`. 
      - The `000-default` test is special: it contains the magic `TEST_DEFAULT_CONFIG` marker, which says to use `ambassador/default-config` for configuration.
   - Some tests will also need to include other files, e.g. TLS certificates. These should be in `config` as well -- see the `002-tls-module-1` test for an example.

- The test SHOULD contain a known-good Ambassador intermediate configuration dump.
   - The intermediate configuration is generated by `ambassador dump`.
   - The known good intermediate configuration is in `$TESTDIR/gold.intermediate.json`.
   - If `gold.intermediate.json` is present, the test will generate `$TESTDIR/intermediate.json` and compare it with the known good version.

- The test SHOULD contain a known-good Envoy configuration dump.
   - The Envoy configuration is generated by `ambassador config`.
   - The known good Envoy configuration is in `$TESTDIR/gold.json`.
   - If `gold.json` is present:
      - The test will generate `$TESTDIR/envoy.json` and compare it with the known good version.
      - The test will also hand `$TESTDIR/envoy.json` to Envoy for validation -- this step runs inside Docker, and the test directory is mounted inside the Docker container as `/etc/ambassador-config`.

- In all cases, Ambassador's diagnostic service is invoked on the Ambassador configuration, and the results are checked for internal consistency.

To extend a test, change the configuration in `$TESTDIR/config`, then run a build (`make DOCKER_REGISTRY=-`) which will run all the CI tests. The tests will fail - if not, you didn't really change the test configuration, right? - but they'll output the diffs they found against the known-good versions. If all looks well, you can update the known-good versions:

- `mv $TESTDIR/intermediate.json $TESTDIR/gold.intermediate.json`
- `mv $TESTDIR/envoy.json $TESTDIR/gold.json`

and then commit all the changes you made in `$TESTDIR`.

End-to-End Tests
----------------

Ambassador's end-to-end tests are currently **not** run by CI, but they **are** part of the release criteria: we will not release an Ambassador for which the end-to-end tests are failing. **Again, you are strongly encouraged to add end-to-end test coverage for features you add.** 

For more information on the end-to-end tests, see [their README](end-to-end/README.md).

Version Numbering
-----------------

**Version numbers are determined by tags in Git, and will be computed by the build process.**

This means that if you build repeatedly without committing, you'll get the same version number. This isn't a problem as you debug and hack, although you may need to set `imagePullPolicy: Always` to have things work smoothly.

It also means that we humans don't say things like "I'm going to make this version 1.23.5" -- Ambassador uses [Semantic Versioning](http://www.semver.org/), and the build process computes the next version by looking at changes since the last tagged version. Start a Git commit comment with `[MINOR]` to tell the build that the change is worthy of a minor-version bump; start it with `[MAJOR]` for a major-version bump. If no marker is present, the patch version will be incremented.

Normal Workflow
---------------

0. `export DOCKER_REGISTRY=$registry`

   This sets the registry to which to push Docker images and is **mandatory**.

   "dwflynn" will push to Dockerhub with user `dwflynn`
   "gcr.io/flynn" will push to GCR with user `flynn`

   If you're using Minikube and don't want to push at all, set `DOCKER_REGISTRY` to "-".

   You can separately tweak the registry from which images will be _pulled_ using `AMBASSADOR_REGISTRY` and `STATSD_REGISTRY`. See the files in `templates` for more here.

1. Use a private branch cut from `develop` for your work.

   Committing to `master` triggers the CI pipeline to do an actual release. The base branch for development work is `develop` -- and you should always create a new branch from `develop` for your changes.

2. Hack away, then `make`. This will:

   a. Compute a version number based on git tags (`git describe --tags`).
   b. Push that version number everywhere in the code that it needs to be.
   c. Run tests and bail if something doesn't pass.
   d. Build Docker images, and push them if DOCKER_REGISTRY says to.
   e. Build YAML files for you in `doc/yaml`.

   **IT WILL NOT COMMIT OR TAG**. With new version numbers everywhere, you can easily `kubectl apply` the updated YAML files and see your changes in your Kubernetes cluster.

   If you make further changes and `make` again, _the version number will not change_. To get a new version number, you'll need to commit.

3. Commit to your feature branch.

   Remember: _not to `develop` or `master`_.

4. Open a pull request against `develop` when you're ready to ship.

What if I Don't Want to Push My Images?
---------------------------------------

**NOTE WELL**: if you're not using Minikube, this is almost certainly a mistake.

But if you are using Minikube, you can set `DOCKER_REGISTRY` to "-" to prevent pushing the images. The Makefile (deliberately) requires you to set DOCKER_REGISTRY, so you can't just unset it.

Building the Documentation and Website
--------------------------------------

Use `make website` to build the docs and website. See [the README](docs/README.md) for docs-specific steps. The `docs/build-website.sh` script (used by `make`) follows those steps and then performs some additional hacks for website use.
