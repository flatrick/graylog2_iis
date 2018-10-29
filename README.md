# graylog2_iis
Configuration to get IIS logs into Graylog with each field extracted

## Currently missing

- I haven't handled `C:/Windows/System32/LogFiles/HTTPERR/*.log` yet. 
- - [Error logging in HTTP APIs](https://support.microsoft.com/en-us/help/820729/error-logging-in-http-apis) 
- Also missing a way to handle incorrect timezone for timestamps (IIS always stores datetime in UTC).
- - [Graylog - Functions - parse_date()](http://docs.graylog.org/en/2.2/pages/pipelines/functions.html#parse-date) 
- - [Time Zone](https://www.joda.org/joda-time/timezones.html) 

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

# Graylog

## Collector

### Beats Input

|Setting|Value
|-|-|
|Path to logfile|`['C:\inetpub\logs\LogFiles\*\*.log']`
|Encoding|`utf-8`
|Type of input file|`iis`
|Lines that you want Filebeat to exclude|`['^#']`

## Inputs

### Manage extractors

|Setting|Choice
|-|-|
|Extractor type|Grok pattern
|Source field|message
|Grok pattern|`%{TIMESTAMP_ISO8601:log_timestamp} %{WORD:S-SiteName} %{HOSTNAME:S-ComputerName} %{IPORHOST:S-IP} %{WORD:CS-Method} %{URIPATH:CS-URI-Stem} %{NOTSPACE:CS-URI-Query} %{NUMBER:S-Port} %{NOTSPACE:CS-Username} %{IPORHOST:C-IP} %{NOTSPACE:CS-Version} %{NOTSPACE:CS-UserAgent} %{NOTSPACE:CS-Cookie} %{NOTSPACE:CS-Referer} %{NOTSPACE:CS-Host} %{NUMBER:SC-Status} %{NUMBER:SC-SubStatus} %{NUMBER:SC-Win32-Status} %{NUMBER:SC-Bytes} %{NUMBER:CS-Bytes} %{NUMBER:Time-Taken}`
|Condition|Only attempt extraction if field matches regular expression
|Field matches regular expression|`W3SVC`
|Extraction strategy|Copy
|Extractor title|IIS-WC3 Full

## Pipelines

### Correct timestamps

*For this rule to work, __Pipeline Processor__ must run after __Message Filter Chain__ (edit this under __System - Configuration__ by pressing __Update__ below the table showing the current order and move __Pipeline Processor__ to after __Message Filter Chain__*

This rule will 
- read all messages that are of type __iis__ and contains the field __log_timestamp__
- parse the field to add it's correct timezone (UTC)
- save the field with "Europe/Stockholm" as timezone in the format: yyyy-MM-dd HH:mm:ss +0100 (non-DST, +0200 when DST is in effect)

```
rule "correct IIS-log timestamp"
when
    contains(
            value: to_string($message.type), 
            search: "iis",
            ignore_case: true)
    AND has_field(field: "log_timestamp")
then
    let new_date = format_date(
                        value: parse_date(
                            value: to_string($message.log_timestamp),
                            pattern: "yyyy-MM-dd HH:mm:ss",
                            timezone: "UTC"),
                        format: "yyyy-MM-dd HH:mm:ss Z",
                        timezone: "Europe/Stockholm");
    set_field("log_timestamp", new_date);
end
```

# Info

 http://www.nsi.bg/nrnm/Help/iisHelp/iis/htm/core/iiintlg.htm 
 https://docs.microsoft.com/en-us/windows/desktop/http/w3c-logging 

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
