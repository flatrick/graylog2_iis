# graylog2_iis
Configuration to get IIS logs into Graylog with each field extracted

## Currently missing

I haven't handled `C:/Windows/System32/LogFiles/HTTPERR/*.log` yet.
https://support.microsoft.com/en-us/help/820729/error-logging-in-http-apis

# Configure IIS to get the logs in the correct format

Set the correct formatting of the log-files
```
%windir%\system32\inetsrv\appcmd.exe set config -section:system.applicationHost/sites -siteDefaults.logfile.logFormat:"W3C"
```

Add the necessary fields
```
%windir%\system32\inetsrv\appcmd.exe set config -section:system.applicationHost/sites -siteDefaults.logfile.logExtFileFlags:Date,Time,ClientIP,UserName,SiteName,ComputerName,ServerIP,Method,UriStem,UriQuery,HttpStatus,Win32Status,BytesSent,BytesRecv,TimeTaken,ServerPort,UserAgent,Cookie,Referer,ProtocolVersion,Host,HttpSubStatus
```

If you wish to change the path to where the logs are stored, edit the command below and run it
```
%windir%\system32\inetsrv\appcmd.exe set config -section:system.applicationHost/sites -siteDefaults.logfile.directory:"%SystemDrive%\inetpub\logs\LogFiles"
```

# GROK

```
%{TIMESTAMP_ISO8601:log_timestamp} %{WORD:S-SiteName} %{HOSTNAME:S-ComputerName} %{IPORHOST:S-IP} %{WORD:CS-Method} %{URIPATH:CS-URI-Stem} %{NOTSPACE:CS-URI-Query} %{NUMBER:S-Port} %{NOTSPACE:CS-Username} %{IPORHOST:C-IP} %{NOTSPACE:CS-Version} %{NOTSPACE:CS-UserAgent} %{NOTSPACE:CS-Cookie} %{NOTSPACE:CS-Referer} %{NOTSPACE:CS-Host} %{NUMBER:SC-Status} %{NUMBER:SC-SubStatus} %{NUMBER:SC-Win32-Status} %{NUMBER:SC-Bytes} %{NUMBER:CS-Bytes} %{NUMBER:Time-Taken}
```

|Swedish name|Field-name according to logfile|GROK
|-|-|-|
|Datum|date|(date+time is handled by the rule below)
|Tid|time|%{TIMESTAMP_ISO8601:log_timestamp}
|Servicenamn|s-sitename|%{WORD:S-SiteName}
|Servernamn|s-computername|%{HOSTNAME:S-ComputerName}
|IP-adress för server|s-ip|%{IPORHOST:S-IP}
|Metod|cs-method|%{WORD:CS-Method}
|URI-stam|cs-uri-stem|%{URIPATH:CS-URI-Stem}
|URI-fråga|cs-uri-query|%{NOTSPACE:CS-URI-Query}
|Server-port|s-port|%{NUMBER:S-Port}
|Användarnamn|cs-username|%{NOTSPACE:CS-Username}
|IP-adress för klient|c-ip|%{IPORHOST:C-IP}
|Protokollversion|cs-version|%{NOTSPACE:CS-Version}
|Användaragent|cs(User-Agent)|%{NOTSPACE:CS-UserAgent}
|Cookie|cs(Cookie)|%{NOTSPACE:CS-Cookie}
|Referenssida|cs(Referer)|%{NOTSPACE:CS-Referer}
|Värd|cs-host|%{NOTSPACE:CS-Host}
|Protokollstatus|sc-status|%{NUMBER:SC-Status}
|Understatus för protokoll|sc-substatus|%{NUMBER:SC-SubStatus}
|Win32-Status|sc-win32-status|%{NUMBER:SC-Win32-Status}
|Skickade bytes|sc-bytes|%{NUMBER:SC-Bytes}
|Mottagna bytes|cs-bytes|%{NUMBER:CS-Bytes}
|Tidsåtgång|time-taken|%{NUMBER:Time-Taken}
