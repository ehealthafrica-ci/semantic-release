# semantic-release
[![Build Status](https://travis-ci.org/boennemann/semantic-release.svg?branch=master)](https://travis-ci.org/boennemann/semantic-release)
[![Dependency Status](https://david-dm.org/boennemann/semantic-release.svg)](https://david-dm.org/boennemann/semantic-release)
[![devDependency Status](https://david-dm.org/boennemann/semantic-release/dev-status.svg)](https://david-dm.org/boennemann/semantic-release#info=devDependencies)

[![NPM](https://nodei.co/npm/semantic-release.png?downloads=true&downloadRank=true&stars=true)](https://nodei.co/npm/semantic-release/)

## What is this thing even?

`semantic-release` is a toolset to fully automate your package's releases. This will determine not only which version to release, but also when – all without you having to care about it ever again.

This is fully integrated with the `npm` lifecycle, so all you have to do is to setup your CI to `npm publish`.

The goal of this package is to remove humans from version numbers and releases. The [SemVer](http://semver.org/) spec clearly and unambiguously defines when to increase the major, minor or patch part and still we tend to think we're clever when we ignore this, because marketing or something.

![https://twitter.com/trodrigues/status/509301317467373571](https://cloud.githubusercontent.com/assets/908178/6091690/cc86f58c-aeb8-11e4-94cb-15f15f486cde.png)

## How does this work?

Conventions, conventions, conventions. Instead of dumping [funny lols](http://whatthecommit.com/) into our commit messages, we can take some time to think about what we changed in the codebase and write it down. Following formalized conventions it this then possible to not only generate a meaningful changelog, but to determine the next semantic version to release. Currently the only supported style is the [AngularJS Commit Message Convention](https://docs.google.com/document/d/1QrDFcIiPjSLDn3EL15IJygNPiHORgU1_OOAqWjiDU5Y/edit) style, but feel free to formalize your own style, write a parser for it, and send a PR to this package. 

```
<type>(<scope>): <subject>
<BLANK LINE>
<body>
```

Examples:

```
feat(ruler): add inches as well as centimeters
```

```
fix(protractor): fix 90 degrees counting as 91 degrees
```

```
fix(pen): use blue ink instead of red ink

BREAKING CHANGE: Pen now uses blue ink instead of red.

To migrate, change your code from the following:

`pen.draw('blue')`

To:

`pen.draw('red')`
```

[Synopsis](https://github.com/ajoslin/conventional-changelog/blob/master/CONVENTIONS.md)

The preferred configuration is the "try to release on every push" mode. What it does is that everytime a build passes `npm publish` is executed. 

### The `prepublish` step

Before `npm` actually gets to publish a new version `semantic-release`'s `prepublish` step does the following:

- Analyze the commits since the last version was published
- Decide on the release type (`major`|`minor`|`patch`) or abort if nothing changed
- Get the last published version from the registry
- Increase the last version with the determined type
- Write the new version to the package

### The `publish` step

`npm` does its thing.

### The `postpublish` step

After `npm` published the new version the `postpublish` step does this:

- Generate a changelog
- Create a new [GitHub Release](https://help.github.com/articles/about-releases/) with the changelog as body

Note: The GitHub Release automatically creates a tag, too.

Note: This is tied to GitHub, feel free to send PRs for other services.

Note: `semantic-release` works around a limitation in `npm`'s `prepublish` step. Once a version is published it prints an error that you can *safely ignore* [npm/npm#7118](https://github.com/npm/npm/issues/7118).

## How do I set this up?

### Installation

First of all you need to install `semantic-release` and save it as a `devDependency`.

```bash
npm install --save-dev semantic-release
# Btw, if you're feeling lazy you can just type this – it's the same thing.
npm i -D semantic-release
```

### Package

```bash
./node_modules/.bin/semantic-release setup
```

What this does:

#### Scripts

The setup command configures `scripts` inside the `package.json`:

```json
"scripts": {
  "prepublish": "semantic-release pre",
  "postpublish": "semantic-release post"
}
```

Note: If you have already configured `scripts` for `prepublish` or `postpublish` they're just executed one after another. For example: `"npm run 6to5 && semantic-release pre"`.

#### Version

It would be preferable not to have a version field in the `package.json` at all, but due to an `npm` limitation it is required to have a _not yet published_ version in there [npm/npm#7118](https://github.com/npm/npm/issues/7118). Because of this the version gets changed to `"0.0.0-semantically-released"` until `npm` _hopefully_ removes its limitations.

#### Repository

If you haven't defined your GitHub repository in the `package.json`s [repository field](https://docs.npmjs.com/files/package.json#repository) the remote `origin`'s repository is used. 

### CI Server

Inside your `.travis.yml`:

```yml 
language: node_js
node_js:
- iojs-v1
sudo: false
cache:
  directories:
  - node_modules
notifications:
  email: false
env:
  # Get your token here: https://github.com/settings/tokens/new
  # You should encrypt this:
  # `travis encrypt GH_TOKEN=<token> --add`
  global: GH_TOKEN=<github-access-token-with-acceess-to-your-repo>
deploy:
  provider: npm
  email: <your-npm-mail@example.com>
  # Very important. Don't forget this one.
  skip_cleanup: true
  # Travis currently only supports the old auth key format.
  # Do `echo -n "<username>:<password>" | base64` to get it.
  # You should encrypt this:
  # `travis encrypt $(echo -n "<username>:<password>" | base64) --add deploy.api_key`
  api_key: <npm-api-key>
  on:
    branch: master
    repo: <user>/<repo>
```

Note: For once this isn't tied to a specific service, but example configuration is shown for [Travis CI](https://travis-ci.org/). Feel free to contribute configuration of other servers or services.

Note: You should [encrypt](http://docs.travis-ci.com/user/environment-variables/#sts=Secure Variables) your api keys and tokens.

Note: Your CI environment has to export `CI=true` in order for `semantic-release` not to automatically perform a dry run. Travis CI does this by default. 

Note: It is crucial that your CI server also fetches all tags when checking out your repository. Travis CI does this by default. 

Note: If you have a more sophisticated build with multiple jobs you should have a look at [travis-after-all](https://github.com/dmakhno/travis_after_all), which is also configured for this [package](.travis.yml).

## ITYM*FAQ*LT
> I think you might frequently ask questions like these

### Why is the `package.json`'s version not updated in my repository?

The `npm` docs even state:

> The most important things in your package.json are the name and version fields. Those are actually required, and your package won't install without them.
> -- [npm docs](https://docs.npmjs.com/files/package.json#version)

While this entirely true the version number doesn't have to be checked into source control. `semantic-release` takes care of the version field right before `npm publish` uses it – and this is the only point when it _really_ is required.

### Is there a way to preview which version would currently get published?

If you're running `npm publish` locally `semantic-release` automatically performs a dry run. This does log the version that would currently get published, but only if you `git fetch --tags` before.

### Can I run this on my own machine rather than on a CI server?

Of course you can, but this doesn't mean you should. Running your tests on an independent machine before releasing software is a crucial part of this workflow. Also it is a pain to set this up locally, with GitHub tokens lying around and everything. That said, you can either set the environment variable `CI=true`, or run the scripts with `--debug=false` explicitly. Don't forget to export `GH_TOKEN=your_token` as well.

### Can I manually trigger the release of a specific version?

You can trigger a release by pushing to your repository. You deliberately can not trigger a _specific_ version release, because this is the whole point of `semantic-release`. Start your packages with `1.0.0` and semver on.  

Note: pre-release flags are kind of an exeption here and a solution for them is being thought of. If you have one please open an issue. For the time being: Have a look at the next question.

### How do I get back to good ol' `npm publish`?

`npm` offers the `--ignore-scripts` flag. Doing `npm publish --ignore-scripts` doesn't execute the `prepublish` and `postpublish` scripts.

### Is it _really_ a good idea to release on every push?

It is indeed a great idea because it _forces_ you to follow best practices. If you don't feel comfortable making every passing feature or fix on your master branch addressable via `npm` you might not treat your master right. Have a look at [branch workflows](https://guides.github.com/introduction/flow/index.html). If you still think you should have control over the exact point in time of your release, e.g. because you are following a release schedule, configure your CI server to release only on the `production`/`deploy`/`release` branch and push your code there in certain intervals.

### Why should I trust `semantic-release` with my releases? What if it breaks?

`semantic-release` has a full integration-test suite that tests _actual_ `npm` publishes and _actual_ GitHub Releases (with private registry/API) on node.js `^0.10`, `^0.12` and io.js `^1`. A new version won't get published if it doesn't pass on all these engines. 

## License

MIT License 
2015 © Stephan Bönnemann and [contributors](https://github.com/boennemann/semantic-release/graphs/contributors)
