ARG ASPNET_REPO=mcr.microsoft.com/dotnet/aspnet
ARG SDK_REPO=mcr.microsoft.com/dotnet/sdk

# Installer image
FROM $SDK_REPO:8.0.100-alpha.1-jammy-amd64 AS installer

# Install .NET Monitor
ENV DOTNET_MONITOR_VERSION=8.0.0-alpha.1.22606.5
RUN curl -fSL --output dotnet-monitor.$DOTNET_MONITOR_VERSION.nupkg https://dotnetbuilds.azureedge.net/public/diagnostics/monitor/$DOTNET_MONITOR_VERSION/dotnet-monitor.$DOTNET_MONITOR_VERSION.nupkg \
    && dotnetmonitor_sha512='7d0ebc07ec62072fd43bdcf0b7abac54eb0bb9793798518b9f3f958cc8ac78a878fc0910c9585535f865447b306f3d3948cc9f4ff7c524f4108ca42344c5bac9' \
    && echo "$dotnetmonitor_sha512  dotnet-monitor.$DOTNET_MONITOR_VERSION.nupkg" | sha512sum -c - \
    && dotnet tool install dotnet-monitor --tool-path /app --add-source / --version $DOTNET_MONITOR_VERSION --framework net8.0 --no-cache \
    # To reduce image size, remove all non-net8.0 TFMs
    # To do this safely, we need to first find everything that dotnet tool installed named "dotnet-monitor".
    # The /dotnet-monitor/ folder exists under .store which is not a stable constant to rely on. The result is multi stage search:
    # 1. Find any files in /app
    # 2. Match anything that is under a *folder* called 'dotnet-monitor'
    # 3. Match anything from step 2 that isn't in a '/tools/net8.0' folder (/tools is the folder we use in a nuget file that is stable)
    # 4. Delete everything from step 3
    && find /app -type f -print | grep -i '.*/dotnet-monitor/.*' | grep -v -i '.*/tools/net8[.]0' | xargs rm -rf \
    # To reduce image size further, remove the non-linux assemblies
    && find /app -type f -print | grep -i '.*/dotnet-monitor/.*' | grep -i -E '/(runtimes|shared|shims)/(win|osx)' | xargs rm -rf \
    # To reduce image size further, remove linux assemblies that do not match the x64 architecture
    && find /app -type f -print | grep -i '.*/dotnet-monitor/.*' | grep -i -E '/(runtimes|shared|shims)/(linux-arm64|linux-musl-arm64)' | xargs rm -rf \
    # To reduce image size further, remove symbol files
    && find /app -type f \( -name *.pdb -o -name *.dbg \) -print | xargs rm -rf \
    # Remove all the empty directories left by the previous step
    && find /app -type d -empty -delete \
    # Allow other users to run the tool
    && chmod 755 /app/dotnet-monitor \
    && rm dotnet-monitor.$DOTNET_MONITOR_VERSION.nupkg


# Monitor image
FROM $ASPNET_REPO:8.0.0-alpha.1-jammy-chiseled-amd64

WORKDIR /app
COPY --from=installer /app .

ENV \
    # Unset ASPNETCORE_URLS from aspnet base image
    ASPNETCORE_URLS= \
    # Disable debugger and profiler diagnostics to avoid diagnosing self.
    COMPlus_EnableDiagnostics=0 \
    # Default Filter
    DefaultProcess__Filters__0__Key=ProcessId \
    DefaultProcess__Filters__0__Value=1 \
    # Remove Unix Domain Socket before starting diagnostic port server
    DiagnosticPort__DeleteEndpointOnStartup=true \
    # Server GC mode
    DOTNET_gcServer=1 \
    # Logging: JSON format so that analytic platforms can get discrete entry information
    Logging__Console__FormatterName=json \
    # Logging: Use round-trip date/time format without timezone information (always logged in UTC)
    Logging__Console__FormatterOptions__TimestampFormat=yyyy'-'MM'-'dd'T'HH':'mm':'ss'.'fffffff'Z' \
    # Logging: Write timestamps using UTC offset (+0:00)
    Logging__Console__FormatterOptions__UseUtcTimestamp=true \
    # Add dotnet-monitor path to front of PATH for easier, prioritized execution
    PATH="/app:${PATH}"

ENTRYPOINT [ "dotnet-monitor" ]
CMD [ "collect", "--urls", "https://+:52323", "--metricUrls", "http://+:52325" ]