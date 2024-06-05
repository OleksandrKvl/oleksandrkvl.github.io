---
layout: post
title:  "Multi-version Doxygen documentation with GitHub Pages"
date:   2024-06-05 11:05:00 +0300
post_url: 2024-06-05-multi-version-doxygen
---

## Table of contents

* [Introduction](#introduction)
* [Problems with mono-version documentation](#mono-docs-problems)
* [Welcome multi-version documentation](#multi-version-docs)
* [Prerequisites](#prerequisites)
* [Overall design](#overall-design)
* [Version switch mechanics](#mechanics)
  * [Adjusting folder names](#folder-names)
  * [Main page](#main-page)
  * [Version selector](#version-selector)
* [Automation](#automation)
  * [Actions](#actions)
    * [Building documentation](#build-docs-action)
    * [Updating `version_selector_html`](#version-selector-action)
    * [Updating redirect page](#redirect-page-action)
  * [Workflows](#workflows)
    * [Generating `git-main` docs](#git-main-docs-workflow)
    * [Generating release docs](#release-docs-workflow)
    * [Generating PR docs](#pr-docs-create-workflow)
    * [Removing PR docs](#pr-docs-remove-workflow)
* [Upgrading from a mono to multi-version documentation](#mono-to-multi-docs)
* [Wrap-up](#wrap-up)

---

## Introduction {#introduction}

In this article I'll show how to host a multi-version Doxygen documentation on
GitHub Pages and automate its generation using GitHub Actions.

---

## Problems with mono-version documentation {#mono-docs-problems}

I have a small [hobby project](https://github.com/OleksandrKvl/sbepp) that
generates documentation using Doxygen and hosts it on GitHub Pages. Nothing
special but as the project begins to evolve, I realized that having a single
documentation is not nice for multiple reasons:
- it requires special notes like "available/deprecated since version N" around
API.
- it requires similar notes for non-API things, like describing overall design,
recommended practices, examples.
- the above notes are not generally useful because changes are usually driven by
users who are supposed to migrate to the latest version.

Basically, over the time, documentation gets polluted with those notes to
cover all the existing versions. Of course, another approach is just to remove
older documentation altogether but it's not friendly to users that for some
reasons can't migrate to the latest version immediately.

---

## Welcome multi-version documentation {#multi-version-docs}

All of the above problems can be solved when a project has dedicated
documentation for each release. Pretty every large project like
[Boost](https://www.boost.org/doc/libs/1_85_0/) or
[Python](https://docs.python.org/3.10/) uses this approach. Unfortunately,
Doxygen doesn't support this out of the box. It generates a completely
standalone set of HTML pages for a given version but has no functionality to
combine multiple such sets and allow the user to switch between them.

I've found no existing tutorial to achieve this but ChatGPT suggested an
approach and guided me through most of its steps. Note that I'm not a front-end
nor a DevOps expert so there's a chance that there's a space for further
improvement but, overall, it should be a solid start for anyone with a similar
goal.  

---

## Prerequisites {#prerequisites}

This tutorial uses CMake 3.29.3 and Doxygen 1.10. It also assumes
that project version is specified using CMake `project()` command and available
via `${PROJECT_VERSION}` variable.

The very basic CMake + Doxygen setup looks like this:

```cmake
find_package(Doxygen REQUIRED)

# Doxygen options in the form of `DOXYGEN_<OPTION_NAME>`...

doxygen_add_docs(
    doc                      # target name
    "${PROJECT_SOURCE_DIR}"  # sources to scan for docs
)
```

All the following changes are made on top of it. It's not a problem if you're
using another approach, just apply similar settings in a way you prefer.

The final demo repository is
[here](https://github.com/OleksandrKvl/multi_version_doxygen_test), it contains
all the parts described below. Its docs are
[here](https://oleksandrkvl.github.io/multi_version_doxygen_test/).

---

## Overall design {#overall-design}

Here's the directory structure we need:

```bash
/           # docs root
    1.0.0/  # version-specific docs
    2.0.0/
```

These dirs will be located in a separate branch (e.g. `gh-pages`) that contains
nothing but the docs themselves. On each release (or another event), a new
documentation will be generated using Doxygen and pushed to the root of that
branch into a version-specific directory. As was said above,
generated docs know nothing about each other so our two main problems are:
- version switch mechanics
- automated docs generation and population of the docs branch

---

## Version switch mechanics {#mechanics}

### Adjusting folder names {#folder-names}

By default, Doxygen generates docs into `html` directory, to have them in a
version-specific directory we can use `HTML_OUTPUT` option:

```cmake
set(DOXYGEN_HTML_OUTPUT "${PROJECT_VERSION}")
```

It will produce the docs into a path like `build_dir/doc/1.0.0`, where `doc` is
the Doxygen target name. But there's a minor problem here. When we build the
`doc` target, we don't know the actual version and hence the path to generated
docs. Without this knowledge it's pretty hard to automate the process. To solve
it, let's add another directory into that path using Doxygen
`OUTPUT_DIRECTORY` setting:

```cmake
set(DOXYGEN_OUTPUT_DIRECTORY "docs")
```

Now, docs are located in `build_dir/doc/docs/1.0.0` and the new `docs`
directory holds nothing else but our version-specific docs. To determine the
generated version, we can simply enumerate directories in `docs`, actually,
there will always be a single one. This trick will be used later in the
[building documentation](#build-docs-action) section.

---

### Main page {#main-page}

OK, now we can have docs in separate version-specific directories but what
should be the main page? When docs are hosted on a standalone server over which
you have full control, you can try to update the path to index page on each new
release. GitHub Pages is not so flexible, it always looks for `index.html` in
the root directory. To make it work, we have to implement HTML redirect:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="refresh" content="0; url=2.0.0/index.html">
    <title>Redirecting...</title>
</head>
<body>
    <p>If you are not redirected automatically, <a href="2.0.0/index.html">click here</a>.</p>
</body>
</html>
```

The target URL changes with each release so the generation of this
file will be automated later. Here's how the root directory will look like at
this stage:

```bash
/           # root of our documentation
    1.0.0/  # version-specific directories
        index.html # version-specific main page generated by Doxygen
        ...
    2.0.0/
        index.html
        ...

    index.html # redirects to the latest release docs, e.g. `2.0.0/index.html`
```

---

### Version selector {#version-selector}

Now comes the interesting part, we need that dropdown selector element that
does the actual switch between versions. Doxygen has `PROJECT_NUMBER` option,
when set, it nicely displays the project version next to the project name.
`doxygen_add_docs` automatically sets it to `${PROJECT_VERSION}`. If you're
using standalone Doxygen config, be sure to set it by hand.
That static number has to be replaced with a version selector. Doxygen allows
customization of the header HTML part that's common for all pages. First, we
need to generate the default one:

```bash
doxygen -w html header.html footerFile styleSheetFile Doxyfile.doc
```

Here, `Doxyfile.doc` is Doxygen configuration file generated by
`doxygen_add_docs`, it can be found in the binary directory of the `doc` target
(e.g.
`build_dir/doc/Doxyfile.doc`). If you're using standalone configuration
file, use it here instead of `Doxyfile.doc`.
From 3 files generated by this command, we need only `header.html` that we are
about to customize. Opening it, we can see where that version text is located:

```html
<span id="projectnumber">&#160;$projectnumber</span>
```

During docs generation, Doxygen replaces that `$projecnumber` with
`PROJECT_NUMBER` value but we don't need that. Instead, we cut it down to just:

```html
<span id="projectnumber"/>
```

Then, we add a `version_selector_handler.js` script that does 3 things:
- injects `<select>` element from `version_selector.html` inside the
  `projectnumber` span
- implements redirect when different version is selected
- maintains proper active value of the `<select>` element

Here's the script:

```js
// version_selector_handler.js
$(function () {
    var repoName = window.location.pathname.split('/')[1];
    $.get('/' + repoName + '/version_selector.html', function (data) {
        // Inject version selector HTML into the page
        $('#projectnumber').html(data);

        // Event listener to handle version selection
        document.getElementById('versionSelector').addEventListener('change', function () {
            var selectedVersion = this.value;
            window.location.href = '/' + repoName + '/' + selectedVersion + '/index.html';
        });

        // Set the selected option based on the current version
        var currentVersion = window.location.pathname.split('/')[2];
        $('#versionSelector').val(currentVersion);
    });
});
```

Note that its URL is in the form of `/<repo_name>/version_selector.html`.
That's because when project is published on GitHub Pages, all its URLs will have
`<repo_name>` prefix. It's not needed if docs are hosted on a fully standalone
domain.

This script has to be injected at the bottom of the `<head>` section of the
`header.html`:

```html
<!-- header.html -->
<head>
  <!-- other content... -->
  <script type="text/javascript" src="$relpath^version_selector_handler.js"></script>
</head>
```

The actual `<select>` element is located in `/version_selector.html` file and
will be populated automatically. Here's how it can look like:

```html
<!-- version_selector.html -->
<select id="versionSelector">
    <option value="2.0.0">2.0.0</option>
    <option value="1.0.0">1.0.0</option>
</select>
```

Here's the full structure of our documentation directory:

```bash
/           # root of our documentation
    1.0.0/  # version-specific directories
        version_selector_handler.js # loads `/version_selector.html`
        index.html # version-specific main page generated by Doxygen
        ...
    2.0.0/
        version_selector_handler.js
        index.html
        ...

    index.html # redirects to the latest release, e.g. `2.0.0/index.html`
    version_selector.html # holds the actual version list
```

Note that while every generated documentation has its own
`version_selector_handler.js` copy, there's only one instance of
`version_selector.html`. There's no need to regenerate existing docs to
introduce a new version, only `version_selector.html` has to be updated.

We need to adjust Doxygen settings to bring the above changes together:

```cmake
# specify custom header path
set(DOXYGEN_HTML_HEADER "header.html")

# extra files that will be copied alongside generated HTML files
set(DOXYGEN_HTML_EXTRA_FILES "version_selector_handler.js")
```

And that's it, now each HTML page has that `<select>` element with version list
which is updated automatically and user can switch between versions as they
want.

---

## Automation {#automation}

The next major step is automation of the above steps using GitHub Actions. The
following assumes you know at least basics of how Actions and Workflows work.

### Actions {#actions}

We need 3 basic building blocks to achieve our goals:
- build the docs
- update `version_selector.html` that contains version list
- update `/index.html` HTML redirect

Using them we can implement different strategies using GitHub Workflows.
I've extracted them to corresponding custom actions. Here, I will show only
their core parts to keep things short, you can find full sources in the [demo
repository](https://github.com/OleksandrKvl/multi_version_doxygen_test/tree/main/.github/actions). Note that the actions below are made configurable and can be
reused as is, check out their `inputs` section for a list of parameters.

---

#### Building documentation {#build-docs-action}

```yaml
# .github/actions/build-docs/action.yaml

steps:
  - name: Install deps
    shell: bash
    run:   |
            sudo apt install -y cmake
            sudo apt install -y wget
            wget -nv https://www.doxygen.nl/files/doxygen-1.10.0.linux.bin.tar.gz
            tar -xzf doxygen-1.10.0.linux.bin.tar.gz
            echo "$(pwd)/doxygen-1.10.0/bin" >> $GITHUB_PATH

  - name: CMake configuration
    shell: bash
    run:  cmake ${{ inputs.cmake_configure_args }} -B ${{ inputs.build_dir }}

  - name: CMake build
    shell: bash
    run:  cmake --build ${{ inputs.build_dir }} --target ${{ inputs.cmake_target }}

  - name: Get docs version
    id: get-docs-version
    shell: bash
    run: |
      subdir=$(basename $(find ${{ inputs.build_dir }}/${{ inputs.docs_dir }} -mindepth 1 -maxdepth 1 -type d | head -n 1))
      echo "version=$subdir" >> $GITHUB_OUTPUT

  # deploy to either version-specific or to explicitly provided directory
```

It just installs CMake and Doxygen and builds the target that is responsible for
invoking Doxygen. The only interesting part here is the last step, it retrieves
the version of the generated documentation by simply enumerating directories in
the Doxygen `OUTPUT_DIRECTORY`.

---

#### Updating `version_selector.html` {#version-selector-action}

`version_selector.html` should contain all the versions we have published on our
docs branch (e.g. `gh-pages`). This is achieved by enumerating
directories and then generating HTML file from that list:

```yaml
# .github/actions/update-version-selector/action.yaml

steps:
  - name: Discover versions
    id: discover-versions
    shell: bash
    run: |
      git fetch origin ${{ inputs.docs_branch }}
      dirs=$(git ls-tree --name-only -d origin/${{ inputs.docs_branch }} | sort -rV)
      echo "counter=$(echo "$dirs" | wc -l | xargs)" >> $GITHUB_OUTPUT

      mkdir ${{ inputs.temp_dir }}
      # Create HTML
      echo '<select id="${{ inputs.selector_id }}">' > ${{ inputs.temp_dir }}/${{ inputs.file_name }}
      for dir in $dirs; do
          if [[ "$(basename "$dir")" != .* ]]; then
              version=$(basename "$dir")
              echo "    <option value=\"$version\">$version</option>" >> ${{ inputs.temp_dir }}/${{ inputs.file_name }}
          fi
      done
      echo '</select>' >> ${{ inputs.temp_dir }}/${{ inputs.file_name }}

  # deploy step...
```

---

#### Updating redirect page {#redirect-page-action}

This one is trivial, just take the target url from the parameter and generate
the standard HTML redirect:

```yaml
# .github/actions/update-redirect-page/action.yaml

steps:
  - name: Generate redirect HTML
    shell: bash
    run: |
      mkdir ${{ inputs.temp_dir }}
      cat << EOF > ${{ inputs.temp_dir }}/${{ inputs.file_name }}
      <!DOCTYPE html>
      <html lang="en">
      <head>
          <meta charset="UTF-8">
          <meta http-equiv="refresh" content="0; url=${{ inputs.target_url }}">
          <title>Redirecting...</title>
      </head>
      <body>
          <p>If you are not redirected automatically, <a href="${{ inputs.target_url }}">click here</a>.</p>
      </body>
      </html>
      EOF

  # deploy step...
```

---

### Workflows {#workflows}

The above three actions are enough to implement almost any strategy for docs
generation. Let's see a couple of common ones.

---

#### Generating `git-main` docs {#git-main-docs-workflow}

It's useful to have not only release-specific docs but the ones corresponding
to the latest, not yet released version of a project. This can be achieved by
generating docs from the `main` branch whenever new commits are pushed into it.
We'll give such docs a `git-main` "version":

```yaml
# .github/workflows/create-git-main-docs.yml

on:
  push:
    branches:
      - main

jobs:
  create-git-main-docs:
    runs-on: ubuntu-22.04

    steps:
      - uses: actions/checkout@v4

      - name: Build docs
        id: build-docs
        uses: ./.github/actions/build-docs
        with:
          cmake_target: 'doc'
          docs_dir: 'doc/docs'
          destination_dir: git-main
          github_token: ${{ secrets.GITHUB_TOKEN }}

      - name: Update version selector
        id: update-version-selector
        uses: ./.github/actions/update-version-selector
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}

      - name: Create redirect page if there are no releases
        if: ${{ steps.update-version-selector.outputs.versions_counter == 1}}
        uses: ./.github/actions/update-redirect-page
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          target_url: git-main/index.html
```

The first two steps are self-explanatory, the only thing to notice is
`destination_dir: git-main` argument to `build-docs` action. It forces
`build-docs` to deploy documentation to the `git-main` directory, not to a
version-specific one because those are reserved for releases. The last step is
required to create redirect page when your documentation branch has no
release-specific docs yet. It's useful when you begin to play with this stuff to
test how everything works without making new releases.

---

#### Generating release docs {#release-docs-workflow}

Generating release-specific docs is the main goal of this tutorial and its
workflow is even simpler than the above. This time `destination_dir` is not set
so the docs are published into a version-specific directory, e.g. `1.0.0`:

```yaml
on:
  release:
    types: [released]

jobs:
  create-release-docs:
    runs-on: ubuntu-22.04

    steps:
      - uses: actions/checkout@v4

      - name: Build docs
        id: build-docs
        uses: ./.github/actions/build-docs
        with:
          cmake_target: 'doc'
          docs_dir: 'doc/docs'
          github_token: ${{ secrets.GITHUB_TOKEN }}

      - name: Update redirect HTML
        uses: ./.github/actions/update-redirect-page
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          target_url: ${{ steps.build-docs.outputs.version }}/index.html

      - name: Update version selector
        uses: ./.github/actions/update-version-selector
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

---

#### Generating PR docs {#pr-docs-create-workflow}

The above two workflows are a good basis to generate docs for a project. As an
example of something "extra", let's generate docs from a pull request. It can be
useful for projects with many contributors to check that docs for a new feature
are correct. Generating them from every PR makes no sense so we need some
condition here. GitHub has different ways to control when to run such a
workflow, I've chosen the simplest one, to run it when PR is labeled as
`documentation`. It's very similar to `git-main` workflow but now
`destination_dir` is set to
`PR-${{ github.event.pull_request.number }}` and redirect page is never touched:

```yaml
# .github/workflows/create-pr-docs.yml

on:
  pull_request:
    types: [labeled, synchronize]
    branches:
      - main

jobs:
  create-pr-docs:
    if: ${{ contains(github.event.pull_request.labels.*.name, 'documentation') }}
    runs-on: ubuntu-22.04

    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}

      - name: Build docs
        id: build-docs
        uses: ./.github/actions/build-docs
        with:
          cmake_target: 'doc'
          docs_dir: 'doc/docs'
          destination_dir: PR-${{ github.event.pull_request.number }}
          github_token: ${{ secrets.GITHUB_TOKEN }}

      - name: Update version selector
        uses: ./.github/actions/update-version-selector
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

---

#### Removing PR docs {#pr-docs-remove-workflow}

Unlike `git-main` and release docs which stay there forever, PR docs should be
removed when PR is closed:

```yaml
# .github/workflows/remove-pr-docs.yml

on:
  pull_request:
    types: [closed]
    branches:
      - main

jobs:
  remove-pr-docs:
    if: ${{ contains(github.event.pull_request.labels.*.name, 'documentation') }}
    runs-on: ubuntu-22.04

    steps:
      - name: Remove PR docs
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ${{ github.workspace }}
          destination_dir: PR-${{ github.event.pull_request.number }}

      - uses: actions/checkout@v4
        with:
          sparse-checkout: .github

      - name: Update version selector
        uses: ./.github/actions/update-version-selector
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

Here, `peaceiris/actions-gh-pages` removes `destination_dir` before pushing new
files to it and since `${{ github.workspace }}` is empty, this effectively
results in removing PR docs.

---

## Upgrading from a mono to multi-version documentation {#mono-to-multi-docs}

Switch from a mono to multi-version documentation requires a bit of manual
intervention. `git-main` docs will be generated automatically once new
functionality is merged into `main`, the same applies to new releases but not
to the previous ones. For my project, I've decided to integrate only the latest
available release at the time to the new multi-version docs branch. But that
release generates docs without the new version selector functionality and
wouldn't work as is. Here's how I've done it:
- pulled release tag locally
- applied Doxygen-specific changes on top of it
- generated its documentation, now it has version selector functionality
- manually pushed it to my `gh-pages` branch into the corresponding
version-specific directory
- manually added that version to `version_selector.html`
- updated `index.html` redirect page

That's it, now old release docs are fully integrated. Not a lot of work for a
single release but if one needs this for more releases, it makes sense to
automate the process.

---

## Wrap-up {#wrap-up}

We're done, at this point we have a fully automated system that generates and
publishes documentation for a project allowing user to switch between the
versions. The presented approach is not the only one, many other customizations
are possible but it should be a good place to start.