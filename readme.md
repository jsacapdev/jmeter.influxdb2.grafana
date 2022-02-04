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
