Proposal: Chart Packaging System Improvements

Allow Helm to use local filesystem as a repository, e.g. ~/.helm/repository instead of having to run a local web-server.
This would be useful for multi-project builds (gradle, maven etc) where:
1) ProjectA packages and uploads ChartA to the local repository
2) ProjectB then declares ChartA dependency. Locating and fetching ChartA from the local repository would be done by Helm.  relying on filesystem paths

###### Versioning system
A chart should not be tied to a repository.
Developers should be concerned only with the unique chart identification only.
At the moment there is no way to create two identically named charts from different developers.

Using Maven-like artifact dependency system charts would be identified using the following format:
```
<group-id>:<artifact-id>:<classifier-id>:<version>
```
Where <classifier-id> is optional and can be left unimplemented for the future use. I will not use the classifier part in further text.

An example Kafka charts created by Apache and SomeDeveloper would then look as follows:
```
org.apache:kafka:1.0.0
org.some-developer:kafka:1.0.0
```

###### Decoupling Chart from Repository and Caching

A repository should not be tied to a chart. Charts can be resolved by their identifier described in the previous section.

This would enable centralised repository list management.
Additional repositories could specified in a chart in a separate section "repositories".

requirements.yaml example:
```
dependencies:
- org.apache:kafka:1.0.0
repositories:
- repository1.org
- repository2.org
```

Chart depdendencies would always be firstly fetced to a local repository that also serves as a cache.
From there, the dependency in question would be copied to the relevant chart.
The next time the same chart is needed, helm would firstly look in the cache and fetch it from there eliminating the need to run a local web-server.

For example, kafka chart would be stored in:
```
~/.helm/repository/org/apache/kafka/1.0.0/
```
The directory would contain:
```
 kafka-1.0.0.tar.gz - chart archive
 kafka-1.0.0.tar.gz.sha1 - checksum for our chart
 kafka-1.0.0.pom - chart descriptor, e.g. what other dependencies it might require
 kafka-1.0.0.pom.sha1 - chart descriptor check sum
```

A generic version of the above would be:

```
 ~/.helm/repository/<group-id>/<version>/<artifact-id>-<classifier-id>-<version>.tar.gz
```

> NOTE: local repository would support unpacked charts for easier development/fixing

###### Better `Staging` and `Stable` Chart Separation
At the moment there is no way to tell wether a downloaded chart is from stable or staging repository.
Staging chart (archive) names could be appended with `SNAPSHOT`.
Releases and snapshots should not mix and therefore reside in separate repositories.

If a release chart is found in the local repository, it would not be re-fetched unless forced.
Snapshots would be checked for updates periodically and/or before each build as specified here:
https://cwiki.apache.org/confluence/display/MAVENOLD/Repository+-+SNAPSHOT+Handling

###### Separation of Concerns
Helm should be treated as a builder tool, not all-in-one. Consider Java, Python or even JavaScript ecosystems: depdenency handling is separated from source processing. For dependency management one uses tools such as maven, gradle, pip or npm.

```
helm package -d <chart_out> <chart_input> -cp <chart path>
```
Where `<chart_path>` is a comma separated list of dependency charts and/or directories containing them.

This would make it possible to manage dependency list with a build automation tool such as Gradle or Maven.

The change proposed in this section would be compatible with the current helm chart management system.


> NOTE 1: if the dependency management mechanism becomes independent from Helm, then it would be possible to reuse existing versioning infrastructure for Maven and Gradle.

> NOTE 2
> the issue with this versioning approach is that in the example root chart would have two postgres charts, one direct and one indirect via kafka. both of which may have different version dependencies:
rootChart:
 -kafka 1.0.0
   - postgres 2.1
 -postgres 1.1

> but the internal  depdendency management makes it difficult to work with multi-project builds where a declared dependency would be automatically build by the build manager.
