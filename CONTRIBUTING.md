Want to contribute? Great! First, read this page.

* [Before You Contribute](#before-you-contribute)
  * [The Small Print](#the-small-print)
* [Large Language Models](#large-language-models)
* [Development Dependencies](#development-dependencies)
* [Writing Code](#writing-code)
  * [Changing the Language](#changing-the-language)
  * [Changing the Node API](#changing-the-node-api)
  * [Synchronizing](#synchronizing)
  * [File Headers](#file-headers)
* [Release Process](#release-process)

## Before You Contribute

Before we can use your code, you must sign the
[Google Individual Contributor License Agreement][cla] (CLA), which you can do
online. The CLA is necessary mainly because you own the copyright to your
changes, even after your contribution becomes part of our codebase, so we need
your permission to use and distribute your code. We also need to be sure of
various other things—for instance that you'll tell us if you know that your code
infringes on other people's patents. You don't have to sign the CLA until after
you've submitted your code for review and a member has approved it, but you must
do it before we can put your code into our codebase.

[cla]: https://cla.developers.google.com/about/google-individual

Before you start working on a larger contribution, you should get in touch with
us first through the issue tracker with your idea so that we can help out and
possibly guide you. Coordinating up front makes it much easier to avoid
frustration later on.

### The Small Print

Contributions made by corporations are covered by a different agreement than the
one above, the
[Software Grant and Corporate Contributor License Agreement][corporate cla].

[corporate cla]: https://developers.google.com/open-source/cla/corporate

## Large Language Models

Do not submit any code or prose written or modified by large language models or
"artificial intelligence" such as GitHub Copilot or ChatGPT to this project.
These tools produce code that looks plausible, which means that not only is it
likely to contain bugs those bugs are likely to be difficult to notice on
review. In addition, because these models were trained indiscriminately and
non-consensually on open-source code with a variety of licenses, it's not
obvious that we have the moral or legal right to redistribute code they
generate.

## Development Dependencies

1. [Install the Dart SDK][]. If you download an archive manually rather than
   using an installer, make sure the SDK's `bin` directory is on your `PATH`.

2. In this repository, run `dart pub get`. This will install all of the Dart
   dependencies.

3. [Install Node.js][]. This is only necessary if you're making changes to the
   language or to Dart Sass's Node API.

[Install the Dart SDK]: https://www.dartlang.org/install
[Install Node.js]: https://nodejs.org/en/download/

## Writing Code

Dart Sass follows the standard [Dart style guide][] wherever possible, including
using the [Dart formatter][] on all code. We also try to have no
[Dart analyzer][] warnings or hints, although if one sneaks in for a few
revisions that's not a big deal.

[Dart style guide]: https://www.dartlang.org/guides/language/effective-dart
[Dart formatter]: https://github.com/dart-lang/dart_style#readme
[Dart analyzer]: https://www.dartlang.org/tools/analyzer

Before you send a pull request, we recommend you run the following steps:

* `dart run grinder` will reformat your code using the Dart formatter to make
  sure it's nice and neat, and [run the synchronizer](#synchronizing) on
  asynchronous files.

* `dart analyze lib test` will run Dart's static analyzer to ensure that there
  aren't any obvious bugs in your code. If you're using a Dart-enabled IDE, you
  can also just check that there aren't any warnings in there.

* `dart run test -x node` will run the tests for the Dart VM API. These are a
  good sanity check, but they aren't comprehensive; GitHub Actions will also run
  Node.js API tests and Sass language tests, all of which must pass before your
  pull request is merged. See [Changing the Language](#changing-the-language)
  and [Changing the Node API](#changing-the-node-api) for more details.

### Changing the Language

If you're making a change to the Sass language, either to fix a bug or add a
feature, you'll need to write tests in the [sass-spec][] repository. This
repository contains language tests that are shared among the main Sass
implementations. Any new feature should be thoroughly tested there, and any bug
should have a regression test added.

[sass-spec]: https://github.com/sass/sass-spec

To create a new spec:

* [Fork sass-spec](https://help.github.com/articles/fork-a-repo/).

* [Install Node.js][] v14.14 or newer.

* ```sh
  # Replace $USER with your GitHub username.
  git clone https://github.com/$USER/sass-spec
  cd sass-spec
  npm install
  ```

* For each test case you want to add:

  * Create a directory within `sass-spec/spec/` for your test. Don't worry too
    much about finding exactly the right place, we'll sort that out during code
    review.

  * Following the [spec style guide][], create an `hrx` file that exercises your
    language change, verifying that the change produces expected output/errors.

    [spec style guide]: https://github.com/sass/sass-spec/blob/master/STYLE_GUIDE.md

* If you're adding a new language feature, it probably won't be supported by
  LibSass yet. You can indicate this and keep tests passing by adding an
  `options.yml` file like this to the directory containing your tests:

  ```yaml
  ---
  :ignore_for:
  - libsass
  ```

  If you're fixing a bug, you'll only need to do this if the bug also appears
  in other Sass implementations.

* Make sure all the language tests, including the new ones, are passing by
  running this within `sass-spec/`:

  ```sh
  # Replace .. with the path to dart-sass if it's not the parent directory.
  npm run sass-spec -- --dart ..
  ```

  * You can also run specs within a single directory:

    ```sh
    npm run sass-spec --dart .. spec/my/new/feature
    ```

  * If you pass the `--interactive` flag, the spec runner will stop each time a
    spec fails and ask you what to do about the failure.

* Once you've added specs and they're passing for Dart Sass, create a pull
  request for [sass-spec][] with `[skip dart-sass]` at the end of the
  message. This tells sass-spec not to run tests against the old version of Dart
  Sass, since it doesn't have your changes yet.

* Finally, create a pull request for Dart Sass with a link to the sass-spec pull
  request at the end of the message. This tells Dart Sass to test against your
  new sass-spec tests.

### Changing the Node API

Most of Dart Sass's code is shared between Dart and Node.js, but the API that's
exported by the [`sass`][npm] npm package is Node-specific. It's defined using
Dart's [JS interop package][], and it's tested by compiling the Dart package to
JS and loading that JS using JS interop to best simulate the conditions under
which it will be used in the real world.

[npm]: https://www.npmjs.com/package/sass
[JS interop package]: https://pub.dartlang.org/packages/js

The tests for the Node API live in `test/node_api`. Before running them, and any
time you make a change to Dart Sass, run `dart run grinder before-test` to
compile the Dart code to JavaScript (note that you don't need to recompile if
you've only changed the test code). To run Node tests, just run
`dart run test -t node`.

### Synchronizing

Dart Sass supports two modes of operation: synchronous ([`compile()`][] and
[`compileString()`][]), which requires all importers and custom functions to be
synchronous themselves, and asynchronous ([`compileAsync()`][] and
[`compileStringAsync()`][]), which allows importers and custom functions to be
asynchronous. These modes use essentially identical logic, but because Dart
represents synchronous and asynchronous computations in fundamentally different
ways they can't share code.

[`compile()`]: https://www.dartdocs.org/documentation/sass/latest/sass/compile.html
[`compileString()`]: https://www.dartdocs.org/documentation/sass/latest/sass/compileString.html
[`compileAsync()`]: https://www.dartdocs.org/documentation/sass/latest/sass/compileAsync.html
[`compileStringAsync()`]: https://www.dartdocs.org/documentation/sass/latest/sass/compileStringAsync.html

To avoid colossal amounts of duplicated code, we have a few files that are
written in an asynchronous style originally and then compiled to their
synchronous equivalents using `dart run grinder synchronize`. In particular:

* `lib/src/visitor/async_evaluate.dart` is compiled to
  `lib/src/visitor/evaluate.dart`.
* `lib/src/async_environment.dart` is compiled to `lib/src/environment.dart`.

When contributing code to these files, you should make manual changes only to
the asynchronous versions and run `dart run grinder` to compile them to their
synchronous equivalents.

Note that the `lib/src/callable/async_built_in.dart` and
`lib/src/callable/built_in.dart` files are *not* automatically synchronized;
they're so small and would require so many special cases that they're not worth
automating.

### File Headers

All files in the project must start with the following header.

```dart
// Copyright 2021 Google LLC. Use of this source code is governed by an
// MIT-style license that can be found in the LICENSE file or at
// https://opensource.org/licenses/MIT.
```

## Release Process

Most of the release process is fully automated on GitHub actions, triggered by
pushing a tag matching the current `pubspec.yaml` version. However, there are a
few things to do before pushing that tag:

* Make sure the `pubspec.yaml` version doesn't end in `-dev`. (This is a Dart
  convention to distinguish commits that aren't meant for release from commits
  that are.)

* Make sure that `CHANGELOG.md` has an entry for the current version.

* Make sure that any packages in `pkg` depend on the current version of `sass`.

* Increment the versions of all packages in `pkg`. These should be incremented
  at least as much as the `sass` version, and more if you add a new API that's
  exposed by one of those packages.

* Make sure that every package in `pkg`'s `CHANGELOG.md` has an entry for its
  current version.

You *don't* need to create tags for packages in `pkg`; that will be handled
automatically by GitHub actions.
