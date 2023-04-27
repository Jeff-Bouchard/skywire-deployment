# Skywire Deployment

This repository contains subtrees of all the repositories used for skywire deployment

* [skywire](https://github.com/skycoin/skywire)
* [dmsg](https://github.com/skycoin/dmsg)
* [skywire-utilities](https://github.com/skycoin/skywire-utilities)
* [skywire-services](https://github.com/skycoin/skywire-services)
* [skycoin-service-discovery](https://github.com/skycoin/skycoin-service-discovery)

To update subtrees in this repo, first `git checkout` either develop or master.

Then, run the following:

```
find . -maxdepth 1 -mindepth 1 -type d ! -name '.git' -printf '%P\n' | while read -r _repo ; do git subtree pull --prefix=${_repo} https://github.com/skycoin/${_repo}.git $(git branch --show-current) ; done
```
