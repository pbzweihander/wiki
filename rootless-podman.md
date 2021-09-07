# TL;DR rootless podman configuration

```
sudo touch /etc/sub{u,g}id
sudo usermod --add-subuids 100000-165536 --add-subgids 100000-165536 $(whoami)
```
