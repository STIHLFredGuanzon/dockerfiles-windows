# escape=`

FROM microsoft/windowsservercore AS installer
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

ENV UMBRACO_VERSION="7.6.0" `
    UMBRACO_DOWNLOAD_ID="270489"

RUN Invoke-WebRequest -UseBasicParsing -OutFile umbraco.zip -Uri "https://our.umbraco.org/ReleaseDownload?id=$($env:UMBRACO_DOWNLOAD_ID)"; `
    Expand-Archive umbraco.zip -DestinationPath C:\umbraco

# app image
FROM microsoft/aspnet:windowsservercore-10.0.14393.1066
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

ENV UMBRACO_VERSION="7.6.0" `
    UMBRACO_ROOT="C:\inetpub\wwwroot\Umbraco"

RUN Remove-Website 'Default Web Site'; `
    New-Item -Path $env:UMBRACO_ROOT -Type Directory -Force; `
    New-Website -Name 'umbraco' -PhysicalPath $env:UMBRACO_ROOT -Port 80 -Force; 
	
# START SQL Server Express
FROM microsoft/windowsservercore

LABEL maintainer "Perry Skountrianos"

# Download Links:
ENV sql_express_download_url "https://go.microsoft.com/fwlink/?linkid=829176"

ENV sa_password="stihl123stihl123" \
    attach_dbs="[Umbraco]" \
    ACCEPT_EULA="Y" \
    sa_password_path="C:\ProgramData\Docker\secrets\sa-password"

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

# make install files accessible
COPY start.ps1 /
WORKDIR /

RUN Invoke-WebRequest -Uri $env:sql_express_download_url -OutFile sqlexpress.exe ; \
        Start-Process -Wait -FilePath .\sqlexpress.exe -ArgumentList /qs, /x:setup ; \
        .\setup\setup.exe /q /ACTION=Install /INSTANCENAME=SQLEXPRESS /FEATURES=SQLEngine /UPDATEENABLED=0 /SQLSVCACCOUNT='NT AUTHORITY\System' /SQLSYSADMINACCOUNTS='BUILTIN\ADMINISTRATORS' /TCPENABLED=1 /NPENABLED=0 /IACCEPTSQLSERVERLICENSETERMS ; \
        Remove-Item -Recurse -Force sqlexpress.exe, setup

RUN stop-service MSSQL`$SQLEXPRESS ; \
        set-itemproperty -path 'HKLM:\software\microsoft\microsoft sql server\mssql14.SQLEXPRESS\mssqlserver\supersocketnetlib\tcp\ipall' -name tcpdynamicports -value '' ; \
        set-itemproperty -path 'HKLM:\software\microsoft\microsoft sql server\mssql14.SQLEXPRESS\mssqlserver\supersocketnetlib\tcp\ipall' -name tcpport -value 1433 ; \
        set-itemproperty -path 'HKLM:\software\microsoft\microsoft sql server\mssql14.SQLEXPRESS\mssqlserver\' -name LoginMode -value 2 ;

CMD .\start -sa_password $env:sa_password -ACCEPT_EULA $env:ACCEPT_EULA -attach_dbs \"$env:attach_dbs\" -Verbose
# END SQL Server Express

HEALTHCHECK --interval=5s `
 CMD powershell -command `
    try { `
     $response = iwr http://localhost/ -UseBasicParsing; `
     if ($response.StatusCode -eq 200) { return 0} `
     else {return 1}; `
    } catch { return 1 }

COPY --from=installer C:\umbraco ${UMBRACO_ROOT}
COPY Set-UmbracoAcl.ps1 C:\
RUN C:\Set-UmbracoAcl.ps1