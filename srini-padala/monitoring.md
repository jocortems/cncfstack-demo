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
  ![App Metrics NuGet packages](./images/monitoring/appmetrics-nugetpackages.png)

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



