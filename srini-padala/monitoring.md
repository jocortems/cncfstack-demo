## Monitoring using App Metrics, Prometheus & Grafana

This document outlines how to instrument Contoso Expenses application and monitor using Prometheus & Grafana.

 - Local development & testing 
 - Deployment & configuration of AKS cluster

### Development

First step is to instrument the application using [App Metrics](https://github.com/AppMetrics/AppMetrics). 

- Add following App Metrics NuGet packages. I've used latest preview version.
```
    1. App.Metrics.AspNetCore 
    2. App.Metrics.AspNetCore.Endpoints
    3. App.Metrics.AspNetCore.Tracking
    4. App.Metrics.Formatters.Prometheus
    5. App.Metrics.AspNetCore.Mvc
```
   + ![App Metrics NuGet packages](./images/monitoring/appmetrics-nugetpackages.png)

- In Startup.cs class, ConfigureServices method add the following on the top. This will register IMetrics interface in our application. 

```
       services.Configure<KestrelServerOptions>(options => { options.AllowSynchronousIO = true; });
		            services.AddMetrics();
```
- In Program.cs class, CreateHostBuilder method add the follolwing.Comment the existing CreateWebHostBuilder and add the following:


```
			    public static IHostBuilder CreateHostBuilder(string[] args) =>
			           Host.CreateDefaultBuilder(args)
			               .UseMetricsWebTracking()
			               .UseMetrics(options =>
			               {
			                   options.EndpointOptions = endpointsOptions =>
			                   {
			                       endpointsOptions.MetricsTextEndpointOutputFormatter = new MetricsPrometheusTextOutputFormatter();
			                       endpointsOptions.MetricsEndpointOutputFormatter = new MetricsPrometheusProtobufOutputFormatter();
			                       endpointsOptions.EnvironmentInfoEndpointEnabled = false;
			                   };
			               })
			               .ConfigureWebHostDefaults(webBuilder =>
			               {
			                   webBuilder.UseStartup<Startup>();
			               });

```
- That’s all, now you can run the application and check out the out of the box metrics using /metrics-text endpoint for application level metrics.
  + For example: https://localhost:44335/metrics-text 

- Now you can let Prometheus scrape App Metric's metrics endpoint (example: https://localhost:44335/metrics-text ) and collect the metrics and store it in Prometheus timeseries database. From there you can leverage Grafana to point to Prometheus database to import the data to its own store to show nice dashboards.

- To run Prometheus locally (you can also run as a docker container)
  + Download stable (not pre-release)  latest Prometheus (example: prometheus-2.17.2.windows-amd64.tar.gz) from https://prometheus.io/download/
  + Extract the zip
  + Run the Prometheus exe (for example: "C:\Users\umarm\Downloads\prometheus-2.17.1.windows-amd64\promtool.exe") 
  + Before you run the exe, modify the prometheus.yaml to scrape right endpoints under Scrape_configs: section
  ```yml
  - job_name: 'contosoweb'
	static_configs:
	  - targets: ['localhost:44335']
	metrics_path: /metrics-text
  ```
  + ![Prometheus Scrape Config](./images/monitoring/prometheus-scrapeconfig.png)

- Run Prometheus.exe and it will be ready to scrape metrics. 

- Run Contoso Web application locally in debug mode, give it about 15 seconds, Prometheus will start scraping metrics endpoint. 

- Now browse to, http://localhost:9090 and search for any metrics for example: application_httprequests_active and you will see 1 in the value. 
 + ![Prometheus Dashboard](./images/monitoring/prometheus-dashboard.png)

 