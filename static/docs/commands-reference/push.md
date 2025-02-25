# push

Uploads files and directories under DVC control to the
[remote storage](/doc/commands-reference/remote).

## Synopsis

```usage
usage: dvc push [-h] [-q | -v] [-j JOBS] [--show-checksums]
                [-r REMOTE] [-a] [-T] [-d] [-R]
                [targets [targets ...]]

positional arguments:
  targets        Limit command scope to these DVC-files. Using -R,
                 directories to search DVC-files in can also be given.
```

## Description

The `dvc push` command is the twin pair to the `dvc pull` command, and together
they are the means for uploading and downloading data to and from remote
storage. [Data sharing](/doc/use-cases/share-data-and-model-files) across
environments and preserving data versions (input datasets, intermediate results,
models, metrics, etc) remotely (S3, SSH, GCS, etc) are the most common use cases
for these commands.

The `dvc push` command allows one to upload data to remote storage. It doesn't
save any changes in the code or DVC-files. Those should be saved by using
`git commit` and `git push`.

Under the hood a few actions are taken:

- The push command by default uses all
  [DVC-files](/doc/user-guide/dvc-file-format in the current version. The
  command-line options listed below will either limit or expand the set of
  DVC-files to consult.

- For each output referenced from each selected DVC-files, it finds a
  corresponding entry in the local <abbr>cache</abbr>. DVC checks if the entry
  exists, or not, in the remote simply by looking for it using the checksum.
  From this DVC gathers a list of files missing from the remote storage.

- Upload the cache files missing from the remote cache, if any, to the remote.

The DVC `push` command always works with a remote storage, and it is an error if
none are specified on the command line nor in the configuration. If a
`--remote REMOTE` option is not specified, then the default remote, configured
with the `core.config` config option, is used. See `dvc remote`, `dvc config`
and this [example](/doc/get-started/configure) for more information on how to
configure a remote.

With no arguments, just `dvc push` or `dvc push --remote REMOTE`, it uploads
only the files (or directories) that are new in the local repository to the
remote cache. It will not upload files associated with earlier versions or
branches of the <abbr>project</abbr> directory, nor will it upload files which
have not changed.

The command `dvc status -c` can list files that are new in the local cache and
are referenced in the <abbr>workspace</abbr>. It can be used to see what files
`dvc push` would upload.

The `dvc status -c` command can show files which exist in the remote cache and
not exist in the local cache. Running `dvc push` from the local cache does not
remove nor modify those files in the remote cache.

If one or more `targets` are specified, DVC only considers the files associated
with those DVC-files. Using the `--with-deps` option, DVC tracks dependencies
backward from the target [stage files](/doc/commands-reference/run), through the
corresponding [pipelines](/doc/commands-reference/pipeline), to find data files
to push.

## Options

- `-r REMOTE`, `--remote REMOTE` specifies which remote cache (see
  `dvc remote list`) to push to. The value for `REMOTE` is a cache name defined
  using the `dvc remote` command. If no `REMOTE` is given, or if no remote's are
  defined in the project, an error message is printed. If the option is not
  specified, then the default remote, configured with the `core.config` config
  option, is used.

- `-a`, `--all-branches` - determines the files to upload by examining DVC-files
  in all branches of the project repository (if using Git). It's useful if
  branches are used to track checkpoints of an experiment or project.

- `-T`, `--all-tags` - the same as `-a`, `--all-branches` but tags are used to
  save different experiments or project checkpoints.

- `-d`, `--with-deps` - determines files to upload by tracking dependencies to
  the target DVC-files (stages). This option only has effect when one or more
  `targets` are specified. By traversing all stage dependencies, DVC searches
  backward from the target stages in the corresponding pipelines. This means DVC
  will not push files referenced in later stages than the `targets`.

- `-R`, `--recursive` - `targets` is expected to contain at least one directory
  path for this option to have effect. Determines the files to push by searching
  each target directory and its subdirectories for DVC-files to inspect.

- `-j JOBS`, `--jobs JOBS` - specifies number of jobs to run simultaneously
  while uploading files to the remote cache. The effect is to control the number
  of files uploaded simultaneously. Default is `4 * cpu_count()`. For example
  with `-j 1` DVC uploads one file at a time, with `-j 2` it uploads two at a
  time, and so forth. For SSH remotes default is set to 4.

- `-h`, `--help` - prints the usage/help message, and exit.

- `-q`, `--quiet` - do not write anything to standard output. Exit with 0 if no
  problems arise, otherwise 1.

- `-v`, `--verbose` - displays detailed tracing information.

## Examples

For using the `dvc push` command, remote storage must be defined. (See
`dvc remote`.) For an existing <abbr>project</abbr>, remotes are usually already
set up and you can use `dvc remote list` to check them. Just to remind how it is
done and set a context for the example, let's define an SSH remote with the
`dvc remote add` command:

```dvc
$ dvc remote add r1 ssh://_username_@_host_/path/to/dvc/cache/directory
$ dvc remote list
r1	ssh://_username_@_host_/path/to/dvc/cache/directory
```

> DVC supports several remote types. For details, see the
> [`remote add`](/doc/commands-reference/remote/add) documentation.

Push all data file caches from the current Git branch to the default remote:

```dvc
$ dvc push

(1/8): [######################] 100% images/0001.jpg
(2/8): [######################] 100% images/0002.jpg
...
(7/8): [######################] 100% images/0007.jpg
(8/8): [###########           ] 57% model.pkl
```

Push outputs of a specific DVC-file:

```dvc
$ dvc push data.zip.dvc

[######################] 100% data.zip
```

## Example: With dependencies

Demonstrating the `--with-deps` flag requires a larger example. First, assume a
[pipeline](/doc/commands-reference/pipeline) has been setup with these
[stages](/doc/commands-reference/run):

```dvc
$ dvc pipeline show

data/Posts.xml.zip.dvc
Posts.xml.dvc
Posts.tsv.dvc
Posts-test.tsv.dvc
matrix-train.p.dvc
model.p.dvc
Dvcfile
```

Imagine the local cache has been modified such that the data files in some of
these stages should be uploaded to the remote cache.

```dvc
$ dvc status --cloud

  new:            data/model.p
  new:            data/matrix-test.p
  new:            data/matrix-train.p
```

One could do a simple `dvc push` to share all the data, but what if you only
want to upload part of the data?

```dvc
$ dvc push --remote r1 --with-deps matrix-train.p.dvc

(1/2): [####################] 100% data/matrix-test.p data/matrix-test.p
(2/2): [####################] 100% data/matrix-train.p data/matrix-train.p

... Do some work based on the partial update

$ dvc push --remote r1 --with-deps model.p.dvc

(1/1): [####################] 100% data/model.p data/model.p

... Push the rest of the data

$ dvc push --remote r1

Everything is up to date.

$ dvc status --cloud

Data and pipelines are up to date.
```

With the first `dvc push` we specified a stage in the middle of this pipeline
(`matrix-train.p.dvc`) while using `--with-deps`. DVC started with that DVC-file
and searched backwards through the pipeline for data files to upload. Because
the `model.p.dvc` stage occurs later, its data was not pushed.

Then we ran `dvc push` specifying the last stage, `model.p.dvc`, and its data
was uploaded. Finally, we ran `dvc push` and `dvc status` with no options to
double check that all data had been uploaded.

## Example: What happens in the cache

Let's take a detailed look at what happens to the DVC cache as you run an
experiment locally and push data to a remote cache. To set the example consider
having created a <abbr>workspace</abbr> that contains some code and data, and
having set up a remote cache.

Some work has been performed in the local workspace, and it contains new data to
upload to the shared remote cache. When running `dvc status --cloud` the report
will list several files in `new` state. By looking in the cache directories we
can see exactly what that means.

```dvc
$ tree .dvc/cache
.dvc/cache
├── 02
│   └── 423d88d184649a7157a64f28af5a73
├── 0b
│   └── d48000c6a4e359f4b81285abf059b5
├── 38
│   └── 64e70211d3bdb367ad1432bfc14c1f.dir
├── 3f
│   └── 957fa0f1bb46534d07f4fc2116d73d
├── 4a
│   └── 8c47036c79c01522e79ac0f518d0f7
├── 5e
│   └── 4a7d0cbe26eda55624439661db925d
├── 6c
│   └── 3074754e3a9b563b62c8f1a38670dc
├── 77
│   └── bea77463abe2b7c6b4d13f00d2c7b4
├── 88
│   └── c3db1c257136090dbb4a7ddf31e678.dir
└── f4

10 directories, 9 files
$ tree ../vault/recursive
../vault/recursive
├── 0b
│   └── d48000c6a4e359f4b81285abf059b5
├── 4a
│   └── 8c47036c79c01522e79ac0f518d0f7
├── 6c
│   └── 3074754e3a9b563b62c8f1a38670dc
├── 88
│   └── c3db1c257136090dbb4a7ddf31e678.dir
└── f4
    └── 7482b18ecca728ba4ae931e5d568fb

5 directories, 5 files
```

The directory `.dvc/cache` is the local cache, while `../vault/recursive` is the
remote cache. This listing clearly shows the local cache has more files in it
than the remote cache. Therefore `new` literally means that new files exist in
the local cache compared to the remote.

Next we can upload part of the data from the local cache to a remote using the
command `dvc push --with-deps STAGE.dvc`. Remember that `--with-deps` searches
backwards from the DVC-file `targets` to locate files to upload, and does not
upload files in subsequent stages.

After doing that we can inspect the remote cache again:

```dvc
$ tree ../vault/recursive
../vault/recursive
├── 0b
│   └── d48000c6a4e359f4b81285abf059b5
├── 38
│   └── 64e70211d3bdb367ad1432bfc14c1f.dir
├── 4a
│   └── 8c47036c79c01522e79ac0f518d0f7
├── 5e
│   └── 4a7d0cbe26eda55624439661db925d
├── 6c
│   └── 3074754e3a9b563b62c8f1a38670dc
├── 77
│   └── bea77463abe2b7c6b4d13f00d2c7b4
├── 88
│   └── c3db1c257136090dbb4a7ddf31e678.dir
└── f4
    └── 7482b18ecca728ba4ae931e5d568fb

8 directories, 8 files
```

The remote cache now has some of the files which had been missing, but not all
of them. Indeed `dvc status --cloud` still lists a couple files as `new`. We can
clearly see this in that a couple files are in the local cache and not in the
remote cache.

After running `dvc push` to cause all files to be uploaded the remote cache now
has all the files:

```dvc
$ tree ../vault/recursive
../vault/recursive
├── 02
│   └── 423d88d184649a7157a64f28af5a73
├── 0b
│   └── d48000c6a4e359f4b81285abf059b5
├── 38
│   └── 64e70211d3bdb367ad1432bfc14c1f.dir
├── 3f
│   └── 957fa0f1bb46534d07f4fc2116d73d
├── 4a
│   └── 8c47036c79c01522e79ac0f518d0f7
├── 5e
│   └── 4a7d0cbe26eda55624439661db925d
├── 6c
│   └── 3074754e3a9b563b62c8f1a38670dc
├── 77
│   └── bea77463abe2b7c6b4d13f00d2c7b4
├── 88
│   └── c3db1c257136090dbb4a7ddf31e678.dir
└── f4
    └── 7482b18ecca728ba4ae931e5d568fb

10 directories, 10 files

$ dvc status --cloud

Data and pipelines are up to date.
```

And running `dvc status --cloud` verifies that indeed there are no more files to
upload to the remote cache.
