---
title: Announcing Dash Haskell Flake
---

Announcing [Dash Haskell Flake](https://github.com/VitalBio/dash-haskell-flake/), a repository to
build [Dash](https://kapeli.com/dash) docsets using Nix [Haskell
Flakes](https://flake.parts/options/haskell-flakes).

The existing tools ([1](https://github.com/jfeltz/dash-haskell),
[2](https://github.com/philopon/haddocset)) on Hackage were slightly out of date, so the new approach
uses [Dashing](https://github.com/technosophos/dashing#readme) to generate indexes directly.

```nix
{
  inputs = {
    flake-parts.follows = "vital-nix/flake-parts";
    haskell-flake.url = "github:srid/haskell-flake";
    nixpkgs.url = "nixpkgs/release-23.11";
    dash-haskell.url = "git+ssh://git@github.com/VitalBio/dash-haskell-flake.git?ref=main";
  };

  outputs = inputs:
    inputs.flake-parts.lib.mkFlake {inherit inputs;} {
      imports = [
        inputs.haskell-flake.flakeModule
        inputs.dash-haskell.flakeModule
      ];
      perSystem = {
        config,
        pkgs,
        ...
      }: {
        haskellProjects.default = {
          ...
        };
        dashHaskell = {
          name = "My Project";
          package = "my-project";
          externalUrl = "https://github.com/my-org/my-project";
          version = "1.0";
          mkDocsetUrl = pkgs.writeShellScript "mk-docset-url" ''
            echo "https://hydra.company.domain$(sed 's/nix\/store/nix-store/' <<< $1)"
          '';
          haskellProject = "default";
        };
        packages = rec {
          inherit (config.dashHaskell.outputs.default) haddock docset;
        };
      };
    };
```

Feedback and enhancements welcome at https://github.com/VitalBio/dash-haskell-flake/.
