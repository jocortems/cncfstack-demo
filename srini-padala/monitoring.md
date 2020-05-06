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

