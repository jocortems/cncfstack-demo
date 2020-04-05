#See https://aka.ms/containerfastmode to understand how Visual Studio uses this Dockerfile to build your images for faster debugging.

FROM openfaas/of-watchdog:0.7.2 as watchdog

FROM mcr.microsoft.com/dotnet/core/aspnet:3.1-buster-slim AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/core/sdk:3.1-buster AS build
WORKDIR /src
COPY ["Contoso.Expenses.API/Contoso.Expenses.API.csproj", "Contoso.Expenses.API/"]
RUN dotnet restore "Contoso.Expenses.API/Contoso.Expenses.API.csproj"
COPY . .
WORKDIR "/src/Contoso.Expenses.API"
RUN dotnet build "Contoso.Expenses.API.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "Contoso.Expenses.API.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
COPY --from=watchdog /fwatchdog /usr/bin/fwatchdog
RUN chmod +x /usr/bin/fwatchdog

ENV fprocess="dotnet Contoso.Expenses.API.dll"
ENV upstream_url="http://127.0.0.1:80"
ENV mode="http"

ENTRYPOINT ["fwatchdog"]