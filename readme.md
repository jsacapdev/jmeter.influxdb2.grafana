# Viewing JMeter test results as the test is running

When JMeter tests are running, its not that easy to understand what is happening in terms of results.

This solution (which is mainly a series of notes in documentation) shows how you can do that.

The solution an an application architecture of:

1. Docker container running some JMeter tests
2. Docker container that has InfluxDb2 as a time series database
3. Docker container that uses Grafana as a dashboard
4. A very simple dotnet core web api that the JMeter tests run against

## Database (InfluxDb)

Create the time series database in a docker container using the following command:

``` pwsh
docker run --name influxdb `
      -d -p 8086:8086 `
      -v $PWD/data:/var/lib/influxdb2 `
      -v $PWD/config:/etc/influxdb2 `
      -e DOCKER_INFLUXDB_INIT_MODE=setup `
      -e DOCKER_INFLUXDB_INIT_USERNAME=my-user `
      -e DOCKER_INFLUXDB_INIT_PASSWORD=my-password `
      -e DOCKER_INFLUXDB_INIT_ORG=my-org `
      -e DOCKER_INFLUXDB_INIT_BUCKET=my-bucket `
      -e DOCKER_INFLUXDB_INIT_RETENTION=1w `
      -e DOCKER_INFLUXDB_INIT_ADMIN_TOKEN=my-super-secret-auth-token `
      influxdb:2.0
```

Then login to the user interface on the local machine if you like:

`http://localhost:8086/`

## Grafana

Again, start up a docker container:

`docker run -d --name grafana -p 3000:3000 grafana/grafana`

Then login to the user interface (the username and password by default are `admin`):

`http://localhost:3000/`

Click on Configuration -> Data Sources -> Add Data Source. Choose InfluxDb as the data source.

Change the query language to Flux. Uncheck basic auth. Use the following URL to point to Influx:

`http://172.17.0.1:8086`

Note that this is the IP of the network docker is sitting in. This is because the container running grafana will need to talk to the container running influx. `Localhost` wont work because you are not trying to connect to the time series database from the host.

Populate organization, bucket and token using the values above used when configuring the influxdb container.

Get the [dashboard](https://grafana.com/grafana/dashboards/13644) and download the json. All of the context comes from a post of [using a grafana dashboard using influx DB 2.0](https://dzone.com/articles/jmeter-integration-with-grafana-and-influxdbv2).

Upload the dashboard to granfana clicking the `+` icon -> `Import`. And choose the data source as `InfluxDb`.

## JMeter

The challenge here is that we need a `jar` that is required in the JMeter script (to send data in the correct format to Influx - so it can be viewed on the dashboard), but that `jar` is not compatible with the default JMeter docker container. So first we need to rebuild the JMeter docker container so that it uses a [newer version of Java](https://github.com/justb4/docker-jmeter/blob/b9c11014d2edac48022288e11682fe938235b4de/Dockerfile#L20) (that is compatible with the `jar`). I changed `openjdk8-jre` to `openjdk11`. And rebuilt it.

Then put the [`jar`](https://github.com/mderevyankoaqa/jmeter-influxdb2-listener-plugin/releases) you need in a location where the JMeter external plugins are kept (i.e. `/home/tda/jmeter/apache-jmeter-5.4.1/lib/ext`).

Then `cd` to the directory that has the `local-web-api.jmx` and run the following command to fire up the docker container running the JMeter script:

``` pwsh
$run = "test_results/$(New-Guid)"; `
New-Item -Name $run -ItemType "directory"; `
docker run --name jmetertest `
    -i `
    -v /home/tda/jmeter/apache-jmeter-5.4.1/lib/ext:/plugins `
    -v ${PWD}:/test `
    --rm `
    --network=host `
    justb4/jmeter:5.4 `
    -Dlog_level.jmeter=DEBUG `
    "-Jjmeterengine.force.system.exit=true" `
    -n `
    -t /test/local-web-api.jmx `
    -l "/test/$run/sample.jtl" `
    -j /test/$run/sample.log `
    -e -o /test/$run/sample;
```

In this case `/home/tda/jmeter/apache-jmeter-5.4.1/lib/ext` has the `jar` we need to log results to the influx db.
