# Packaging cardano-node as a debian package
This document outlines one way to compile cardano-node into a deb package.  This provides a way to easily install cardano-node onto any Debian or Ubuntu system and have the package management system take care of everything.

This is not an official deb package and it doesn't do things the official Debian or Ubuntu way.  In particular, it is not possible to use the official Debian/Ubuntu version of haskell to compile the cardano-node.  Furthermore, this deb package will not deliver the proper copyright documents or any manuals.

The deb package produced will install things in the following standard locations:
* Executables (cardano-node, cardano-cli, plus some utilities) get placed in /usr/bin/
* Configuration files go in /etc/cardano/
* Cardano blockchain data is stored in /var/lib/cardano/
* systemd service file is placed in /lib/systemd/system/

# Installing pre-built deb packages
If all you want is to install a pre-built cardano-node deb package then these are available for arm64 and amd64 architectures at:
https://TerminadaPool.github.io/deb

To install the latest cardano-node binary deb package do the following:
```
wget -O- https://TerminadaPool.github.io/deb/KEY.gpg | gpg --dearmor | sudo tee /usr/share/keyrings/terminada.io.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/terminada.io.gpg] https://TerminadaPool.github.io/deb ./" | sudo tee /etc/apt/sources.list.d/terminada.io.list
sudo apt update
sudo apt install cardano-node
```

If you install this binary deb package, files will be placed on your system as noted above.  
If you later do an "apt purge cardano-node" then all these installed files will be removed.  

Note: A fully synced Cardano blockchain will currently require:
* Around 100G of disk space
* 16G RAM and 16G Swap
* Dual core 2GHz processor

# How to build your own cardano-node deb package
## Setup your build environment
Install build dependencies and some extra requirments  
(as root)  
```
apt install build-essential fakeroot devscripts debhelper git curl automake build-essential pkg-config libffi-dev libgmp-dev libssl-dev libtinfo-dev libsystemd-dev zlib1g-dev make g++ jq libncursesw5 libtool autoconf llvm llvm-dev libnuma-dev libncurses-dev libncurses5
```

Install some extra libraries required by cardano-node
```
apt install libsodium23 libsodium-dev
```

You will also require libsecp256k1 development library  
See: https://github.com/TerminadaPool/libsecp256k1-iog-debian for how to build it as a deb package.  
Once you have built the libsecp256k1-iog library, install it too.

## Create a separate user ('builder') for building
This is advisable so that the installation of GHC and its configuration won't interfere with anything in your normal user account.  
(as root)  
```
adduser --gecos '' --disabled-password builder
```
Nb. Password logins disabled.  
Now you can use 'builder' user to do everything.  

#### Switch user to your builder account  
(as root)  
```
su - builder
```

****
## The rest of this document assumes you are using your 'builder' account.
The following commands are all run from this 'builder' user account.

Where there is a number of commands, each line ends with a backslash continuation sequence.  The sequence of commands can then be simply copied and pasted into a terminal to save typing.
****

## Install GHC and Cabal
```
curl --proto '=https' --tlsv1.2 -sSf https://get-ghcup.haskell.org | sh
```
Select 'N' to everything when asked. We will set up path manually.  

The following is an example transcript:  
> ghcup installs only into the following directory,  
> which can be removed anytime:  
>   /home/builder/.ghcup  
> 
> Press ENTER to proceed or ctrl-c to abort.  
> 
> Do you want ghcup to automatically add the required PATH variable to "/root/.bashrc"?  
> [P] Yes, prepend  [A] Yes, append  [N] No  [?] Help (default is "P").  
> N
>
> Do you want to install haskell-language-server (HLS)?  
> [Y] Yes  [N] No  [?] Help (default is "N").  
> N
>
> Do you want to install stack?  
> [Y] Yes  [N] No  [?] Help (default is "N").  
> N
> 
> System requirements  
>   Please install the following distro packages: build-essential curl libffi-dev libffi7 libgmp-dev libgmp10 libncurses-dev libncurses5 libtinfo5  
> 
> Press ENTER to proceed or ctrl-c to abort.  

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

# Check the latest version of cardano-node
```
curl -s https://api.github.com/repos/input-output-hk/cardano-node/releases/latest | jq -r .tag_name
```
Compare this with the latest_version variable in the build commands below.

# Build the cardano-node deb
This sequence of commands will update your GHC compiler and dependencies as well as completely remove and recreate the '~/src/cardano-node' directory.  
Familiarise yourself with the following commands first.  
Then you can simply copy and paste them all into a terminal and press enter.  
```
ghcup upgrade; \
ghcup install ghc '8.10.7'; \
ghcup set ghc '8.10.7'; \
cabal update; \
echo; \
echo "--------------------------------------------------"; \
echo "Build environment:"; \
ghcup --version; \
ghc --version; \
cabal --version; \
echo "--------------------------------------------------"; \
echo; \
sleep 2; \

echo "Removing directory ${HOME}/src/cardano-node"; \
rm -rf ${HOME}/src/cardano-node; \
echo "Creating directory ${HOME}/src/cardano-node"; \
mkdir -p ${HOME}/src/cardano-node; \
cd ${HOME}/src/cardano-node/; \

latest_version='1.35.4'; \

git clone https://github.com/input-output-hk/cardano-node.git cardano-node-${latest_version}; \
cd cardano-node-${latest_version}; \
git fetch --all --recurse-submodules --tags; \
git checkout tags/${latest_version}; \
cd ..; \
tar cvzf cardano-node_${latest_version}.orig.tar.gz cardano-node-${latest_version}; \
cd cardano-node-${latest_version}; \
unset latest_version; \
git clone https://github.com/TerminadaPool/cardano-node-debian.git debian; \
debuild --prepend-path "$HOME/.cabal/bin:$HOME/.ghcup/bin" -us -uc;
```

Your deb will be produced in the parent directory: ~/src/cardano-node/  
And named something like: cardano-node_1.35.4-1_amd64.deb

****
****
