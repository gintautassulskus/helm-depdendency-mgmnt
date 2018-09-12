## Proposal: Chart Packaging System Improvements

#### Versioning System

At the moment, Helm charts are uniquely identified by two properties - their name and version. The Helm community could potentially benefit from adopting a Maven-like convention to name Helm charts. The naming convention is comprised of four elements (see below), where `group-id` specifies the project and `artifact-id` is the chart name; `<classifier-id>` is optional and can be left unimplemented for the future use:

```
<group-id>:<artifact-id>:<classifier-id>:<version>
```

Consider the following use-cases:
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

This is useful if the internal chart retains key elements from the original chart but differs in configuration that is required accross one or more projects.

#### Decoupling Chart from Repository; Caching

Arguments to separate chart dependency and repository management:

1) Most of the charts originate only from two repositories: stable and staging. Hence the repetition.
2) Repositories should not be specified inside the charts:
   2.1) repositories may move or experience downtime. E.g. dead links would slow down and complicate the process of dependency resolution.
   2.2) Changes to the repository must be reflected in the chart and its nested dependencies.
3) Explicit file system path usage should be discouraged  - it enforces rigid project structures that are difficult to maintain and reuse, especially across multi-project setups.

The requirements.yaml file would contain only the list of dependencies:
```
dependencies:
- id: com.project1:kafka:1.2.3
- id: com.project2:kafka:4.5.6
  alias: still_works
  enabled: true
```

Helm would firstly look for the dependencies in a local repository located in `~/.helm/repository`. For example, one of the kafka chart examples mentioned earlier would be stored in:
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
helm package -d <chart_out> <chart_input> -cr <chart_repository>
```
Where `<chart_repository>` is a comma separated list of repository URLs and/or directories containing charts.

This would make it possible to manage dependency list with a build automation tool such as Gradle or Maven.
Some use cases:
1) Declare chart dependencies in maven/gradle files along other dependencies, e.g. gradle depednecy:

```
helmRepositories {
    repository "stable"
    repository "companyRepo"
}
```

3) The declared chart dependencies then could automatically trigger Helm chart builds in other projects.

```
helmRepositories {
    repository project(":some:grdle:sub-project")
}
```

4) Improve integration with code, especially in the context of microservices. Each microservice comes with its own infrastructure, therefore it makes sense to have a project that holds both helm charts and, e.g. Java code:

```
project/
 src/
   helm/ - helm charts here
   main/ - java stuff
 ```

Such project structure then allows sharing common properties between the infrastructure and the code, e.g. database address and port. Linking infrastructure to code can finally be fully automated!

#### Better `Staging` and `Stable` Chart Separation

At the moment there is no way to tell wether a downloaded chart's status is `stable` or `staging`.
Staging chart versions could be appended with `-STAGING`.
