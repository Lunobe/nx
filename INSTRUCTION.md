### Instruction:

* in `flake.nix`:
  ```nix
  inputs.nx = {
    url = "github:Lunobe/nx";
    inputs.nixpkgs.follows = "nixpkgs";
  };

  inputs.nix-index-database.url = "github:nix-community/nix-index-database";
  inputs.nix-index-database.inputs.nixpkgs.follows = "nixpkgs";

  specialArgs = { repoDir = "/etc/nixos"; hostName = "myhost"; };

  nixosConfigurations.myhost = nixpkgs.lib.nixosSystem {
    inherit specialArgs;
    modules = [
      ./configuration.nix
      home-manager.nixosModules.default
      {
        home-manager.useGlobalPkgs = true;
        home-manager.useUserPackages = true;
        home-manager.users.youruser = ./home.nix;
        home-manager.sharedModules = [
          inputs.nix-index-database.homeModules.default
          inputs.nx.homeManagerModules.default
        ];
        home-manager.extraSpecialArgs = specialArgs;
      }
    ];
  };
  ```
  `hostName` has to match what you rebuild with (`nixos-rebuild switch --flake .#myhost`) — a couple nx subcommands (`nx install`, `nx search option/meta`) eval against `nixosConfigurations.${hostName}` directly.

  `repoDir` and `hostName` reach the nx module via `home-manager.extraSpecialArgs` — no import list to edit in `home.nix`, nx is wired in entirely through `sharedModules`.

* in `configuration.nix`, make sure flakes are enabled:
  ```nix
  nix.settings.experimental-features = [ "nix-command" "flakes" ];
  ```

* in `home.nix`, make sure these are enabled (nx's fish functions assume them):
  ```nix
  programs.fish.enable = true;
  programs.git.enable = true;
  programs.nix-index.enable = true;
  programs.nix-index-database.comma.enable = true;
  ```

* `nx list/install/uninstall/deploy` expect a `modules/home/packages.nix` shaped like this (imported into `home.nix`):
  ```nix
  { pkgs, ... }: {
    home.packages = with pkgs; [
      somepackage
    ];
  }
  ```

* `nx secret` needs agenix set up, with an identity at `/var/lib/agenix-bootstrap/identity.txt` and a `secrets/secrets.nix`. Don't care about secrets management? Just delete the `secret` case in `dispatcher.nix` and the `secret` half of `cmd-secret-search-config.nix` — `search`/`config` in that same file work fine without agenix.

* alejandra doesn't need installing separately, nx pulls it straight from nixpkgs.

* rebuild and switch:
  ```
  sudo nixos-rebuild switch --flake /etc/nixos#myhost
  ```

* to pick up changes to nx itself later: `nix flake update nx` (or update every input with `nix flake update`), then rebuild.

`nx --help` from there for the full command list. Enjoy!
