# Upload-Artifact

This uploads artifacts from your workflow allowing you to share data between jobs and store data once a workflow is complete.
This action is an extension of Github's [upload-artifact](https://github.com/actions/upload-artifact) action that `tar` the files before uploading.

See also [download-artifact](https://github.com/ATOS-Actions/download-artifact).

[![Release](https://github.com/ATOS-Actions/upload-artifact/actions/workflows/on_push.yml/badge.svg#main)](https://github.com/ATOS-Actions/upload-artifact/actions/workflows/on_push.yml)

# Usage

See [action.yml](action.yml)

### Upload an Individual File

```yaml
steps:
  - uses: actions/checkout@v2

  - run: mkdir -p path/to/artifact

  - run: echo hello > path/to/artifact/world.txt

  - uses: atos-actions/upload-artifact@v1
    with:
      name: my-artifact
      path: path/to/artifact/world.txt
```

### Upload an Entire Directory

```yaml
- uses: atos-actions/upload-artifact@v1
  with:
    name: my-artifact
    path: path/to/artifact/ # or path/to/artifact
```

### Upload using a Wildcard Pattern

```yaml
- uses: atos-actions/upload-artifact@v1
  with:
    name: my-artifact
    path: path/**/[abc]rtifac?/*
```

### Upload using Multiple Paths and Exclusions

```yaml
- uses: atos-actions/upload-artifact@v1
  with:
    name: my-artifact
    path: |
      path/output/bin/
      path/output/test-results
      !path/**/*.tmp
```

For supported wildcards along with behavior and documentation, see [@actions/glob](https://github.com/actions/toolkit/tree/main/packages/glob) which is used internally to search for files.

If a wildcard pattern is used, the path hierarchy will be preserved after the first wildcard pattern:

```
path/to/*/directory/foo?.txt =>
    ∟ path/to/some/directory/foo1.txt
    ∟ path/to/some/directory/foo2.txt
    ∟ path/to/other/directory/foo1.txt

would be flattened and uploaded as =>
    ∟ some/directory/foo1.txt
    ∟ some/directory/foo2.txt
    ∟ other/directory/foo1.txt
```

If multiple paths are provided as input, the least common ancestor of all the search paths will be used as the root directory of the artifact. Exclude paths do not affect the directory structure.

Relative and absolute file paths are both allowed. Relative paths are rooted against the current working directory. Paths that begin with a wildcard character should be quoted to avoid being interpreted as YAML aliases.

The [@actions/artifact](https://github.com/actions/toolkit/tree/main/packages/artifact) package is used internally to handle most of the logic around uploading an artifact. There is extra documentation around upload limitations and behavior in the toolkit repo that is worth checking out.

### Customization if no files are found

If a path (or paths), result in no files being found for the artifact, the action will succeed but print out a warning. In certain scenarios it may be desirable to fail the action or suppress the warning. The `if-no-files-found` option allows you to customize the behavior of the action if no files are found:

```yaml
- uses: atos-actions/upload-artifact@v1
  with:
    name: my-artifact
    path: path/to/artifact/
    if-no-files-found: error # 'warn' or 'ignore' are also available, defaults to `warn`
```

### Conditional Artifact Upload

To upload artifacts only when the previous step of a job failed, use [`if: failure()`](https://help.github.com/en/articles/contexts-and-expression-syntax-for-github-actions#job-status-check-functions):

```yaml
- uses: atos-actions/upload-artifact@v1
  if: failure()
  with:
    name: my-artifact
    path: path/to/artifact/
```

### Uploading without an artifact name

You can upload an artifact without specifying a name

```yaml
- uses: atos-actions/upload-artifact@v1
  with:
    path: path/to/artifact/world.txt
```

If not provided, `artifact` will be used as the default name which will manifest itself in the UI after upload.

### Uploading to the same artifact

With the following example, the available artifact (named `artifact` by default if no name is provided) would contain both `world.txt` (`hello`) and `extra-file.txt` (`howdy`):

```yaml
- run: echo hi > world.txt
- uses: atos-actions/upload-artifact@v1
  with:
    path: world.txt

- run: echo howdy > extra-file.txt
- uses: atos-actions/upload-artifact@v1
  with:
    path: extra-file.txt

- run: echo hello > world.txt
- uses: atos-actions/upload-artifact@v1
  with:
    path: world.txt
```

Each artifact behaves as a file share. Uploading to the same artifact multiple times in the same workflow can overwrite and append already uploaded files:

```yaml
strategy:
  matrix:
    node-version: [8.x, 10.x, 12.x, 13.x]
steps:
  - name: Create a file
    run: echo ${{ matrix.node-version }} > my_file.txt
  - name: Accidentally upload to the same artifact via multiple jobs
    uses: atos-actions/upload-artifact@v1
    with:
      name: my-artifact
      path: ${{ github.workspace }}
```

> **_Warning:_** Be careful when uploading to the same artifact via multiple jobs as artifacts may become corrupted. When uploading a file with an identical name and path in multiple jobs, uploads may fail with 503 errors due to conflicting uploads happening at the same time. Ensure uploads to identical locations to not interfere with each other.

In the above example, four jobs will upload four different files to the same artifact but there will only be one file available when `my-artifact` is downloaded. Each job overwrites what was previously uploaded. To ensure that jobs don't overwrite existing artifacts, use a different name per job:

```yaml
uses: atos-actions/upload-artifact@v1
with:
  name: my-artifact ${{ matrix.node-version }}
  path: ${{ github.workspace }}
```

### Environment Variables and Tilde Expansion

You can use `~` in the path input as a substitute for `$HOME`. Basic tilde expansion is supported:

```yaml
- run: |
    mkdir -p ~/new/artifact
    echo hello > ~/new/artifact/world.txt
- uses: atos-actions/upload-artifact@v1
  with:
    name: Artifacts-V3
    path: ~/new/**/*
```

Environment variables along with context expressions can also be used for input. For documentation see [context and expression syntax](https://help.github.com/en/actions/reference/context-and-expression-syntax-for-github-actions):

```yaml
env:
  name: my-artifact
steps:
  - run: |
      mkdir -p ${{ github.workspace }}/artifact
      echo hello > ${{ github.workspace }}/artifact/world.txt
  - uses: atos-actions/upload-artifact@v1
    with:
      name: ${{ env.name }}-name
      path: ${{ github.workspace }}/artifact/**/*
```

For environment variables created in other steps, make sure to use the `env` expression syntax

```yaml
steps:
  - run: |
      mkdir testing
      echo "This is a file to upload" > testing/file.txt
      echo "artifactPath=testing/file.txt" >> $GITHUB_ENV
  - uses: atos-actions/upload-artifact@v1
    with:
      name: artifact
      path: ${{ env.artifactPath }} # this will resolve to testing/file.txt at runtime
```

### Retention Period

Artifacts are retained for 90 days by default. You can specify a shorter retention period using the `retention-days` input:

```yaml
- name: Create a file
  run: echo "I won't live long" > my_file.txt

- name: Upload Artifact
  uses: atos-actions/upload-artifact@v1
  with:
    name: my-artifact
    path: my_file.txt
    retention-days: 5
```

The retention period must be between 1 and 90 inclusive. For more information see [artifact and log retention policies](https://docs.github.com/en/free-pro-team@latest/actions/reference/usage-limits-billing-and-administration#artifact-and-log-retention-policy).

## Additional Documentation

See [Github's upload-artifact action](https://github.com/actions/upload-artifact) for additional documentation.
See [Storing workflow data as artifacts](https://docs.github.com/en/actions/advanced-guides/storing-workflow-data-as-artifacts) for additional examples and tips.

See extra documentation for the [@actions/artifact](https://github.com/actions/toolkit/blob/main/packages/artifact/docs/additional-information.md) package that is used internally regarding certain behaviors and limitations.
