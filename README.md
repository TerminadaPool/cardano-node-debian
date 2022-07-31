# Packaging cardano-node as a debian package
This document outlines one way to compile cardano-node into a debian package.  This provides a way to easily install cardano-node onto any stable debian system and have the package management system take care of everything.

This is not an official debian package and it doesn't do things the official debian way.  In particular, it is not possible to use the official debian stable version of haskell to compile the cardano-node.  Furthermore, this deb package will not deliver the proper copyright documents or any manuals.

The deb package produced will install things in the following standard locations:
* Executables (cardano-node, cardano-cli, plus some utilities) get placed in /usr/bin/
* Configuration files go in /etc/cardano/
* Cardano blockchain data is stored in /var/lib/cardano/
* systemd service file is placed in /lib/systemd/system/

# How to make your own cardano-node deb package
## Setup your build environment
Install build dependencies and some extra requirments  
(as root)  
```
apt install build-essential fakeroot devscripts debhelper git curl automake build-essential pkg-config libffi-dev libgmp-dev libssl-dev libtinfo-dev libsystemd-dev zlib1g-dev make g++ tmux jq wget libncursesw5 libtool autoconf llvm llvm-dev libnuma-dev libncurses-dev libncurses5
```

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
rm -rf ${HOME}/src/cardano-node; \
mkdir -p ${HOME}/src/cardano-node; \
cd ${HOME}/src/cardano-node/; \
latest_version=$(curl -s https://api.github.com/repos/input-output-hk/cardano-node/releases/latest | jq -r .tag_name); \
git clone https://github.com/input-output-hk/cardano-node.git cardano-node-${latest_version}; \
cd cardano-node-${latest_version}; \
git fetch --all --recurse-submodules --tags; \
git checkout tags/${latest_version}; \
# Begin Fix: Version 1.34.1 fix so cardano-cli works when in P2P mode; \
sed -i 's/tag: 4fac197b6f0d2ff60dc3486c593b68dc00969fbf/tag: 48ff9f3a9876713e87dc302e567f5747f21ad720/g' cabal.project; \
# End Fix; \
cd ..; \
tar cvzf cardano-node_${latest_version}.orig.tar.gz cardano-node-${latest_version}; \
cd cardano-node-${latest_version}; \
unset latest_version; \
git clone https://github.com/TerminadaPool/cardano-node-debian.git debian; \
debuild --prepend-path "$HOME/.cabal/bin:$HOME/.ghcup/bin" -us -uc;
```

Your deb will be produced in the parent directory: ~/src/cardano-node/  
And named something like: cardano-node_1.34.1-2_amd64.deb

****
****

### Update your copy of cardano-node-debian repository for new upstream version
If there is a new version of cardano-node and this repository has not been updated to use it then you can get an error like:  
> This package has a Debian revision number but there does not seem to be
> an appropriate original tar file or .orig directory in the parent directory;
> (expected one of cardano-node_1.34.1.orig.tar.gz, cardano-node_1.34.1.orig.tar.bz2, cardano-node_1.34.1.orig.tar.lzma, cardano-node_1.34.1.orig.tar.xz or cardano-node-1.30.1.orig)
> continue anyway? (y/n)

Answer this with 'n'.

You can do the following to update your copy of the repository to the latest version:
```
latest_version="$(curl -s https://api.github.com/repos/input-output-hk/cardano-node/releases/latest | jq -r .tag_name)"; \
cd "${HOME}/src/cardano-node/cardano-node-${latest_version}"/; \
export EMAIL='builder@localhost'; dch --newversion "${latest_version}-1" --package cardano-node;
```
Then build the deb:
```
cd "${HOME}/src/cardano-node/cardano-node-${latest_version}"/; \
unset latest_version; \
debuild --prepend-path "$HOME/.cabal/bin:$HOME/.ghcup/bin" -us -uc;
```
