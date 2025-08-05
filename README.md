# Build cardano-node and cardano-cli as deb packages
This document outlines _one_ way to build the cardano-node software into Debian packages.  Building the software as deb packages makes it easier to install on any Debian or Ubuntu system and have the package management system take care of everything.

These are not official deb packages and this document does not do things the official Debian/Ubuntu way.  Furthermore, these deb packages do not deliver the proper copyright documents or any manuals.

These deb packages will install the software in the following standard locations:
- Libraries are placed in /usr/lib/
- Executables are placed in /usr/bin/
- Configuration files are placed in /etc/cardano/
- Cardano blockchain database is stored in /var/lib/cardano
- Systemd service file is placed in /lib/systemd/system/

## Check for any recent changes to install instructions
- See wiki article: [Getting cardano-node: Building from source](<https://developers.cardano.org/docs/operate-a-stake-pool/node-operations/installing-cardano-node>)
- Review version history of the corresponding markdown file: https://github.com/cardano-foundation/developer-portal/blob/staging/docs/operate-a-stake-pool/node-operations/installing-cardano-node.md
  - Click "History" in top right, then click any recent commits to see what changes have been made.  If no significant changes then continue.

## Setup your build environment
I usually create a virtual machine for building software and do everything on it.  This allows me to keep everything separated from my desktop PC.

I also create a separate user ('builder') for building.  This is advisable, even if you choose to build on your desktop PC, so that the installation of any local compilers, or any specific configurations, won't interfere with anything in your normal user account.
```
sudo adduser --gecos '' --disabled-password builder
```

Install build dependencies and some extra requirements
```
sudo apt install build-essential fakeroot devscripts debhelper autoconf-archive d-shlibs pkg-kde-tools git curl automake pkg-config libffi-dev libgmp-dev libssl-dev libtinfo-dev libsystemd-dev zlib1g-dev make jq libncursesw5 libtool autoconf libncurses-dev libncurses5 libnuma-dev liblmdb-dev
```

Optionally install llvm and set cc and c++ to use clang
```
sudo apt install llvm clang; \
sudo update-alternatives --install /usr/bin/cc cc /usr/bin/clang 100; \
sudo update-alternatives --install /usr/bin/c++ c++ /usr/bin/clang++ 100
```
Ubuntu users might need to skip the previous optional step if their Ubuntu version of clang doesn't support optimization flag '-ffat-lto-objects'.  Revert the previous changes with:
```
sudo apt purge llvm clang; \
sudo apt autoremove;
```

Ubuntu users may need to manually install libssl1.1 if it is not on their system:
```
wget http://security.ubuntu.com/ubuntu/pool/main/o/openssl/libssl1.1_1.1.1f-1ubuntu2.22_amd64.deb; \
sudo dpkg -i libssl1.1_1.1.1f-1ubuntu2.22_amd64.deb
```

## Install the IOG versions of specific dependencies
Make sure you have built all these to deb files first:
- See: [Build IOG specified version of libsodium as a deb package](https://github.com/TerminadaPool/libsodium-iog-debian)
- See: [Build IOG specified version of libsecp256k1 as a deb package](https://github.com/TerminadaPool/libsecp256k1-iog-debian)
- See: [Build IOG specified version of libblst as a deb package](https://github.com/TerminadaPool/libblst-iog-debian)

If you have put these dependency debs into a local repository then you can simply use apt to install them:
```
apt install libsodium-dev-iog libsecp256k1-dev-iog libblst-iog
```
Otherwise use dpkg to install them:
```
dpkg -i libsodium23-iog_1.0.18_amd64.deb libsodium-dev-iog_1.0.18_amd64.deb; \
dpkg -i libsecp256k1-2-iog_0.3.2_amd64.deb libsecp256k1-dev-iog_0.3.2_amd64.deb; \
dpkg -i libblst-iog_0.3.14_amd64.deb;
```

Switch to your builder account
```
sudo su - builder
```

The rest of this document assumes you are using your 'builder' account:
>builder@build:~$

## Install GHC and Cabal
```
curl --proto '=https' --tlsv1.2 -sSf https://get-ghcup.haskell.org | sh
```
Select 'N' to everything when asked. We will set up path manually.

The following is an example transcript:
>ghcup installs only into the following directory, which can be removed anytime:  
>  /home/builder/.ghcup  
>  
>Press ENTER to proceed or ctrl-c to abort.  
>  
>Do you want ghcup to automatically add the required PATH variable to "/root/.bashrc"?  
>[P] Yes, prepend  [A] Yes, append  [N] No  [?] Help (default is "P").  
>N  
>  
>Do you want to install haskell-language-server (HLS)?  
>[Y] Yes  [N] No  [?] Help (default is "N").  
>N  
>  
>Do you want to install stack?  
>[Y] Yes  [N] No  [?] Help (default is "N").  
>N  
>  
>System requirements  
>  Please install the following distro packages: build-essential curl libffi-dev libffi7 libgmp-dev libgmp10 libncurses-dev libncurses5 libtinfo5  
>  
>Press ENTER to proceed or ctrl-c to abort.  

These 'system requirements' packages should already have been installed above so just press enter.

## Setup $PATH in ~/.bashrc to include cabal and ghcup bin directories
```
echo '[[ ":$PATH:" != *":$HOME/.ghcup/bin:"* ]] && PATH="$HOME/.ghcup/bin:$PATH"' >> ${HOME}/.bashrc; \
echo '[[ ":$PATH:" != *":$HOME/.cabal/bin:"* ]] && PATH="$HOME/.cabal/bin:$PATH"' >> ${HOME}/.bashrc; \
echo 'export PATH' >> ${HOME}/.bashrc; \
source ${HOME}/.bashrc;
```
Check versions
```
ghcup --version; \
cabal --version
```

Use ```ghcup tui``` to view which version of ghc and cabal are available.  Then choose which versions to install and set as default.

### Optionally force rebuilding of all Haskell dependencies
Previously built dependencies are stored in ${HOME}/.cabal/store/

To force the haskell compiler to rebuild all the dependencies from scratch, remove this directory and all its subdirectories with:
```
rm -rf ${HOME}/.cabal/store
```

## Check the latest cardano-node release version
See: https://github.com/intersectMBO/cardano-node

Or from the command line with:
```
CARDANO_NODE_VERSION="$(curl -s https://api.github.com/repos/intersectMBO/cardano-node/releases/latest | jq -r .tag_name)"; \
echo "Latest CARDANO_NODE_VERSION is: ${CARDANO_NODE_VERSION}";
```
You can also check available tags at: https://github.com/IntersectMBO/cardano-node/tags

The following sequence of commands will remove and recreate the "${HOME}/src/cardano-node" directory.  Familiarise yourself with the following commands before running them.  You can simply copy and paste the entire list of commands below into a bash terminal to run them in sequence.

## Build cardano-cli, cardano-node, cardano-submit-api deb packages
Nb. Ensure ${CARDANO_NODE_VERSION}" variable has been set either manually or by using the previous command.
```
deb_build_instructions_repo='https://github.com/TerminadaPool/cardano-node-debian.git'; \

cardano_node_repo='https://github.com/IntersectMBO/cardano-node.git'; \

package='cardano-node'
basedir="${HOME}/src/${package}"; \

mkdir -p "${basedir}"; \
cd "${basedir}"; \
rm -rf "${package}-${CARDANO_NODE_VERSION}"; \

git clone "${cardano_node_repo}" "cardano-node-${CARDANO_NODE_VERSION}"; \
cd "cardano-node-${CARDANO_NODE_VERSION}"; \
git fetch --all --recurse-submodules --tags; \
git checkout "tags/${CARDANO_NODE_VERSION}"; \

git clone "${deb_build_instructions_repo}" debian; \

debuild --prepend-path "$HOME/.cabal/bin:$HOME/.ghcup/bin" -us -uc -b; \

unset deb_build_instructions_repo cardano_node_repo package basedir;
```

Your debs will be produced in the parent directory: "${HOME}/src/cardano-node/".  They will be named something like:

> cardano-cli_10.1.4_amd64.deb
> cardano-node_10.1.4_amd64.deb
> cardano-submit-api_10.1.4_amd64.deb

Put these files in your own local debian repository and install them on every machine required using apt.

## Notes:
- The ghc and cabal versions used for compilation are configured in [rules](https://github.com/TerminadaPool/cardano-node-debian/blob/master/rules) file.
- cabal.project.local is also configured in [rules](https://github.com/TerminadaPool/cardano-node-debian/blob/master/rules) file.
- Both cardano-node and cardano-cli packages are configured to depend upon the IOG specified versions of libsodium (libsodium23-iog) and libsecp256k1 (libsecp256k1-2-iog).  See: [control](https://github.com/TerminadaPool/cardano-node-debian/blob/master/control) file.
- If cardano-node deb package is purged from the system then unaltered configuration files will be removed as well as /etc/cardano directory if it is subsequently empty.  However, the 'cardano' user will be left on the system.  See: [cardano-node.postrm](https://github.com/TerminadaPool/cardano-node-debian/blob/master/cardano-node.postrm).
- There are example configuration files installed to '/etc/cardano/{mainnet,preprod,preview}/pooltool-api-key'. The script [cn-monitor-block-delay](https://github.com/TerminadaPool/cardano-node-debian/blob/master/bin/cn-monitor-block-delay) can use these files to obtain the "Pooltool Api Key" without obtaining such information from the command line where it is visible to tools like 'ps'.
- The systemd [cardano-node service](https://github.com/TerminadaPool/cardano-node-debian/blob/master/cardano-node.service) file will optionally configure its $SERVICE_PORT from /etc/cardano/{mainnet,preprod,preview}/service-port if such file contains a SERVICE_PORT=VALUE line (Eg: SERVICE_PORT=3001).
- There are some additional simple tools in the [bin subdirectory](https://github.com/TerminadaPool/cardano-node-debian/blob/master/bin) which get installed to /usr/bin/ on the system.
- Expect to see lintian errors like the following at the end of the build process:
>E: cardano-cli: embedded-library libyaml [usr/bin/cardano-cli]  
>E: cardano-cli: embedded-library libyaml [usr/bin/cardano-submit-api]  
>E: cardano-node: embedded-library libyaml [usr/bin/cardano-node]  
>E: cardano-cli-dbgsym: unpack-message-for-deb-data Output from 'readelf --all --wide usr/lib/debug/.build-id/38/6c97fd1b0d66cdf82ce266e02753ec53350991.debug' is not valid UTF-8  
>E: cardano-cli-dbgsym: unpack-message-for-deb-data Output from 'readelf --all --wide usr/lib/debug/.build-id/74/a3012a640a238f66de1826f93f0a77b7dcb007.debug' is not valid UTF-8  
>E: cardano-cli-dbgsym: unpack-message-for-deb-data Output from 'readelf --all --wide usr/lib/debug/.build-id/fa/db822a91370d4704149b863a99d11587429f8c.debug' is not valid UTF-8  
>E: cardano-node-dbgsym: unpack-message-for-deb-data Output from 'readelf --all --wide usr/lib/debug/.build-id/5b/47687ea24048b81acfe15296947430f48b617e.debug' is not valid UTF-8  
>W: cardano-cli-dbgsym: debug-file-with-no-debug-symbols [usr/lib/debug/.build-id/38/6c97fd1b0d66cdf82ce266e02753ec53350991.debug]  
>W: cardano-cli-dbgsym: debug-file-with-no-debug-symbols [usr/lib/debug/.build-id/74/a3012a640a238f66de1826f93f0a77b7dcb007.debug]  
>W: cardano-cli-dbgsym: debug-file-with-no-debug-symbols [usr/lib/debug/.build-id/fa/db822a91370d4704149b863a99d11587429f8c.debug]  
>W: cardano-node-dbgsym: debug-file-with-no-debug-symbols [usr/lib/debug/.build-id/5b/47687ea24048b81acfe15296947430f48b617e.debug]  
>W: cardano-cli: hardening-no-pie [usr/bin/bech32]  
>W: cardano-cli: hardening-no-pie [usr/bin/cardano-cli]  
>W: cardano-cli: hardening-no-pie [usr/bin/cardano-submit-api]  
>W: cardano-node: hardening-no-pie [usr/bin/cardano-node]  
>W: cardano-cli: no-manual-page [usr/bin/bech32]  
>W: cardano-cli: no-manual-page [usr/bin/cardano-cli]  
>W: cardano-cli: no-manual-page [usr/bin/cardano-submit-api]  
>W: cardano-cli: no-manual-page [usr/bin/cn-date-to-slot]  
>W: cardano-cli: no-manual-page [usr/bin/cn-slot-to-date]  
>W: cardano-node: no-manual-page [usr/bin/cardano-node]  
>W: cardano-node: no-manual-page [usr/bin/cn-leaderlog]  
>W: cardano-node: no-manual-page [usr/bin/cn-monitor-block-delay]  
>W: cardano-node: no-manual-page [usr/bin/cn-update-topology]  

debuild switches:
- -b: build binary packages only
- -uc: No signing of chainlog
- -us: No signing of source code


### References
- [Compiling cardano-node](https://github.com/input-output-hk/cardano-node-wiki/blob/main/docs/getting-started/install.md)
- [Configuration files](https://book.play.dev.cardano.org/environments.html)
