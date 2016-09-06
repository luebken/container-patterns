# 3. Self-Describing

A good module has an explicit, or as wikipedia says [well defined](https://en.wikipedia.org/wiki/Modular_programming#Key_aspects) interface. So with all the ways declaring an API for our container we also need a way to document this. Unfortunatel, as this is not always possible within the declaration; we want to expand on the definition and use labels to document our API.

## Document With Labels

We like to propose to document the API with labels, as all container engines have the means of doing so.

Docker has the ability to add arbitrary metadata to images via [labels](https://docs.docker.com/engine/userguide/labels-custom-metadata/) which can be even [JSON](https://docs.docker.com/engine/userguide/labels-custom-metadata/#store-structured-data-in-labels).

Gareth Rushgrove has been evangelising the idea of creating standards on this data and creating ["shipping manifests"](https://speakerdeck.com/garethr/shipping-manifests-bill-of-lading-and-docker-metadata-and-container
). He has even written a tool that validates the schema of this data: [docker-label-inspector](https://github.com/garethr/docker-label-inspector). See his Dockercon [presentation](https://www.youtube.com/watch?v=j4SZ1qoR8Hs) and [slides](https://speakerdeck.com/garethr/shipping-manifests-bill-of-lading-and-docker-metadata-and-container) for details.

From Red Hat there is an initiative on [standardizing certain labels](https://github.com/projectatomic/ContainerApplicationGenericLabels).

Example:

As a simple example we want to document that our container expects a certain environment variable. Here a suggestion using the prefix `api.ENV`:

```
LABEL api.ENV.OPENWEATHERMAP_APIKEY="" \
      api.ENV.OPENWEATHERMAP_APIKEY.description="Access key for OpenWeatherMap. See http://openweathermap.org/appid for details." \
      api.ENV.OPENWEATHERMAP_APIKEY.mandatory="false"
```
See https://github.com/luebken/currentweather/blob/master/Dockerfile

We can than use tools like `docker inspect` to read these labels:

```
$ docker inspect -f "{{json .Config.Labels }}" <container image>
```

## View the API contract of a module container

If you take this a step further and agree on some standard labels and schemas you could write tools that introspect the API contract of a container. e.g.


```
$ container-api luebken/currentweather-nodejs
 -----------------------------------------------------------
| Image:  luebken/currentweather-nodejs
|-----------------------------------------------------------
| Author:   Matthias Luebken, matthias@luebken.com
| Size:     158 MB
| Created:  2016-03-01 15:59
|-----------------------------------------------------------
| Container API:
| * Mandatory ENVs to configure:
|   - ENV:           OPENWEATHERMAP_APIKEY
|   - Description:   APIKEY to access the OpenWeatherMap. Get one at http://openweathermap.org/appid
|   - Mandatory:     true
| * Optional ENVs to configure:
|     - < empty >
| * Available ports:
|     -  1337/tcp {}
| * Volumes:
 -----------------------------------------------------------
```

For more info see: https://github.com/luebken/container-api
