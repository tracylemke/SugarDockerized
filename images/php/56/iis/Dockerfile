# escape=`
#
# This Dockerfile is provided for demonstration purposes only and it is not supported by Microsoft
# PHP 5.6.36 x64 running on IIS
#

FROM microsoft/iis AS php56

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop';"]

#RUN mkdir C:\tmp
#RUN mkdir C:\config

RUN `
    try { `
        mkdir C:\config; `
        mkdir C:\tmp; `
        `
        # Need to set the SecurityProtocol policy with the ServicePointManager to resolve conn error `
        [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.SecurityProtocolType]'Ssl3,Tls,Tls11,Tls12'; `
        `
        # Install PHP `
        # $UserAgent =  [Microsoft.PowerShell.Commands.PSUserAgent]::InternetExplorer; ` # <<< Ideally, this should be enough but tends to fail quite often `
        # So, I just hardcoded IE's string value `
        $UserAgent = 'Mozilla/5.0 (compatible; MSIE 9.0; Windows NT; Windows NT 10.0; en-US)'; `
        Invoke-WebRequest 'https://windows.php.net/downloads/releases/php-5.6.36-nts-Win32-VC11-x64.zip' -UserAgent $UserAgent -OutFile C:\tmp\php.zip; `
        Write-Host "php downloaded."; `
        `
        Invoke-WebRequest 'https://download.microsoft.com/download/6/A/A/6AA4EDFF-645B-48C5-81CC-ED5963AEAD48/vc_redist.x64.exe' -UserAgent '' -OutFile C:\tmp\vc_redist-x64.exe; `
        Write-Host "VC14 downloaded."; `
        `
        Invoke-WebRequest 'https://sourceforge.net/projects/wincache/files/wincache-2.0.0/wincachewpi-2.0.0.8-7.1-nts-vc14-x64.exe/download' -UserAgent $UserAgent -OutFile C:\tmp\php_wincache.exe; `
        Write-Host "WinCache downloaded."; `
        `
        Expand-Archive -Path c:\tmp\php.zip -DestinationPath C:\PHP; `
        `
        # Install PHP Win Cache `
        C:\tmp\php_wincache.exe /Q /C "/T:C:\tmp\php_wincache_msi"; `
        Start-Process -FilePath msiexec -ArgumentList '/a C:\tmp\php_wincache_msi\wincache71x64wpi.msi /qb TARGETDIR=C:\tmp\php_wincache_msi\extracted' -NoNewWindow -PassThru -Wait; `
        Copy-Item C:\tmp\php_wincache_msi\extracted\PFiles\php_wincache.dll c:\PHP\ext; `
        `
        # Configure PHP `
        # Copy-Item C:\PHP\php.ini-production C:\PHP\php.ini; `
    } `
    catch { `
        $_.Exception; `
        $_; `
        exit 1; `
    }

COPY config/opcache.exclude.files config/
COPY config/php.ini PHP/

FROM microsoft/iis

COPY --from=php56 ["/php/", "/php/"]
COPY --from=php56 ["/tmp/vc_redist-x64.exe", "/tmp/vc_redist-x64.exe"]

#
# Enable required IIS Features
# Install VC Redist 14
# Configure IIS
# Configure system PATH
#
RUN dism.exe /Online /Enable-Feature /FeatureName:IIS-CGI /All && `
    C:\tmp\vc_redist-x64.exe /quiet /install && `
    del C:\tmp\vc_redist-x64.exe && `
    %windir%\system32\inetsrv\appcmd.exe set config /section:system.webServer/fastCgi /+[fullPath='c:\PHP\php-cgi.exe'] && `
    %windir%\system32\inetsrv\appcmd.exe set config /section:system.webServer/handlers /+[name='PHP_via_FastCGI',path='*.php',verb='*',modules='FastCgiModule',scriptProcessor='c:\PHP\php-cgi.exe',resourceType='Either'] && `
    %windir%\system32\inetsrv\appcmd.exe set config -section:system.webServer/fastCgi /[fullPath='c:\PHP\php-cgi.exe'].instanceMaxRequests:10000 && `
    %windir%\system32\inetsrv\appcmd.exe set config -section:system.webServer/fastCgi /+[fullPath='c:\PHP\php-cgi.exe'].environmentVariables.[name='PHP_FCGI_MAX_REQUESTS',value='10000'] && `
    %windir%\system32\inetsrv\appcmd.exe set config -section:system.webServer/fastCgi /+[fullPath='c:\PHP\php-cgi.exe'].environmentVariables.[name='PHPRC',value='C:\PHP'] && `
    %windir%\system32\inetsrv\appcmd.exe set config /section:defaultDocument /enabled:true /+files.[value='index.php'] && `
    setx PATH /M %PATH%;C:\PHP && `
    setx PHP /M "C:\PHP" && `
    del C:\inetpub\wwwroot\* /Q



# Add phpinfo in
RUN powershell.exe -Command "'<?php phpinfo(); ?>' | Out-File C:\inetpub\wwwroot\phpinfo.php" -Encoding UTF8

ENTRYPOINT ["C:\\ServiceMonitor.exe", "w3svc"]
