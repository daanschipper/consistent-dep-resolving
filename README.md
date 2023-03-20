Example showing consistent dependency version resolving across all configurations for a multi-project build using the version catalog!

The two dependencies `com.fasterxml.jackson.core:jackson-databind:2.8.9` and `io.vertx:vertx-core:3.5.3` lead to a version conflict as `vertx` transitively depends on `jackson-databind:2.9.5`.   
However, Gradle would actually resolve the following dependency versions:
* `jackson-core` version `2.9.5` (brought by `vertx-core`)
* `jackson-databind` version `2.9.5` (by conflict resolution)
* `jackson-annotation` version `2.9.0` (dependency of `jackson-databind:2.9.5)

This leads to the following issues
* Confusion as to which version is actually used, one might expect version `2.8.9` for `jackson-databind` is used, but it is actually `2.9.5`. 
* Different versions can be resolved for the `compileClasspath` and `runtimeClasspath`.

This repository combines several guides into one example to showcase consistent dependency version resolving:
* [Aligning dependency versions](https://docs.gradle.org/current/userguide/dependency_version_alignment.html#version_alignment)
* [Sharing dependency versions between projects](https://docs.gradle.org/current/userguide/platforms.html)
* [Declaring Rich Versions](https://docs.gradle.org/current/userguide/rich_versions.html)
* [Preventing accidental dependency upgrades](https://docs.gradle.org/current/userguide/resolution_strategy_tuning.html)

## Failing on version conflict
By default, Gradle performs optimistic upgrades, meaning that if version 2.8.9 and 2.9.5 of the same library are found in the graph, we resolve to the highest version, 2.9.5. 
By adding `failOnVersionConflict()` with the following snippet Gradle will fail the build to be made aware of the version conflict.

```groovy
configurations.all {
    resolutionStrategy {
        failOnVersionConflict()
    }
}
```

Gradle will then fail with the following error as different versions of `jackson-core` and `jackson-databind` are resolved.
```plaintext
> Conflict(s) found for the following module(s):
  - com.fasterxml.jackson.core:jackson-core between versions 2.9.5 and 2.8.9
  - com.fasterxml.jackson.core:jackson-databind between versions 2.9.5 and 2.8.9
```

## Version catalog
A version catalog is a list of dependencies, represented as dependency coordinates, that a user can pick from when declaring dependencies in a build script. This example uses the TOML format, but you can also choose to define it in the `settings.gradle`, for more information see [Central declaration of dependencies](https://docs.gradle.org/current/userguide/platforms.html#sub:central-declaration-of-dependencies).
Using a version catalog, whether it is a `libs.versions.toml` file or defined in `settings.gradle`, does not guarantee a single source of truth for dependencies: it’s a conventional location where dependencies can be declared. As soon as you start using catalogs, it’s strongly recommended to declare all your dependencies in a catalog and not hardcode group/artifact/version strings in build scripts. Be aware that it may happen that plugins add dependencies, which are dependencies defined outside this file.

### Strict versions and publishing
As explained in [Consequences of using strict versions](https://docs.gradle.org/current/userguide/dependency_downgrade_and_exclude.html#sec:strict-version-consequences),
using a strict version must be carefully considered, in particular by library authors.

Do not declare strict versions like `jackson = { strictly = "2.8.9" }` in the version catalog as this may trigger an error if the consumer disagrees as strict versions are still considered globally during graph resolution.
Use a platform instead to control transitive dependency versions.

## Platform
A platform is meant to influence the dependency resolution graph, for example by adding constraints on transitive dependencies: it’s a solution for structuring a dependency graph and influencing the resolution result.


The following snippet shows how to define strict versions by iterating over all libraries declared in the version catalog: 
```groovy
def versionCatalogName = "libs"
def versionCatalog = extensions.getByType(VersionCatalogsExtension).named(versionCatalogName)

dependencies {
    constraints {
        versionCatalog.getLibraryAliases().each { String alias ->
            versionCatalog.findLibrary(alias).ifPresent(library -> {
                api(library) {
                    version {
                        strictly "${library.get().getVersionConstraint().getRequiredVersion()}"
                    }
                }
            })
        }
    }
}
```

## Making sure resolution is reproducible
The following dependency versions should not be used to make dependency resolving reproducible over time
* dynamic dependency versions are used (version ranges, latest.release, 1.+, ...)
* or changing versions are used (SNAPSHOTs, fixed version with changing contents, ...)

Fail the build with the following snippet if such cases are detected.
```groovy
configurations.all {
    resolutionStrategy {
        failOnNonReproducibleResolution()
    }
}
```