# Greenkeeper

We’re using [Greenkeeper](https://greenkeeper.io) to automatically notify us of
changes in our `package.json` dependencies.

While staying on the latest version of everything is not strictly necessary,
especially for a project that’s in maintenance mode and working fine, it’s still
good to stay on top of what’s changing, in case there’s a security patch or
something to apply.

Greenkeeper works by opening PRs with the updated `package.json` file and
letting CI (in our case, Travis) run the tests. It has a secret API key on
Travis that lets it post a commit back to the PR to update the yarn.lock file.
(Note that currently that API key is tied to Fin’s account. We should get a
dedicated account for this.)

## How to respond to a Greenkeeper PR

 1) Read the bits of the PR that mention changes in this version, including git
    commits. Greenkeeper will point out if the new version is inside or outside
    the version range specified in `package.json`. Be wary of major version
    changes that require updates to maintain compatibilty. *Make sure that the
    time investment in updating the code is worth it.*

 1) Use GitHub (or locally) bring the branch up-to-date with the base branch
    (probably “develop”)

 1) Check out the branch locally.

 1) **Run `yarn install`** This is in bold because unless you do this, you won’t
    actually be testing the updated packages.

 1) If you had to merge develop into the branch, and if you’re doing multiple
    Greenkeeper PRs in the same sitting, it is *highly likely* that `yarn.lock`
    has changed at this point. *Don’t forget to add a commit for it.*

 1) Exercise the areas of the code related to the package that’s being updated.
    For the most part, *don’t rely on the unit tests to tell you that
    everything’s fine*. Remember that we don’t have 100% coverage in lines of
    code, let alone 100% coverage on features.

 1) If things are broken, make the decision about whether the time investment is
    worth it to upgrade, or if it’s feasible. For example, we’re currently
    sticking to ESLint 3 rather than 4 because not all of the ESLint plugins
    have updated to support the new version. That’s ok.

 1) When everything is working and you’ve commited and pushed any changes in
    your working directory, merge the PR.

 1) If you’ve decided not to upgrade this package, leave a comment in the PR
    about why and then close it.

## Tips

Seriously make sure to run `yarn install` on the branch, and commit any changes
to `yarn.lock`.

If Greenkeeper keeps bothering you about a version that you can’t upgrade to,
you can pin the current version in `package.json`.

If this ends up not being worth it, disable the integration.
