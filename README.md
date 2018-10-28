# graylog2_iis
Configuration to get IIS logs into Graylog with each field extracted


# Configure IIS to get the logs in the correct format

Set the correct formatting of the log-files
```
%windir%\system32\inetsrv\appcmd.exe set config -section:system.applicationHost/sites -siteDefaults.logfile.logFormat:"W3C"
```

Add the necessary fields
```
%windir%\system32\inetsrv\appcmd.exe  set config -section:system.applicationHost/sites -siteDefaults.logfile.logExtFileFlags:Date,Time,ClientIP,UserName,SiteName,ComputerName,ServerIP,Method,UriStem,UriQuery,HttpStatus,Win32Status,BytesSent,BytesRecv,TimeTaken,ServerPort,UserAgent,Cookie,Referer,ProtocolVersion,Host,HttpSubStatus
```

If you wish to change the path to where the logs are stored, edit the command below and run it
```
%windir%\system32\inetsrv\appcmd.exe  set config -section:system.applicationHost/sites -siteDefaults.logfile.directory:"%SystemDrive%\inetpub\logs\LogFiles"
```

# GROK

```
%{TIMESTAMP_ISO8601:log_timestamp} %{WORD:S-SiteName} %{NOTSPACE:S-ComputerName} %{IPORHOST:S-IP} %{WORD:CS-Method} %{URIPATH:CS-URI-Stem} (?:-|\"%{URIPATH:CS-URI-Query}\") %{NUMBER:S-Port} %{NOTSPACE:CS-Username} %{IPORHOST:C-IP} %{NOTSPACE:CS-Version} %{NOTSPACE:CS-UserAgent} %{NOTSPACE:CS-Cookie} %{NOTSPACE:CS-Referer} %{NOTSPACE:CS-Host} %{NUMBER:SC-Status} %{NUMBER:SC-SubStatus} %{NUMBER:SC-Win32-Status} %{NUMBER:SC-Bytes} %{NUMBER:CS-Bytes} %{NUMBER:Time-Taken}
```
