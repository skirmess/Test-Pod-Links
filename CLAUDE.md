# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`Test::Pod::Links` is a single-module CPAN distribution (`lib/Test/Pod/Links.pm`)
that checks every HTTP/S link inside an `L<>` tag of a distribution's Pod by
issuing an `HTTP::Tiny` `head` request. It is meant to be used as an author test
in a consumer's `xt/` directory.

## Commands

```sh
prove -l                       # run the whole t/ suite
prove -lv t/pod_file_ok.t      # run one test file, verbose
prove -lvr t/                  # recursive
prove -lr xt/author            # author tests (perlcritic, perltidy, spelling, versions, ...)

perl Makefile.PL && make && make test   # EXTENDED_TESTING=1 enables extra checks
```

Dependencies: `cpanm --installdeps --with-develop .`
(add `--with-feature dzil` only if you need to build a release tarball).

## Generated files — do not hand-edit

The distribution is built with Dist::Zilla using the external bundle
`@Author::SKIRMESS` from https://github.com/skirmess/perl-dzil-bundle (see
`dist.ini`; the bundle is checked out into the git-ignored `dzil-bundle/`).

Many files carry an `Automatically generated file; DO NOT EDIT.` header and are
regenerated from that bundle rather than maintained here — everything under
`xt/`, `.perltidyrc`, `.xtfilesrc`, `t/00-load.t`, `.github/workflows/test.yml`,
`README`, `README.md`, `META.*`, and `Makefile.PL`. `Makefile.PL`'s prereq lists
and the module `$VERSION` come from Dist::Zilla, so a new dependency belongs in
the module's `use` statements; changing `Makefile.PL` alone is not enough.
`README.md` is rendered from the Pod in `lib/Test/Pod/Links.pm` — edit the Pod.

Changes go in `Changes` under the `{{$NEXT}}` heading.

## Architecture

`Test::Pod::Links` is a plain-hash object with three collaborators:

- **File selection** is delegated entirely to `Test::XTFiles`/`XT::Files`
  (configured by the consumer's `.xtfilesrc`). `all_pod_files_ok` takes no
  arguments — that was the 0.003 API change — and filters the returned list
  through `Pod::Simple::Search->contains_pod`.
- **Link extraction** parses with `Pod::Simple::SimpleTree`, then
  `_extract_links_from_pod` walks the tree recursively collecting `L` element
  attribute hashes. Only nodes with `type eq 'url'` whose target matches
  `^https?://` are checked; everything else (module/section links) is ignored.
- **Fetching** goes through the `_ua` accessor, which only requires an object
  with a `head` method, and results are memoized per URL in `$self->{_cache}`
  for the lifetime of the object, so a URL repeated across many files is
  fetched once.

`ignore` (exact strings) and `ignore_match` (regexes) are compiled at
construction time into one alternation regex stored in `_ignore_regex`.

### Test::Builder constraints

The module writes to a file-scoped `Test::Builder` instance and deliberately
avoids subtests, `plan`, and (inside methods meant to be tested)
`done_testing`, because `Test::Builder::Tester` cannot verify those. Keep this
in mind before restructuring the output — see the comment block near the top of
the module.

## Tests

- `t/*.t` use `Test::Builder::Tester` (`test_out`/`test_fail`/`test_diag`/
  `test_test`) to assert the exact TAP the module emits, so changing a test
  name or diag string breaks these tests.
- No test hits the network. `t/lib/Local/HTTP/Tiny/Mock.pm` is a `head`
  replacement with recorded `HTTP::Tiny` responses keyed by URL (it dies on an
  unknown URL, and its `history` method proves caching works);
  `t/lib/Local/HTTP/NoUA.pm` is a minimal non-`HTTP::Tiny` user agent. Both are
  loaded via `FindBin`/`use lib`.
- `corpus/` holds Pod fixtures named for what they contain (`7_links_4_web.pod`,
  `malformed.pod`, ...) plus `corpus/dist1`..`dist3` and `corpus/empty`, which
  are miniature distributions. The `all_pod_files_ok-*.t` files `chdir` into one
  of those directories and never chdir back — each such scenario therefore needs
  its own `.t` file.
- `Test::MockModule`'s `redefine` (not `mock`) is the convention for stubbing
  `Test::Builder::done_testing`/`skip_all` and `pod_file_ok`.
