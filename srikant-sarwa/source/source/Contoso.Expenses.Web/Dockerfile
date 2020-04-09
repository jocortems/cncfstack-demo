FROM mcr.microsoft.com/dotnet/core/sdk:2.2 AS build-env
WORKDIR /app

COPY ./Contoso.Common ./Contoso.Common/
COPY ./Contoso.Expenses.Web ./Contoso.Expenses.Web/

WORKDIR /app/Contoso.Expenses.Web

RUN dotnet publish -c Release -o out

FROM mcr.microsoft.com/dotnet/core/aspnet:2.2
WORKDIR /app
COPY --from=build-env /app/Contoso.Expenses.Web/out .

ENTRYPOINT ["dotnet", "Contoso.Expenses.Web.dll"]