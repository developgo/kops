# Release process

The kOps project is released on an as-needed basis. The process is as follows:

## Release branches

We maintain a `release-1.17` branch for kOps 1.17.X, `release-1.18` for kOps 1.18.X,
etc.

`master` is where development happens.  We create new branches from master as we
prepare for a new minor release.  As we are
preparing for a new Kubernetes release, we will try to advance the master branch
to focus on the new functionality and cherry-pick back
to the release branches only as needed.

Generally we don't encourage users to run older kOps versions, or older
branches, because newer versions of kOps should remain compatible with older
versions of Kubernetes.

Beta and stable releases should be made from the `release-1.X` branch.
The tags should be made on the release branches. Alpha releases may be made on
either `master` or a release branch.

## Creating new release branches

Typically, kOps alpha releases are created off the master branch and beta and stable releases are created off of release branches.
In order to create a new release branch off of master prior to a beta release, perform the following steps:

1. Update the periodic E2E Prow jobs for the "next" kOps/Kubernetes minor version.
   * Edit [build_jobs.py](https://github.com/kubernetes/test-infra/tree/master/config/jobs/kubernetes/kops/build_jobs.py)
   to add the new minor version to `k8s_versions` and `kops_versions`.
     Also update the list of minor versions in `generate_versions()` and `generate_pipeline()`.
   * Edit the [testgrid config.yaml](https://github.com/kubernetes/test-infra/blob/master/config/testgrids/kubernetes/kops/config.yaml)
   to add the new minor version to both lists in the file, prefixed with `kops-k8s-`.
   * Remove the oldest minor version from each of those lists.
   * Run the `build_jobs.py` script.
2. Edit [kops-presubmits.yaml](https://github.com/kubernetes/test-infra/tree/master/config/jobs/kubernetes/kops/kops-presubmits.yaml)
   to create a new `pull-kops-e2e-kubernetes-aws-1-X` presubmit E2E prow job for the new release branch.
3. Create a new milestone in the GitHub repo.
4. Update [prow's milestone_applier config](https://github.com/kubernetes/test-infra/blob/dc99617c881805981b85189da232d29747f87004/config/prow/plugins.yaml#L309-L313)
   to update master to use the new milestone and add an entry for the new feature branch. 
   Create this as a separate PR as it will require separate review.
5. Create the new release branch in git and push it to the GitHub repo.
6. On the master branch, create a PR to update to the next minor version:
   * Update `OldestSupportedKubernetesVersion` and `OldestRecommendedKubernetesVersion` in
   [apply_cluster.go](https://github.com/kubernetes/kops/tree/master/upup/pkg/fi/cloudup/apply_cluster.go)
   * Add a row for the new minor version to [upgrade_k8s.md](https://github.com/kubernetes/kops/tree/master/permalinks/upgrade_k8s.md)
   * Fix any tests broken by the now-unsupported versions.
   * Create release notes for the next minor version. The release notes should mention the
   Kubernetes support removal and deprecation.
6. On master, off of the branch point, create the first alpha release for the new minor release.

## Creating releases

### Update versions

See [1.19.0-alpha.1 PR](https://github.com/kubernetes/kops/pull/9494) for an example.

* Use the hack/set-version script to update versions:  `hack/set-version 1.20.0`

The syntax is `hack/set-version <new-release-version>`

`new-release-version` is the version you are releasing.

* Update the golden tests: `hack/update-expected.sh`

* Commit the changes (without pushing yet): `git add -p && git commit -m "Release 1.X.Y"`

* This is the "release commit".

### Check builds are OK

```
rm -rf .build/ .bazelbuild/
make ci
```

### Send Pull Request to propose a release

```
git push $USER
hub pull-request
```

Wait for the PR to merge.

### Reviewing the release commit PR

To review someone else's release commit, verify that:

* A release at that point is desired. (For example, there are no unfixed blocking bugs.)
* There is nothing in the commit besides version number updates and golden outputs.

The "verify-versions" CI task will ensure that the versions have been updated in all the
expected places.

### Tag the release commit

(TODO: Can we automate this?  Maybe we should have a tags.yaml file)

Check out the release commit.
Make sure you are not on a newer one! Do not tag the merge commit!

```
VERSION=$(tools/get_version.sh | grep VERSION | awk '{print $2}')
echo ${VERSION}
```

```
git tag -a -m "Release ${VERSION}" v${VERSION}
git show v${VERSION}
```

Double check it is the release commit!

```
git push git@github.com:kubernetes/kops v${VERSION}
git fetch origin
```


### Wait for CI job to complete

The staging CI job should now see the tag, and build it (from the trusted prow cluster, using Google Cloud Build).

The job is here: https://testgrid.k8s.io/sig-cluster-lifecycle-kops#kops-postsubmit-push-to-staging

It (currently) takes about 10 minutes to run.

In the meantime, you can compile the release notes...

### Compile release notes

This step is not necessary for an ".0-alpha.1" release as these are made off
of the branch point for the previous minor release.

For example:

```
git checkout -b relnotes_${VERSION}

FROM=1.21.0-alpha.2
TO=1.21.0-alpha.3
DOC=1.21
git log v${FROM}..v${TO} --oneline | grep Merge.pull | grep -v Revert..Merge.pull | cut -f 5 -d ' ' | tac  > /tmp/prs
echo -e "\n## ${FROM} to ${TO}\n"  >> docs/releases/${DOC}-NOTES.md
relnotes  -config .shipbot.yaml  < /tmp/prs  >> docs/releases/${DOC}-NOTES.md
```

Review then send a PR with the release notes:

```
git add -p docs/releases/${DOC}-NOTES.md
git commit -m "Release notes for ${VERSION}"
git push ${USER}
hub pull-request
```

### Propose promotion of artifacts

Create container promotion PR:

```
cd ${GOPATH}/src/k8s.io/k8s.io

git checkout main
git pull
git checkout -b kops_images_${VERSION}

cd k8s.gcr.io/images/k8s-staging-kops
echo "" >> images.yaml
echo "# ${VERSION}" >> images.yaml
k8s-container-image-promoter --snapshot gcr.io/k8s-staging-kops --snapshot-tag ${VERSION} >> images.yaml
```

You can dry-run the promotion with 

```
cd ${GOPATH}/src/k8s.io/k8s.io
k8s-container-image-promoter --thin-manifest-dir k8s.gcr.io
```

Currently we send the image and non-image artifact promotion PRs separately.

```
git add -p k8s.gcr.io/images/k8s-staging-kops/images.yaml
git commit -m "Promote kOps $VERSION images"
git push ${USER}
hub pull-request -b main
```


Create binary promotion PR:

```
cd ${GOPATH}/src/k8s.io/k8s.io

git checkout main
git pull
git checkout -b kops_artifacts_${VERSION}

mkdir -p ./k8s-staging-kops/kops/releases/${VERSION}/
gsutil rsync -r  gs://k8s-staging-kops/kops/releases/${VERSION}/ ./k8s-staging-kops/kops/releases/${VERSION}/

promobot-generate-manifest --src k8s-staging-kops/kops/releases/ >> artifacts/manifests/k8s-staging-kops/${VERSION}.yaml
```

Verify, then send a PR:

```
git add artifacts/manifests/k8s-staging-kops/${VERSION}.yaml
git commit -m "Promote kOps $VERSION binary artifacts"
git push ${USER}
hub pull-request -b main
```


### Promote to GitHub / S3 (legacy) / artifacts.k8s.io


Binaries to github (all releases):

```
cd ${GOPATH}/src/k8s.io/kops/
shipbot -tag v${VERSION} -config .shipbot.yaml -src ${GOPATH}/src/k8s.io/k8s.io/k8s-staging-kops/kops/releases/${VERSION}/
```

Binaries to S3 bucket (only for releases < 1.22):

```
aws s3 sync --acl public-read k8s-staging-kops/kops/releases/${VERSION}/ s3://kubeupv2/kops/${VERSION}/
```

Until the binary promoter is automatic, we also need to promote the binary artifacts manually (only for stable releases):

```
mkdir -p /tmp/thin/artifacts/filestores/k8s-staging-kops/
mkdir -p /tmp/thin/artifacts/manifests/k8s-staging-kops/

cd ${GOPATH}/src/k8s.io/k8s.io
cp artifacts/manifests/k8s-staging-kops/${VERSION}.yaml /tmp/thin/artifacts/manifests/k8s-staging-kops/

cat > /tmp/thin/artifacts/filestores/k8s-staging-kops/filepromoter-manifest.yaml << EOF
filestores:
- base: gs://k8s-staging-kops/kops/releases/
  src: true
- base: gs://k8s-artifacts-prod/binaries/kops/
  service-account: k8s-infra-gcr-promoter@k8s-artifacts-prod.iam.gserviceaccount.com
EOF

promobot-files --filestores /tmp/thin/artifacts/filestores/k8s-staging-kops/filepromoter-manifest.yaml --files /tmp/thin/artifacts/manifests/k8s-staging-kops/ --dry-run=true
```

After validation of the dry-run output:
```
promobot-files --filestores /tmp/thin/artifacts/filestores/k8s-staging-kops/filepromoter-manifest.yaml --files /tmp/thin/artifacts/manifests/k8s-staging-kops/ --dry-run=false --use-service-account
```

### Smoke test the release

```
wget https://artifacts.k8s.io/binaries/kops/${VERSION}/linux/amd64/kops

mv kops ko
chmod +x ko

./kops version
```

Also run through a `kops create cluster` flow, ideally verifying that
everything is pulling from the new locations.

### Publish to GitHub

* Download release
* Validate it
* Add notes
* Publish it

### Release to Homebrew

This step is only necessary for stable releases in the latest stable minor version.

* Following the [documentation](homebrew.md) we must release a compatible homebrew formulae with the release.
* This should be done at the same time as the release, and we will iterate on how to improve timing of this.

### Update conformance results with CNCF

This step is only necessary for a first stable minor release (a ".0").

Use the following instructions: https://github.com/cncf/k8s-conformance/blob/master/instructions.md

### Update latest minor release in documentation

This step is only necessary for a first stable minor release (a ".0").

Create a PR that updates the following documents:

* Rotate the new version into the version matrix in both
[releases.md](https://github.com/kubernetes/kops/tree/master/docs/welcome/releases.md)
and [README-ES.md](https://github.com/kubernetes/kops/tree/master/README-ES.md).
* Remove the "has not been released yet" header in the version's release notes.

### Add link to release notes

This step is only necessary for a first beta minor release (a ".0-beta.1").

Create a PR that updates the following document:

* Add a reference to the version's release notes in [mkdocs.yml](https://github.com/kubernetes/kops/tree/master/mkdocs.yml)

### Update the alpha channel and/or stable channel

Once we are satisfied the release is sound:

* Bump the kOps recommended version in the alpha channel

Once we are satisfied the release is stable:

* Bump the kOps recommended version in the stable channel
