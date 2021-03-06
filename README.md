## Splunk Nozzle

Cloud Foundry Firehose-to-Splunk Nozzle

### Usage
Splunk nozzle is used to stream Cloud Foundry Firehose events to Splunk HTTP Event Collector. Using pre-defined Splunk sourcetypes, the nozzle automatically parses the events and enriches them with additional metadata before forwarding to Splunk. For detailed descriptions of each Firehose event type and their fields, refer to underlying [dropsonde protocol](https://github.com/cloudfoundry/dropsonde-protocol). Below is a mapping of each firehose event type to its corresponding Splunk sourcetype. Refer to [Searching Events](#searching-events) for example Splunk searches.

| Firehose event type | Splunk sourcetype | Description
|---|---|---
| Error | `cf:error` | An Error event represents an error in the originating process
| HttpStartStop | `cf:httpstartstop` | An HttpStartStop event represents the whole lifecycle of an HTTP request
| LogMessage | `cf:logmessage` | A LogMessage contains a "log line" and associated metadata
| ContainerMetric | `cf:containermetric` | A ContainerMetric records resource usage of an app in a container
| CounterEvent | `cf:counterevent` | A CounterEvent represents the increment of a counter
| ValueMetric | `cf:valuemetric` | A ValueMetric indicates the value of a metric at an instant in time

In addition, logs from the nozzle itself are of sourcetype `cf:splunknozzle`.

### Setup

The Nozzle requires a user with the scope `doppler.firehose` and 
`cloud_controller.admin_read_only` (the latter is only required if `ADD_APP_INFO` is true). 
You can either
* Add the user manually using [uaac](https://github.com/cloudfoundry/cf-uaac)
* Add a new user to the deployment manifest; see [uaa.scim.users](https://github.com/cloudfoundry/uaa-release/blob/master/jobs/uaa/spec)

Manifest example:
```yaml
uaa:
  scim:
    users:
      - splunk-firehose|password123|cloud_controller.admin_read_only,doppler.firehose
```

`uaac` example:
```shell
uaac target https://uaa.[system domain url]
uaac token client get admin -s [admin client credentials secret]
uaac -t user add splunk-nozzle --password password123 --emails na
uaac -t member add cloud_controller.admin_read_only splunk-nozzle
uaac -t member add doppler.firehose splunk-nozzle
```

`cloud_controller.admin_read_only` will work for cf v241 
or later. Earlier versions should use `cloud_controller.admin` instead.

### Development

#### Software Requirements

Make sure you have the following installed on your workstation:

| Software | Version
| --- | --- |
| go | go1.7.x
| glide | 0.12.x

Then install all dependent packages via [Glide](https://glide.sh/):
```
$ cd <REPO_ROOT_DIRECTORY>
$ glide install
```

#### Environment

For development against [bosh-lite](https://github.com/cloudfoundry/bosh-lite),
copy `scripts/nozzle.sh.template` to `scripts/nozzle.sh` and supply missing values:
```
$ cp script/dev.sh.template scripts/nozzle.sh
$ chmod +x scripts/nozzle.sh
```

Build project:
```
$ go build main.go
```

Reinstall dependencies:
```
glide update
```

Update dependencies (without `strip-vcs` you'll end up with submodules):
```
glide install --strip-vendor --strip-vcs --update-vendored
```

Add a new dependency:
```
glide get github.com/kelseyhightower/envconfig --strip-vendor
```

Run tests with [Ginkgo](http://onsi.github.io/ginkgo/)
```
$ ginkgo -r
```

Run app
```
# this will run: go run main.go
$ ./scripts/nozzle.sh
```

#### CI

https://concourse.cfplatformeng.com/teams/splunk/pipelines/splunk-firehose-tile-build

### Push as an App to Cloud Foundry

[splunk-firehose-nozzle-release](https://github.com/cloudfoundry-community/splunk-firehose-nozzle-release)
packages this code into a
[BOSH](https://bosh.io) release for deployment. The code could also be run on
Cloud Foundry as an application. See the **Setup** section for details
on making a user and credentials.

1. Download the latest release
    
    ```shell
    git clone https://github.com/cloudfoundry-community/splunk-firehose-nozzle.git
    cd splunk-firehose-nozzle
    ```
    
1. Authenticate to Cloud Foundruy
    
    ```shell
    cf login -a https://api.[your cf system domain] -u [your id]
    ```

1. Copy the manifest template and fill in needed values (using the credentials created during setup)

    ```shell
    cp manifest.yml.template manifest.yml 
    vim manifest.yml
    ```

1. Push the nozzle

    ```shell
    cf push
    ```

#### Troubleshooting
In most cases, you would only need to troubleshoot from Splunk which include not only firehose data but also this nozzle internal logs. However, if the nozzle is still not forwarding any data, a good place to start is to get app internal logs directly:

```shell
cf logs splunk-firehose-nozzle
```

A common mis-configuration occurs when having invalid or unsigned certificate for Cloud Foundry API endpoint. In that case, for non-production environments, you can set `SKIP_SSL_VALIDATION` to `true` in above manifest.yml before re-deploying the app.

#### Searching Events

Here are two short Splunk queries to start exploring some of the events

```
sourcetype="cf:valuemetric"
    | stats avg(value) by job_instance, name
```

```
sourcetype="cf:counterevent"
    | eval job_and_name=source+"-"+name
    | stats values(job_and_name)
```
