# syntax=docker/dockerfile:1
FROM node:22-bookworm-slim AS frontend
WORKDIR /src/library.client
COPY library.client/package.json library.client/package-lock.json ./
RUN npm ci
COPY library.client/ ./
# Empty means same-origin requests to the API served by ASP.NET Core.
ARG VITE_API_URL=
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS backend
WORKDIR /src
COPY Library.Server/Library.Server.csproj Library.Server/
COPY Zheng.Bll/Zheng.Bll.csproj Zheng.Bll/
COPY Zheng.Infra.Data/Zheng.Infra.Data.csproj Zheng.Infra.Data/
COPY Zheng.Mdl/Zheng.Mdl.csproj Zheng.Mdl/
RUN dotnet restore Library.Server/Library.Server.csproj -p:BuildClient=false
COPY Library.Server/ Library.Server/
COPY Zheng.Bll/ Zheng.Bll/
COPY Zheng.Infra.Data/ Zheng.Infra.Data/
COPY Zheng.Mdl/ Zheng.Mdl/
RUN dotnet publish Library.Server/Library.Server.csproj -c Release --no-restore -p:BuildClient=false -o /out

# Compose runs this bundle once before starting the website.
# Building the bundle does not connect to a database.
FROM backend AS migration-build
RUN dotnet tool install --tool-path /tools dotnet-ef --version 9.0.6
RUN mkdir -p /migrations && /tools/dotnet-ef migrations bundle \
    --project Zheng.Infra.Data/Zheng.Infra.Data.csproj \
    --context PostgresLibraryContext --configuration Release \
    --output /migrations/efbundle

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS migrator
WORKDIR /app
COPY --from=migration-build /migrations/efbundle ./
USER $APP_UID
ENTRYPOINT ["./efbundle"]

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
ENV ASPNETCORE_ENVIRONMENT=Production
COPY --from=backend /out/ ./
COPY --from=frontend /src/library.client/dist/ ./wwwroot/
RUN mkdir -p /app/wwwroot/uploads /home/app/.aspnet/DataProtection-Keys \
    && chown -R $APP_UID:$APP_UID /app/wwwroot/uploads /home/app/.aspnet
USER $APP_UID
EXPOSE 8080
CMD ["sh", "-c", "exec dotnet Library.Server.dll --urls http://0.0.0.0:${PORT:-8080}"]
