#### Proposal: Chart Packaging System Improvements

###### Versioning System

Helm charts are uniquely identified by two properties - their name and version. I believe the Helm community would benefit from adopting a Maven-like artifact naming convention (see below), where `group-id` specifies the project and `artifact-id` is the chart name; `<classifier-id>` is optional and can be left unimplemented for the future use:
```
<group-id>:<artifact-id>:<classifier-id>:<version>
```

Consider the following cases:
1) Uniquely identifying a chart with different configurations to be used for two different projects:
```
com.project1:kafka:1.2.3
com.project2:kafka:4.5.6
```

These charts can then be stored in a local helm repository without name clash.

2) More flexible chart naming. Extended charts could have the same names as the original chart:

```
com.helm.repository.project:kafka:1.0.0
```

###### Decoupling Chart from Repository; Caching

The motivation to separate chart dependency and repository management.

1) Most of the charts I use originate only from two repositories: stable and staging. Hence the repetition.
2) If the dependency chart is moves to another repository, repo domain name changes or the repo is simply experiencing a downtime, then the requirements document has to be updated. And this would have to be done for nested dependencies as well.
3) Explicit file system path usage should be discouraged  - it enforces rigid project structures that are difficult to maintain.
4) Repositories should not be specified inside the charts because they may move.
Dead links would slow down and complicate the process of dependency resolution.

The requirements.yaml file would contain only the list of dependencies:
```
dependencies:
- com.project1:kafka:1.2.3
- com.project2:kafka:4.5.6
```

Helm would firstly look for the dependencies in a local repository located in `~/.helm/repository`. The local repository is just a directory structure. For example, one of the kafka chart examples mentioned earlier would be stored in:
```
~/.helm/repository/com/project1/kafka/1.2.3/
```
The directory would contain:
```
 kafka-1.2.3.tar.gz - chart archive
 kafka-1.2.3.tar.gz.sha1 - checksum for our chart
 kafka-1.2.3.pom - chart descriptor, e.g. what other dependencies it might require
 kafka-1.2.3.pom.sha1 - chart descriptor check sum
```

If the dependency in question is not found locally, Helm would look for it in remote repositories. The list of external repositories would be defined either centrally or passed as a parameter:

```
helm package -d <chart_out> <chart_input> -cp <chart_path>
```
Where `<chart_path>` is a comma separated list of repository URLs and/or directories containing charts.

This would make it possible to manage dependency list with a build automation tool such as Gradle or Maven.
Some use cases:
1) Declare chart dependencies in maven/gradle files along other dependencies
2) Declare project cross-dependencies that would automatically trigger Helm chart builds
3) Greater integration with code, especially in the context of microservices. Each microservice comes with its own infrastructure, therefore it makes sense to have a project that holds both helm charts and, e.g. Java code:

```
project/
 src/
   helm/ - charts here
   main/ - java stuff
 ```

Such project structure then allows sharing common properties between the infrastructure and the code, e.g. database address and port. Linking infrastructure to code can finally be fully automated!

###### Better `Staging` and `Stable` Chart Separation

At the moment there is no way to tell wether a downloaded chart's status is `stable` or `staging`.
Staging chart names could be appended with `-STAGING`.
