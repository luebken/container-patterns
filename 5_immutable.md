# 5. Immutable

The container image contains the OS, libraries, configurations, files and application code. Once a container image is built it shouldn’t be changed, especially not between different staging environments like dev, qa and production. State should be extracted and changes to the container should be applied by rebuilding the container.

**Best practices on being immutable:**
* Have a [dev / prod parity](http://12factor.net/dev-prod-parity) with the container image
* Extract runtime state in volumes
* Create a final file layout on build
* And don't go into the container and change configuration with the risk of creating a [SnowflakeServer](http://martinfowler.com/bliki/SnowflakeServer.html)
