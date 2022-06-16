# Contributors Guidelines
## First things to check

1. First you must add `mike` as a plugin in *mkdocs.yml* to ensure MkDocs recognises that you are using `mike` for versioning.
```yml
plugins:
  - mike
```

2. Added a *requirements.txt* file to the repository files which is used in the GitHub Actions workflow to install needed dependencies.

3. Enable GitHub Actions on repository

## Setting-up the CI system

### Introduction

The idea around the CI was implemented using Git tags and GitHub releases.

*Major Changes*
So anytime you make major improvements to the documentation and have committed those changes, you run the following commands:

``` bash

git tag v*.*.* # create a tag
git push origin v*.*.* # push changes to the tag not the branch

```
This will trigger the CI workflow named as `CI to update tags with changes` which we will later discuss.
 
After that, you will create a new release on GitHub with the latest tag you pushed.
This will also trigger the CI workflow named as `CI to set new docs release as latest version` which we will later discuss.

*Minor Changes*
When you make minor changes such as updating a document in an existing tag, you run the following commands:

```bash

git tag -f {existing_tagname}	# update existing tag
git push -f origin {existing_tagname} # push changes to the existing tag

```
This will trigger the CI workflow named as `CI to update tags with changes`.

*NOTE*: Releases are created only when you plan on rolling out a new version of documentation.

### Explaination of `CI to update tags with changes`

Below is an explaination of the code in *ci.yml* file:

```yml

name: CI to update tags with changes 
on:
  push:		# the workflow is triggered by push events
    tags:
      - 'v*.*.*' 	# but the workflow is triggered by push events which occur on tags that have the format `v*.*.*`, e.g. v1.2.0.

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-python@v2
        with:
          python-version: 3.9
      - name: Install docs dependencies
        run: pip install -r requirements.txt 	# Install the python dependencies needed from the requirements.txt file
      - name: Fetch GH-PAGES
        run: git fetch origin gh-pages --depth=1 	# We need to manually fetch the gh-pages branch so that we don't get issues since our CI doesn't have a local instance of the documentation branch to commit to
      - name: Setup Git User
        run: |
          git config --global user.name Docs deploy
          git config --global user.email docs@dummy.bot.com
      - name: Get the tag version
        id: get_version
        run: echo ::set-output name=VERSION::$(echo $GITHUB_REF | cut -d / -f 3)	# We get the tag name (e.g. v0.1.0) from the $GITHUB_REF environment variable which contains something like this: "refs/tags/v0.1.0"
      - name: Update the latest doc version
        run:  mike deploy ${{ steps.get_version.outputs.VERSION }} -b gh-pages --push	# Deploy the new version or updates of an existing version to the documentation branch (in this case: gh-pages). This will add the tag as a link in the version dropdown 


```

### Explaination of `CI to set new docs release as latest version`


Below is an explaination of the code in *release.yml* file:

```yml

name: CI to set new docs release as latest version
on:
  release:	# the workflow is triggered by release events (i.e. when a new release is creted)
    tags:
      - 'v*.*.*' # but the workflow is triggered by release events which occur on tags that have the format `v*.*.*`, e.g. v1.2.0.
    types: [published]	# also the workflow is triggered by release events which have the type published.

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-python@v2
        with:
          python-version: 3.9
      - name: Install docs dependencies
        run: pip install -r requirements.txt	# Install the python dependencies needed from the requirements.txt file
      - name: Fetch GH-PAGES
        run: git fetch origin gh-pages --depth=1	# We need to manually fetch the gh-pages branch so that we don't get issues since our CI doesn't have a local instance of the documentation branch to commit to
      - name: Setup Git User
        run: |
          git config --global user.name Docs deploy
          git config --global user.email docs@dummy.bot.com
      - name: Set new doc version as latest
        run:  mike alias -b gh-pages --update-aliases ${{ github.event.release.tag_name }} latest --push	# When a new release is created, we presume it to be the latest version so we update the `latest` alias to point to this new version
      - name: Change default latest version
        run:  mike set-default latest --push	# We set the default documentation version that people should be redirected to as `latest`

```

### Conclusion

The two workflow files might still need improvement but at this point they can handle versioning for your documentation very well.
