# escape=`

ARG REPO=mcr.microsoft.com/dotnet/aspnet

# Installer image
FROM mcr.microsoft.com/windows/servercore:ltsc2022-amd64 AS installer

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

RUN `
    # Retrieve .NET SDK
    $sdk_version = '7.0.100-alpha.1.22069.1'; `
    Invoke-WebRequest -OutFile dotnet.zip https://dotnetbuilds.azureedge.net/public/Sdk/$sdk_version/dotnet-sdk-$sdk_version-win-x64.zip; `
    $dotnet_sha512 = '5baa534026cdf1020afd0187ec264fb347c1dc8abcc0a0e66829bb75dd70cef7c9315e21f21f2086cef59a0e2240bf85c7f8d4c0f5fd2c06f8d83b4a1c1cef26'; `
    if ((Get-FileHash dotnet.zip -Algorithm sha512).Hash -ne $dotnet_sha512) { `
        Write-Host 'CHECKSUM VERIFICATION FAILED!'; `
        exit 1; `
    }; `
    mkdir dotnet; `
    tar -oxzf dotnet.zip -C dotnet; `
    Remove-Item -Force dotnet.zip; `
    `
    # Install PowerShell global tool
    $powershell_version = '7.3.0-preview.1'; `
    Invoke-WebRequest -OutFile PowerShell.Windows.x64.$powershell_version.nupkg https://pwshtool.blob.core.windows.net/tool/$powershell_version/PowerShell.Windows.x64.$powershell_version.nupkg; `
    $powershell_sha512 = '9dcaace7382b8946bde8f8fcf177794962f881882e3cfd7e71fb4b64778872a471cefc3af5bf3cfc0de5af40602be85f67ec78ceaa57e049f5ad341ac4ec837d'; `
    if ((Get-FileHash PowerShell.Windows.x64.$powershell_version.nupkg -Algorithm sha512).Hash -ne $powershell_sha512) { `
        Write-Host 'CHECKSUM VERIFICATION FAILED!'; `
        exit 1; `
    }; `
    & \dotnet\dotnet tool install --add-source . --tool-path \powershell --version $powershell_version PowerShell.Windows.x64; `
    & \dotnet\dotnet nuget locals all --clear; `
    Remove-Item -Force PowerShell.Windows.x64.$powershell_version.nupkg; `
    Remove-Item -Path \powershell\.store\powershell.windows.x64\$powershell_version\powershell.windows.x64\$powershell_version\powershell.windows.x64.$powershell_version.nupkg -Force; `
    `
    # Workaround for https://github.com/dotnet/sdk/issues/18410
    \dotnet\dotnet build-server shutdown; `
    `
    # Delete everything in the dotnet folder that's not needed in the SDK layer but will instead be derived from base layers
    Get-ChildItem -Exclude 'LICENSE.txt','ThirdPartyNotices.txt','packs','sdk','sdk-manifests','templates','shared' -Path dotnet `
        | Remove-Item -Force -Recurse; `
    Get-ChildItem -Exclude Microsoft.WindowsDesktop.App -Path dotnet\shared | Remove-Item -Force -Recurse

# SDK image
FROM $REPO:7.0.0-alpha.1-nanoserver-ltsc2022

ENV `
    # Unset ASPNETCORE_URLS from aspnet base image
    ASPNETCORE_URLS= `
    # Do not generate certificate
    DOTNET_GENERATE_ASPNET_CERTIFICATE=false `
    # Do not show first run text
    DOTNET_NOLOGO=true `
    # SDK version
    DOTNET_SDK_VERSION=7.0.100-alpha.1.22069.1 `
    # Enable correct mode for dotnet watch (only mode supported in a container)
    DOTNET_USE_POLLING_FILE_WATCHER=true `
    # Unset Logging__Console__FormatterName from aspnet base image
    Logging__Console__FormatterName= `
    # Skip extraction of XML docs - generally not useful within an image/container - helps performance
    NUGET_XMLDOC_MODE=skip `
    # PowerShell telemetry for docker image usage
    POWERSHELL_DISTRIBUTION_CHANNEL=PSDocker-DotnetSDK-NanoServer-ltsc2022

# In order to set system PATH, ContainerAdministrator must be used
USER ContainerAdministrator
RUN setx /M PATH "%PATH%;C:\Program Files\powershell"
USER ContainerUser

COPY --from=installer ["/dotnet", "/Program Files/dotnet"]

COPY --from=installer ["/powershell", "/Program Files/powershell"]

# Trigger first run experience by running arbitrary cmd
RUN dotnet help