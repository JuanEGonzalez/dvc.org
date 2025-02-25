# Data and Model Files Versioning

> This document provides an overview of the basic files versioning DVC workflow.
> To get a hands on experience and try it yourself we recommend you to follow
> along the [Versioning](/doc/get-started/example-versioning) _Get Started_
> example.

DVC allows storing and versioning data files and directories, ML models,
intermediate results with Git, without checking the file contents into Git. It
is useful when dealing with files that are too large for Git to handle. DVC
stores information about your data file in a special
[DVC-file](/doc/user-guide/dvc-file-format), that has a description of a file
that can be used for versioning. DVC supports various types of remote locations
for your data files and allows you to easily store and share your data alongside
your code.

![](/static/img/model-versioning-diagram.png)

In this very basic scenario, DVC is a better replacement for `git-lfs` (see
[Related Technologies](/doc/understanding-dvc/related-technologies)) and ad-hoc
scripts on top of Amazon S3 (or any other cloud) that are usually used to manage
ML <abbr>data artifacts</abbr> like data files, models, etc. Unlike `git-lfs`,
DVC doesn't require installing a server; it can be used on-premises (NAS, SSH,
for example) or with any major cloud provider (S3, Google Cloud, Azure).

Let's say you already have a <abbr>project</abbr> that uses a bunch of images
stored in `images/` directory and has a `model.pkl` file – model file deployed
to production.

```dvc
$ ls images
0001.jpg 0002.jpg 0003.jpg 0004.jpg ...

$ ls
model.pkl ...
```

To start using dvc and keeping track of a model and images we need first to
initialize it in your repository:

```dvc
$ dvc init
```

DVC creates a `.dvc/` directory that stores special files and also a
`.dvc/cache` directory that will be used to store cache for your data.

```dvc
$ git status

...
    new file:   .dvc/.gitignore
    new file:   .dvc/config

$ git commit -m "Initialize dvc"
```

Start tracking images and models with `dvc add`:

```dvc
$ dvc add images
$ dvc add model.pkl
```

> Refer also to `dvc run` for more advanced ways to version data.

Commit your changes:

```dvc
$ git status

...
Untracked files:
    .gitignore
    images.dvc
    model.pkl.dvc

$ git add .gitignore images.dvc model.pkl.dvc
$ git commit -m "track images and models with dvc"
```

There are two ways to get to the previous version of the dataset or model: a
full <abbr>workspace</abbr> checkout or checkout of a specific data or model
file. Let's consider the full checkout first. It's quite straightforward:

> `v1.0` is a Git tag that should be created in advance to identify the dataset
> version you are interested in, it can be just a Git commit hash instead.

```dvc
$ git checkout v1.0
$ dvc checkout
```

These commands will restore the working tree to the first snapshot we made -
code, dataset and model files. DVC optimizes this operation internally to avoid
copying dataset or model files each time. So `dvc checkout` is quick even if you
have large dataset or model files.

On the other hand, if we want to keep the current version of code and go back to
the previous dataset only, we can do something like this (make sure that you
don't have uncommitted changes in the `data.dvc`):

```dvc
$ git checkout v1.0 data.dvc
$ dvc checkout data.dvc
```

If you run `git status` you will see that `data.dvc` is modified and currently
points to the `v1.0` of the dataset. While code and model files are from the
`v2.0` version.

![](/static/img/versioning.png)

To share your data with others you need to setup a remote repository. See the
[Share Data And Model Files](/doc/use-cases/share-data-and-model-files) use case
to get a high level overview on how to setup it and use `dvc pull` and
`dvc push` commands to collaborate. Please, don't forget to see the
[Versioning](/doc/get-started/example-versioning) example to get a hands-on
experience with datasets and models versioning.
