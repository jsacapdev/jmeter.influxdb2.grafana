# Viewing JMeter test results as the test is running

When JMeter tests are running, its not that easy to understand what is happening in terms of results.

This solution (which is mainly a series of notes in documentation) shows how you can do that.

The solution an an application architecture of:

1. Docker container running some JMeter tests
2. Docker container that has InfluxDb2 as a time series database
3. Docker container that uses Grafana as a dashboard
4. A very simple dotnet core web api that the JMeter tests run against


