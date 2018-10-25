## Proposal: Chart Packaging System Improvements

#### Motivation

In Helm v2, charts are uniquely identified using a combination of three properties: chart name, chart version and repository - `name:version:repository`.
The inclusion of the `repository` presents several implications that may potentially hinder future improvements and management of Helm ecosystem:

1) No built-in vendor distinction. Take for example identically named apache `kafka` and bitnami `kafka` charts. Both have the same `name` and their `version`'s can overlap. They are uniquely distinguished only by their `repository` . Consequentially, there is no way to store both charts in the same, e.g. private, repository. The alternative is to rename the charts as `kafka-bitnami` and `kafka-apache`, but this non-standardised and non-enforced vendoring may potentially cause even more confusion in the community;

2) Repositories may move or experience downtime. Since chart idnetification dependends on the `repository` property, chart dependency resolution is susceptible to changes of the repsotory address and communication protocol (the hardcoded http/https) - the changes to the repository will have to be reflected in the chart and its nested dependencies;

3) Repository outages cannot be worked around with mirror servers due to hardcoded `repository` address in the requriements.yaml;

4) Explicit file system path usage should be discouraged - it enforces rigid project structures that are difficult to maintain and does not promote reusability, especially across multi-project setups.

5) Repositories must be declared twice: in requirements.yaml and added as a helm repository;

The multi-level nested chart scenarios typically depend on a greater number of charts and therefore are more likely to face the aforementioned issues.

A similar issue pertaining to `respository` identifier was raised in docker community:
https://forums.docker.com/t/working-with-private-registry-image-names-must-contain-port-number/3654

The following sections make up a proposal to address the above.

#### Versioning System

The Helm community could potentially benefit from adopting a Maven-like convention to uniquely identify Helm charts. The naming convention is comprised of four elements (see below), where `group` specifies the project/vendor and `name` is the chart name; `<classifier>` is optional and can be left unimplemented for the future use:

```xml
<group>:<artifact>:<classifier>:<version>
```

The proposed identification makes it possible to uniquely identify a chart from two different vendors:
```
com.apache:kafka:1.2.3
com.bitnami:kafka:1.2.3
```

Locally customised/extended chart retains the `name` that best describes the contents of the chart`:

```
com.custom:kafka:1.2.3
```

These charts can then be stored in a local Helm repository without a clash.


#### Decoupling Chart from Repository; Caching

Replacing `repository` with `group` in the charts as proposed in the previous section simplifies chart dependency management.

For example, the `requirements.yaml` file then would no longer store repository information and would be concerned only with chart identification:
```yaml
dependencies:
- group: com.apache
  name: kafka
  version: 1.2.3
- id: com.bitnami
  name kafka
  version: 1.2.3
  alias: still_works
  enabled: true
```

Helm would firstly look for the dependencies in a local repository/cache located in `~/.helm/repository`. For example, one of the kafka chart examples mentioned earlier would be stored in:
```
~/.helm/repository/com/apache/kafka/1.2.3/
```
The directory would contain:
```
 kafka-1.2.3.tar.gz - chart archive
 kafka-1.2.3.tar.gz.sha1 - checksum for the chart
 kafka-1.2.3.pom - chart descriptor, e.g. what other dependencies it might require
 kafka-1.2.3.pom.sha1 - chart descriptor checksum
```

If the dependency in question is not found locally, Helm would look for it in specified locations and/or remote repositories. The list of external repositories would be defined either centrally as today or passed as a parameter:

```
helm package -d <chart_out> <chart_input> --chart-path <path_to_charts>
```

Where `<path_to_charts>` is a comma separated list repository URLs and/or directories/files containing charts.

#### Better `Staging` and `Stable` Chart Separation

At the moment there is no way to tell whether chart is `stable` or `innvubating`.
Again, we could borrow the convention from Maven and append staging chart's `name` with `-STAGING`.
