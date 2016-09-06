# 6. Self-Contained
The container should only rely on the Linux kernel. All dependencies should be added at build time. E.g. In the Java world build an Uber-Jar which includes a webserver.

Sometimes developers put application code into a volume to cut the container build time. If you do this, please ensure that you do it only for yourself. And remember to include all application code once the container leaves your computer.

The exception to this rule might sensible data like secretes or god forbid passwords. A detailed writeup about those will follow with [issue 7](https://github.com/luebken/container-patterns/issues/7).

## Configuration
If your container relies on configuration files generate them on the fly. Also ensure that you use sensible defaults for parameters which enables simple zero-config deployments.

There are several tools out there to help generate config files. E.g. [progrium/entrykit](https://github.com/progrium/entrykit#render---template-rendering) or [kelseyhightower/confd](https://github.com/kelseyhightower/confd). But also a simple shell script might do the job e.g.:

```
#!/bin/sh
set -e
datadir=${APP_DATADIR:="/var/lib/data"}
host=${APP_HOST:="127.0.0.1"}
port=${APP_PORT:="3306"}
cat <<EOF > /etc/config.json
{
  "datadir": "${datadir}",
  "host": "${host}",
  "port": "${port}",
}
EOF
mkdir -p ${APP_DATADIR}
exec "/app"
```
Which is blatantly copied from the great blogpost by Kelsey on [12 Fractured Apps](https://medium.com/@kelseyhightower/12-fractured-apps-1080c73d481c#.g3ydhs5zw).


[//]: # (TODO write something about config files http://www.mricho.com/confd-and-docker-separating-config-and-code-for-containers/)
[//]: # (TODO https://twitter.com/kelseyhightower/status/657761769570594816)


#### Further reading on configuration

* Configuration of applications running in containers: https://github.com/joyent/containerbuddy
* A dynamic configuration file generation tool: https://github.com/markround/tiller