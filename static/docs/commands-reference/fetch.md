# fetch

Get files that are under DVC control from
[remote](/doc/commands-reference/remote#description) storage into the local
cache.

## Synopsis

```usage
usage: dvc fetch [-h] [-q | -v] [-j JOBS] [--show-checksums]
                 [-r REMOTE] [-a] [-T] [-d] [-R]
                 [targets [targets ...]]

positional arguments:
  targets        Limit command scope to these DVC-files. Using -R,
                 directories to search DVC-files in can also be given.
```

## Description

The `dvc fetch` command is a means to download files from remote storage into
the local cache, but without placing them in the <abbr>workspace</abbr>. This
makes the data files available for linking (or copying) into the workspace.
(Refer to [dvc config cache.type](/doc/commands-reference/config#cache).) Along
with `dvc checkout`, it's performed automatically by `dvc pull` when the target
[DVC-files](/doc/user-guide/dvc-file-format) are not already in the local cache:

```
Controlled files             Commands
---------------- ---------------------------------

remote storage
     +
     |         +------------+
     | - - - - | dvc fetch  | ++
     v         +------------+   +   +----------+
local cache                      ++ | dvc pull |
     +         +------------+   +   +----------+
     | - - - - |dvc checkout| ++
     |         +------------+
     v
 workspace
```

Fetching could be useful when first checking out an existing <abbr>DVC
project</abbr>, since files under DVC control could already exist in remote
storage, but won't be in your local cache. (Refer to `dvc remote` for more
information on DVC remotes.) These necessary data or model files are listed as
dependencies or outputs in a DVC-file (target
[stage](/doc/commands-reference/run)) so they are required to
[reproduce](/doc/get-started/reproduce) the corresponding
[pipeline](/doc/commands-reference/pipeline). (See
[DVC-File Format](/doc/user-guide/dvc-file-format) for more information on
dependencies and outputs.)

`dvc fetch` ensures that the files needed for a DVC-file to be
[reproduced](/doc/get-started/reproduce) exist in the local cache. If no
`targets` are specified, the set of data files to fetch is determined by
analyzing all DVC-files in the current branch, unless `--all-branches` or
`--all-tags` is specified.

The default remote is used unless `--remote` is specified. See `dvc remote add`
for more information on how to configure different remote storage providers.

`dvc fetch`, `dvc pull`, and `dvc push` are related in that these 3 commands
perform data synchronization among local and remote storage. The specific way in
which the set of files to push/fetch/pull is determined begins with calculating
the checksums of the files in question, when these are
[added](/doc/get-started/add-files) to DVC. File checksums are then stored in
the corresponding DVC-files (usually saved in a Git branch). Only the checksums
specified in DVC-files currently in the project are considered by `dvc fetch`
(unless the `-a` or `-T` options are used).

## Options

- `-r REMOTE`, `--remote REMOTE` - name of the
  [remote storage](/doc/commands-reference/remote#description) to fetch from
  (see `dvc remote list`). If not specified, the default remote is used (see
  `dvc config core.remote`). The argument `REMOTE` is a remote name defined
  using the `dvc remote` command.

- `-d`, `--with-deps` - determine files to download by tracking dependencies to
  the target DVC-files (stages). This option only has effect when one or more
  `targets` are specified. By traversing all stage dependencies, DVC searches
  backward from the target stages in the corresponding pipelines. This means DVC
  will not fetch files referenced in later stages than the `targets`.

- `-R`, `--recursive` - `targets` is expected to contain at least one directory
  path for this option to have effect. Determines the files to fetch by
  searching each target directory and its subdirectories for DVC-files to
  inspect.

- `-j JOBS`, `--jobs JOBS` - number of threads to run simultaneously to handle
  the downloading of files from the remote. Using more jobs may improve the
  total download speed if a combination of small and large files are being
  fetched. The default value is `4 * cpu_count()`. For SSH remotes default is
  just 4.

- `-a`, `--all-branches` - fetch cache for all branches, not just the active
  one. This means DVC may download files needed to reproduce different versions
  of a DVC-file ([experiments](/doc/get-started/experiments)), not just the
  current one.

- `-T`, `--all-tags` - fetch cache for all tags. Similar to `-a` above.

- `--show-checksums` - show checksums instead of file names when printing the
  download progress.

* `-h`, `--help` - prints the usage/help message, and exit.

* `-q`, `--quiet` - do not write anything to standard output. Exit with 0 if no
  problems arise, otherwise 1.

* `-v`, `--verbose` - displays detailed tracing information.

## Examples

Let's employ a simple <abbr>workspace</abbr> with some data, code, ML models,
pipeline stages, as well as a few Git tags, such as the <abbr>DVC project</abbr>
created in our
[get started example repo](https://github.com/iterative/example-get-started).
Then we can see what happens with `dvc fetch` as we switch from tag to tag.

<details>

### Click and expand to setup the project

Start by cloning our sample repo if you don't already have it:

```dvc
$ git clone https://github.com/iterative/example-get-started
$ cd example-get-started
```

</details>

The workspace looks almost like in this
[pipeline setup](/doc/get-started/example-pipeline):

```dvc
.
├── data
│   └── data.xml.dvc
├── evaluate.dvc
├── featurize.dvc
├── prepare.dvc
├── train.dvc
└── src
    └── <code files here>
```

We have these tags in the repository that represent different iterations of
solving the problem:

```dvc
$ git tag

baseline-experiment     <- first simple version of the model
bigrams-experiment       <- use bigrams to improve the model
```

## Example: Default behavior

This project comes with a predefined HTTP
[remote storage](/doc/commands-reference/remote). We can now just run
`dvc fetch` that will download the most recent `model.pkl`, `data.xml`, and
other files that are under DVC control into our local cache:

```dvc
$ dvc status --cloud
...
    deleted:            model.pkl
    deleted:            data/features/...

$ dvc fetch
...
(2/6): [##############################] 100% data/features/test.pkl
(3/6): [##############################] 100% model.pkl
(4/6): [##############################] 100% data/features/train.pkl
...

$ tree .dvc
.dvc
├── cache           <- dir .dvc/cache was created and populated
│   ├── 38
│   │   └── 63d0e317dee0a55c4e59d2ec0eef33
│   ├── 42
│   │   └── c7025fc0edeb174069280d17add2d4.dir
│   ├── ...
├── config
├── ...
```

> `dvc status --cloud` (or `-c`) compares local cache vs default remote.

As seen above, used without arguments, `dvc fetch` downloads all assets needed
by all DVC-files in the current branch, including for directories. The checksums
`3863d0e317dee0a55c4e59d2ec0eef33` and `42c7025fc0edeb174069280d17add2d4`
correspond to the `model.pkl` file and `data/features/` directory, respectively.

Let's link files from local cache to the workspace with:

```dvc
$ dvc checkout
Checking out '{'scheme': 'local', 'path': '.../example-get-started/model.pkl'}' with cache '3863d0e317dee0a55c4e59d2ec0eef33'.
Checking out '{'scheme': 'local', 'path': '.../example-get-started/data/...
```

## Example: Specific stages

> Please delete the `.dvc/cache` directory first (with `rm -Rf .dvc/cache`) to
> follow this example if you tried the previous one (**Default behavior**).

`dvc fetch` only downloads the data files of a specific stage when the
corresponding DVC-file (command target) is specified:

```dvc
$ dvc fetch prepare.dvc
...
(1/2): [##############################] 100% data/prepared/test.tsv
(2/2): [##############################] 100% data/prepared/train.tsv

$ tree .dvc/cache
.dvc/cache
├── 42
│   └── c7025fc0edeb174069280d17add2d4.dir
├── 58
│   └── 245acfdc65b519c44e37f7cce12931
├── 68
│   └── 36f797f3924fb46fcfd6b9f6aa6416.dir
└── 9d
    └── 603888ec04a6e75a560df8678317fb
```

> Note that `prepare.dvc` is the first stage in our example's pipeline.

Cache entries for the necessary directories, as well as the actual
`data/prepared/test.tsv` and `data/prepared/train.tsv` files were download,
checksums shown above.

## Example: With dependencies

After following the previous example (**Specific stages**), only the files
associated with the `prepare.dvc` stage file have been fetched. Several
dependencies/outputs of other pipeline stages are still missing from local
cache:

```dvc
$ dvc status -c
...
    deleted:            model.pkl
    deleted:            data/features/test.pkl
    deleted:            data/features/train.pkl
    deleted:            data/data.xml
```

One could do a simple `dvc fetch` to get all the data, but what if you only want
to retrieve the data up to our third stage, `train.dvc`? We can use the
`--with-deps` (or `-d`) flag:

```dvc
$ dvc fetch --with-deps train.dvc
...
(1/4): [##############################] 100% model.pkl
(2/4): [######                        ] 21% data/data.xml
(2/4): [##############################] 100% data/features/train.pkl
(3/4): [##############################] 100% data/features/test.pkl
(4/4): [##############################] 100% data/data.xml

$ tree .dvc/cache
.dvc/cache
├── 38
│   └── 63d0e317dee0a55c4e59d2ec0eef33
├── 42
│   └── c7025fc0edeb174069280d17add2d4.dir
├── 58
│   └── 245acfdc65b519c44e37f7cce12931
├── 68
│   └── 36f797f3924fb46fcfd6b9f6aa6416.dir
├── 9d
│   └── 603888ec04a6e75a560df8678317fb
├── a3
│   └── 04afb96060aad90176268345e10355
├── aa
│   └── 35101ce881d04b41d5b4ff3593b423
└── dc
    └── a9c512fda11293cfee7617b66648dc
```

Fetching using `--with-deps` starts with the target DVC-file (stage) and
searches backwards through its pipeline for data files to download into the
local cache. All the data for the second and third stages ("featurize" and
"train") has now been downloaded to cache. We could now use `dvc checkout` to
get the data files needed to reproduce this pipeline up to the third stage into
the workspace (with `dvc repro train.dvc`).

> Note that in this sample project, the last stage file `evaluate.dvc` doesn't
> add any more data files than those form previous stages so at this point all
> of the files for this pipeline are in local cache and `dvc status -c` would
> output `Pipelines are up to date.`
