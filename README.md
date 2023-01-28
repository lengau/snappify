# snappify
Tools to help automatically repackage apps as snaps.

Several of these tools may be useful for other repackaging, not just for snaps.
Where this is intended, it will be noted in the tool's documentation. If it
is not noted but you do use the tool for things other than snaps, please
open a pull request or issue about it. Otherwise, future updates may 
inadvertently remove this flexibility.

Unless declared otherwise, each tool in this repository is licensed under
the GNU GPL v3. Packages created using tools from this repository are not
subject to any licencing requirements.

# Reusable GitHub workflows

The initial purpose of this repository is to provide several [reusable GitHub
workfows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
to make it easier to automatically repackage apps as snaps.

## Sync Releases

The sync-gh-releases workflow series provides a way to repackage existing
GitHub releases. It creates and tags a series of commits in your repository
based on upstream commits.

A required input to this workflow is `prep_script`, which points to the location
within your repository of a script that makes the necessary changes to 
prepare the release. This script **must** update at least one existing tracked
file or create and `git add` a new file.

You can call this workflow from within your workflow as follows:

```yaml
permissions:
  contents: write  # This is required by sync-gh-releases to create commits.

jobs:
  sync-gh-releases:
    uses: lengau/snappify/.github/workflows/sync-gh-releases.yaml@main
    with:
      upstream_repository: octocat/Hello-World
      prep_script: release/prep_release.sh
      max_releases: 5
      git_name: "sync-gh-release[bot]"
      git_email: "sync-gh-release-bot@bots.noreply.github.com"
  release:
    runs-on: ubuntu-latest
    needs: sync-gh-releases
    strategy:
      max-parallel: 1  # Ensures the releases occur in the same order as upstream.
      matrix:
        tag: ${{ needs.sync-gh-releases.outputs.release_tags }}
    steps:
      - uses: actions/checkout@v3
        with:
          ref: ${{ matrix.tag }}
      - run: echo "Here's where I would release version ${{ matrix.tag }}"
```

The only required inputs are the `upstream_repository` and the `prep_script`.
The prep script will be called as follows:

```bash
prep_script {upstream_release_tag} {release_tag} {upstream_release_json_file}
```

For forwards compatibility, it should accept additional parameters. It must be 
able to run on the [latest Ubuntu runner 
image](https://github.com/actions/runner-images) with no additional software
installed. (If additional software is needed, it may install it. But this is
the responsibility of the prep script.)

In most cases, the upstream release tag and the release tag will be the same.
However, in rare cases they could differ, such as if a release on the current
repository gets removed.

The release JSON file is a JSON response from the GitHub API when querying
for that particular upstream release. It will be placed in the temporary
directory in the RUNNER_TEMP enviranment variable. Tools to help you extract the
necessary information in your prep script include [Python's `json` 
module](https://docs.python.org/3.10/library/json.html) and the [`jq` terminal
app](https://stedolan.github.io/jq/).
