# ast2vec [![Build Status](https://travis-ci.org/src-d/ast2vec.svg)](https://travis-ci.org/src-d/ast2vec) [![codecov](https://codecov.io/github/src-d/ast2vec/coverage.svg?branch=develop)](https://codecov.io/gh/src-d/ast2vec) [![PyPI](https://img.shields.io/pypi/v/ast2vec.svg)](https://pypi.python.org/pypi/ast2vec)

Machine Learning models on top of Abstract Syntax Trees.

Currently, there are implemented:

* id2vec, source code identifier embeddings
* docfreq, source code identifier document frequencies (part of TF-IDF)
* nBOW, weighted bag of vectors, as in [src-d/wmd-relax](https://github.com/src-d/wmd-relax)
* topic modeling

This project can be the foundation for [MLoSC](https://github.com/src-d/awesome-machine-learning-on-source-code) research and development. It abstracts feature extraction and working with models, thus allowing to focus on the higher level tasks.

It is written in Python3 and has been tested on Linux and macOS. ast2vec is tightly coupled with [Babelfish](http://doc.bblf.sh) and delegates all the AST parsing to it.

Here is the list of projects which are built with ast2vec:

* [vecino](https://github.com/src-d/vecino) - finding similar repositories
* [tmsc](https://github.com/src-d/tmsc) - topic modeling of repositories
* [role2vec](https://github.com/src-d/rol2vec) - AST node embedding and correction
* [snippet-ranger](https://github.com/src-d/snippet-ranger) - topic modeling of source code snippets

## Installation

```
pip3 install ast2vec
```

You need to have `libxml2` installed. E.g., on Ubuntu `apt install libxml2-dev`.

## Usage

This project exposes two interfaces: API and command line. The command line is

```
ast2vec --help
```

There is an example of using Python API [here](Doc/how_to_use_ast2vec.ipynb).

It exposes several tools to generate the models and setup the environment.

API is divided into two domains: models and training. The first is about using while the second
is about creating. Models: [Id2Vec](ast2vec/id2vec.py),
[DocumentFrequencies](ast2vec/df.py), [NBOW](ast2vec/nbow.py), [Cooccurrences](ast2vec/coocc.py).
Transformers (keras/sklearn style): [Repo2nBOWTransformer](ast2vec/repo2/nbow.py#L72),
[Repo2CooccTransformer](ast2vec/repo2/coocc.py#L101),
[PreprocessTransformer](ast2vec/id_embedding.py#L22),
[SwivelTransformer](ast2vec/id_embedding.py#L218) and
[PostprocessTransformer](ast2vec/id_embedding.py#L241).

## Docker image

```
docker build -t srcd/ast2vec .
BBLFSH_DRIVER_IMAGES="python=docker://bblfsh/python-driver:v0.8.2;java=docker://bblfsh/java-driver:v0.6.0" docker run -e BBLFSH_DRIVER_IMAGES -d --privileged -p 9432:9432 --name bblfsh bblfsh/server:v0.7.0 --log-level DEBUG
docker run -it --rm srcd/ast2vec --help
```

If the first command fails with

```
Cannot connect to the Docker daemon. Is the docker daemon running on this host?
```

And you are sure that the daemon is running, then you need to add your user to `docker` group:
refer to the [documentation](https://docs.docker.com/engine/installation/linux/linux-postinstall/#manage-docker-as-a-non-root-user).

## Algorithms

#### Identifier embeddings

We build the source code identifier co-occurrence matrix for every repository.

1. Clone or read the repository from disk.
2. Classify files using [enry](https://github.com/src-d/enry).
3. Extract [UAST](https://doc.bblf.sh/uast/specification.html) from each supported file.
4. [Split and stem](ast2vec/repo2/base.py#L160) all the identifiers in each tree.
5. [Traverse UAST](ast2vec/repo2/coocc.py#L86), collapse all non-identifier paths and record all
identifiers on the same level as co-occurring. Besides, connect them with their immediate parents.
6. Write the individual co-occurrence matrices.
7. [Merge](ast2vec/id_embedding.py#L50) co-occurrence matrices from all repositories. Write the
document frequencies model.
8. Train the embeddings using [Swivel](ast2vec/swivel.py) running on Tensorflow. Interactively view
the intermediate results in Tensorboard using `--logs`.
9. Write the identifier embeddings model.
10. Publish generated models to the Google Cloud Storage.

1-6 is performed with `repo2coocc` tool / `Repo2CooccTransformer` class,
7 with `id2vec_preproc` / `id_embedding.PreprocessTransformer`, 8 with `id2vec_train` / `id_embedding.SwivelTransformer`,
9 with `id2vec_postproc` / `id_embedding.PostprocessTransformer` and 10 with `publish`.

#### Weighted Bag of Vectors

We represent every repository as a weighted bag-of-vectors, provided by we've got document
frequencies ("docfreq") and identifier embeddings ("id2vec").

1. Clone or read the repository from disk.
2. Classify files using [enry](https://github.com/src-d/enry).
3. Extract [UAST](https://doc.bblf.sh/uast/specification.html) from each supported file.
4. [Split and stem](ast2vec/repo2/base.py#L160) all the identifiers in each tree.
5. Leave only those identifiers which are present in "docfreq" and "id2vec".
6. Set the weight of each such identifier as TF-IDF.
7. Set the value of each such identifier as the corresponding embedding vector.
8. Write the nBOW model.
9. Publish it to the Google Cloud Storage.

1-8 is performed with `repo2nbow` tool / `Repo2nBOWTransformer` class and 9 with `publish`.

#### Topic modeling

See [here](topic_modeling.md).

## Contributions
[![PEP8](https://img.shields.io/badge/code%20style-pep8-orange.svg)](https://www.python.org/dev/peps/pep-0008/)

We use [PEP8](https://www.python.org/dev/peps/pep-0008/) with line length 99 and ". All the tests
must pass:

```
unittest discover /path/to/ast2vec
```

## License

Apache 2.0.
