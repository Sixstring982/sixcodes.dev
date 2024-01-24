+++
title = "Using nix for your OCaml project"
date = 2024-01-22
[taxonomies]
tag=["OCaml","Nix"]
+++

# Summary

In this article, we discuss several strategies for starting to use `nix` for
`ocaml` development. We'll include strategies for new projects, as well as ones
for converting large existing codebases to be based on Nix.

# Background

## What is `nix` (at a high level)?

`nix` can refer to a few things within the `nix` ecosystem:

* A package manager, first and foremost.
  * Nix strives to be a "purely-functional" package manager, meaning that packages
    managed by `nix` are built in a consistent way.
* An operating system -- namely `NixOS`
  * This takes the `nix` package manager to its ultimate conclusion -- `NixOS`
    uses the `nix` package manager to configure your entire linux-based
    operating system.
* The `nix` language
  * This is a DSL for configuring the above. It's pretty similar to a
    lazily-evaluated JSON with functions.
* A command-line utility (called `nix`) for interfacing with the above.

## How does `nixpkgs` work?

`nixpkgs` is a repository of `nix` code, which is used by default to configure
the `nix` package manager. It's hosted at
[https://github.com/NixOS/nixpkgs](https://github.com/NixOS/nixpkgs).

`nixpkgs` provides several _channels_ -- effectively, these are special git
commits which we can point the `nix` package manager at. Channelse come in a few
different flavors:

* Stable channels -- e.g. `nixpkgs-23.11` -- do not change after creation
  (besides bug fixes and security pathces), and are released every six months
* `nixpkgs-unstable` is pinned to the repo main branch, so new contributions
  to `nixpkgs` can be pulled in to your project regularly.

You can also clone/fork `nixpkgs` and point `nix` to this repo instead. This
can be useful for adding e.g. private repositories, but other mechanisms like
overlays are probably better for this.

## What's an overlay?

A `nix` overlay is a function which can be used to override package definitions
from a `nix` channel. You can use these to get `nix` to use different package
versions, or provide new packages that aren't in the public `nixpkgs`.

## What's a flake?

I like to think of a flake as "`nix`'s version of `package.json`".

Technically, it's a standardized way to provide a project-level `nix`
configuration, which defines a few things:

* Inputs (which are other flakes that your flake depends on)
  * Note that `nixpkgs` is conveniently also a flake, making it easy to depend
    on!
* Outputs, definining a few things:
  * Packages, which other flakes can depend on
  * Development environments, which come into scope when using `nix develop`

This is an extremely useful tool for just about any software project. We'll
be writing a flake later on for our OCaml project.

## How does `nix` fit into an existing OCaml project?

Most OCaml projects use [`opam`](https://opam.ocaml.org/) -- the **O**Caml
**Pa**ckage **M**anager -- to manage project dependencies.

`nix` replaces `opam` for your project, since it not only provides OCaml
packages, but it can also provide your entire OCaml toolchain.

# Overview

## An `ocaml` + `nix` starter project

Let's start by using a Nix flake for our project. The easiest way to do this
is to use a template. If you'd like to use a template, feel free to -- you can
do so by running:

```shell
$ nix flake init -t github:sixstring982/flakes#ocaml-dune-project
```

But for now let's go ahead and write one from scratch.
This should help us get a good idea about how flakes work, and what's
necessary to create one.

`flake.nix`:
```nix
{
  # Human-readable project description.
  description = "My OCaml project";

  # Flakes can use other flakes as "inputs".
  inputs = {
    # Pin nixpkgs to the unstable channel. Other channels can
    # be used here instead.
    nixpkgs.url = "github:NixOS/nixpkgs/nixpkgs-unstable";

    # Import flake-utils, which provides useful utilities for
    # writing flakes.
    flake-utils.url = "github:numtide/flake-utils";

    # Import nix-filter, which can help us filter our project
    # files, to avoid unnecessary rebuilds.
    nix-filter.url = "github:numtide/nix-filter";
  };

  # Outputs describe the things that nix is going to set up for
  # our project, and any other things for other flakes to depend on.
  outputs = { self, nixpkgs, flake-utils }:
    let
      # Set up some commonly-used bindings
      legacyPackages = nixpkgs;

      # Version of the OCaml compiler you'd like to use (e.g. 5.1).
      ocamlVersion = "ocamlPackages_5_1";
      # List of available OCaml packages in nixpkgs for our compiler
      ocamlPackages = legacyPackages.ocaml-ng.${ocamlVersion};

      # This should be the name of your OCaml project, as opam sees it.
      projectName = "my_project";
      projectVersion = "0.0.1";

      # Filtered sources (prevents unecessary rebuilds)
      sources = {
        ocaml = nix-filter.lib {
          root = ./.;
          include = [
            ".ocamlformat"
            "dune-project"
            (nix-filter.lib.inDirectory "bin")
            (nix-filter.lib.inDirectory "lib")
            (nix-filter.lib.inDirectory "test")
          ];
        };

        nix = nix-filter.lib {
          root = ./.;
          include = [
            (nix-filter.lib.matchExt "nix")
          ];
        };
      };
    in
    # eachDefaultSystem allows us to easily make this package cross-platform.
    flake-utils.lib.eachDefaultSystem(system: {
      # These packages are built by this project.
      # You can use `nix build` to build these.
      packages = {
        # The `default` package builds when running `nix build` with no
        # arguments.
        default = self.packages.${system}.${projectName};

        # Our package is one which is built with `dune`.
        ${projectName} = ocamlPackages.buildDunePackage {
            pname = ${projectName};
            version = ${projectVersion};
            duneVersion = "3";
            src = sources.ocaml;

            # Here, we list other OCaml libraries that our project
            # depends on.
            buildInputs = [
              # ocamlPackages.opium
            ];

            strictDeps = true;

            preBuild = "dune build ${projectName}.opam";
        };
      };

      # Flakes can have development shells, which can be enterred
      # via `nix develop`.
      devShells = {
        default = legacyPackages.mkShell {
          # Tools that you'll need available via your command line
          # when working on this project
          packages = [
            legacyPackages.fswatch
            legacyPackages.ocamlformat
            ocamlPackages.ocaml-lsp
          ];

          # Tools that are available via building your project
          inputsFrom = [
            self.packages.${system}.${projectName}
          ];
        };
      };
    });
}
```

TIP: If you want your flake to reload every time you use it, set up
[direnv](https://direnv.net/) and write `use flake` into `.envrc`

This approach should work great for setting up a new OCaml project which
depends solely on dependencies found in `nixpkgs`.

## Using `nix` in an existing `ocaml` project

However, if you're wanting to adapt an existing OCaml project to replace
`opam` with `nix`, you'll need to migrate somehow.

### Impure option: `opam-nix`

If you're OK with an impure option -- i.e. still using `opam` to manage your
OCaml packages, but getting the other benefits from `nix` -- you could use
`opam-nix`.

[`opam-nix`](https://www.tweag.io/blog/2023-02-16-opam-nix/) is a flake
developed by `tweag`, which fakes `opam` to some degree, allowing you to easily
bootstrap an existing `opam` project with `nix`.

You'll likely run into problems using this with a larger codebase, however.
it may be better to use a pure option if this ends up happening to you.

### Pure options

#### Happy option: Use `nixpkgs` as-is

If all of your dependencies are already in `nixpkgs`, it's fairly
straightforward to migrate your project to use `nix` in a pure fashion -- simply
use the flake above, and fill out the `buildInputs` section with all of your
build inputs.

Note that the versions of these inputs may change depending on the ones that
are in `nixpkgs` -- as seen later on in this article, you'll need to use
overlays in order to override these versions if you need to do so.

#### Unhappy path

If you need to depend on other packages which are not already in `nixpkgs`, you
have a few options:

##### Contribute to `nixpkgs`

If you need to depend on a package which could be added to `nixpkgs`, you could
contribute to `nixpkgs` in order to add it. Keep in mind that this may be a lot
of work if this package is not compatible with other packages in `nixpkgs` --
you may need to upgrade other packages at this time as well.

See [this pull request](https://github.com/NixOS/nixpkgs/pull/276114) as an
example of adding an OCaml package which was not already available.

##### Using overlays

If a package that you need to depend on is going to be tricky to get into
`nixpkgs`, you couuld override the `nixpkgs` definition in your flake with
an _overlay_.

Here's what you would need to change in the above flake in order to add a new
package -- in this case, upgrading `riot` to `0.0.7`:


```nix
# Here, we tell `nix` that we want to overlay `nixpkgs` with some overrides.
# We'll need to use functional updates for the entire `ocaml-ng` definition,
# and any sub-definitions that need updating.
#
# This pattern can be used to override any number of nix packages, OCaml or
# otherwise.
legacyPackages = import nixpkgs {
  inherit system; overlays = [
  (final: prev: {
    ocaml-ng = prev.ocaml-ng // {
      ${ocamlVersion} = prev.ocaml-ng.${ocamlVersion} // {
        riot = prev.ocaml-ng.${ocamlVersion}.riot.overrideAttrs (_: {
          version = "0.0.7";

          src = legacyPackages.fetchFromGitHub {
            owner = "leostera";
            repo = "riot";
            rev = "7dc78950b5dc172aef74bacee977abd2011005d2";
            sha256 = "sha256-YiUok7cwczyl8ee3LkNiWU/O+hWQdqTGRFVmJOLMpsw=";
          };

          propagatedBuildInputs = with final.ocaml-ng.${ocamlVersion}; [
            bigstringaf
            cstruct
            poll
            ptime
            telemetry
            uri
          ];
        });
      };
    };
  })
];
};
```
