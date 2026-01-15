---
title: "Using Fluent Bit on Windows to ship event logs to VictoriaLogs"
date: 2026-01-14T23:30:00-00:00
draft: false
---

[Windows install docs](https://docs.fluentbit.io/manual/installation/downloads/windows)

[Windows service setup](http://docs.fluentbit.io/manual/installation/downloads/windows#can-you-manage-fluent-bit-service-using-powershell)

Fluent Bit is designed solely as a low-overhead forwarding agent, primarily for logs (though it does have support for metrics as well), and has a wide ecosystem of "input" and "output" plugins that permit it to be used to collect logs or metrics from a wide variety of things, then ship said logs or metrics to a wide variety of data stores (e.g., in our case, it'll ship via a web request POSTing events to VictoriaLogs).

Compared to the similarly-named (and related) Fluentd:

- Fluent Bit is a lightweight data collector or forwarder.
- Fluentd is older, more feature-rich, and has more of a focus on parsing, aggregation, and routing. It has a substantially larger footprint. [Ref: documentation comparing the two](https://docs.fluentbit.io/manual/about/fluentd-and-fluent-bit).

Essentially, Fluent Bit is a focused, performant tool that does just what we want (collect logs!) On the Windows system I'm currently writing this with, Fluent Bit is using about 3 MB of memory to scrape my local event logs, queue them in a SQLite database on disk, then ship them to VictoriaLogs. [Historical benchmarks](https://www.outcoldsolutions.com/blog/2018-11-19-performance-collectord-fluentd-fluentbit/) have shown Fluent Bit doesn't scale particularly well to millions of events a minute, but does just fine in smaller environments.

My VictoriaLogs configuration is fairly standard. [See config](https://github.com/hpst3r/nixos/tree/1d3b2e7891ff8e0a538797ffc941bde57597e8fc/modules/services/monitoring/servers/victorialogs).

Relevant info:

- Server is listening on 443 for "victorialogs.lab.wporter.org" (thanks to NGINX)
- Using trusted SSL certificate on the server only (no mTLS)
- No authentication

I'll first go over getting this set up for a single Windows machine. Then, we'll see about getting Windows Event Forwarding configured (and setting up a server to ship forwarded logs to VictoriaLogs with Fluent Bit).

## Installing Fluent Bit on Windows

Quick PowerShell snippet:

```PowerShell
[string] $Version = 'fluent-bit-4.2.1-win64'
[string] $Archive = "$Version.zip"
[string] $TempArchivePath = "$env:TEMP\$Archive"
[string] $TempExtractPath = "$env:TEMP\$Version"
[string] $HashFilePath = "$TempArchivePath.sha256"
[string] $InstallPath = "C:\Program Files\fluent-bit"

Invoke-WebRequest `
  -Uri "https://packages.fluentbit.io/windows/$Archive" `
  -OutFile $TempArchivePath

Invoke-WebRequest `
  -Uri "https://packages.fluentbit.io/windows/$Archive.sha256" `
  -OutFile $HashFilePath # this is the checksum

# hashes for Windows do not currently match
# CI issue: https://github.com/fluent/fluent-bit/issues/11203
# if (-not ((Get-FileHash $TempArchivePath).Hash -eq (Get-Content $HashFilePath)) ) {
#   throw "Fatal: archive hash does not match provided checksum."
# }

# install the program
Expand-Archive `
  -Path $TempArchivePath `
  -DestinationPath $TempExtractPath

# move TEMP\fluent-bit-maj.min.rev-arch\ to Program Files\fluent-bit
Move-Item `
  -Path "$TempExtractPath\$Version" `
  -Destination $InstallPath

# register service
if (-not Get-Service fluent-bit) {
  New-Service fluent-bit `
    -BinaryPathName "`"$InstallPath\bin\fluent-bit.exe`" -c `"$InstallPath\conf\fluent-bit.conf`"" `
    -StartupType Automatic `
    -Description "This service runs Fluent Bit, a log collector that enables real-time processing and delivery of log data to centralized logging systems."
}

# start the service
Restart-Service fluent-bit
```

## Configuring Fluent Bit for one machine

Quick setter:

```powershell
$Config = @"
[SERVICE]
    # push (flush local cache) at 5s interval
    flush        5
    daemon       on
    # info level = you will see each 200 OK
    log_level    info
    # gc -wait (tail) this file to see what's going on!
    log_file     C:\ProgramData\fluent-bit\fluent-bit.log
    # external parsers file
    parsers_file parsers.conf
    # fluent bit can run a http server for stats on itself
    http_server  off
    http_listen  127.0.0.1
    http_port    2020
    storage.path C:\ProgramData\fluent-bit\storage
    storage.sync normal
    storage.checksum off
    storage.max_chunks_up 128

[INPUT]
    # use the newer winevtlog input plugin, not old winlog
    name                winevtlog
    # which log channels do you want to collect?
    channels            Security,System,Application
    # what local scrape interval?
    interval_sec        5
    # where should the cache go?
    db                  C:\ProgramData\fluent-bit\winevtlog-local.sqlite
    storage.type        filesystem
    read_existing_events false
    
    # these expose ALL metadata as individual JSON fields
    string_inserts      true
    render_event_as_xml false
    
    # use ANSI encoding if you have blank message issues on older versions of Windows
    use_ansi            false

# send to victorialogs via https using jsonline import fmt
[OUTPUT]
    name            http
    match           *
    host            victorialogs.lab.wporter.org
    port            443
    # use jsonline ingest endpoint. organize streams by Computer,(event)Channel
    uri             /insert/jsonline?_stream_fields=Computer,Channel&_msg_field=Message&_time_field=date
    format          json_lines
    json_date_format iso8601
    # you REALLY want to use compression with text.
    compress        gzip
    
    # tls configuration
    tls             On
    tls.verify      On
    
    # retry and buffering
    retry_limit     False
    storage.total_limit_size 5G
    
    # connection keepalive
    net.keepalive   On
    net.keepalive_idle_timeout 30
    
    # async only hits the configured DNS server,
    # in my env this means bypassing tailscale & failing to resolve
    net.dns.resolver LEGACY
"@
Set-Content "$InstallPath\conf\fluent-bit.conf" $Config
Restart-Service fluent-bit
```

To test the winevtlog input, or view event logs in realtime on the console, you can use this simpler configuration:

```PowerShell
$TestConfig = @"
[SERVICE]
    flush        1
    daemon       off
    log_level    info

[INPUT]
    name                winevtlog
    channels            Security,System,Application
    interval_sec        1
    db                  C:\ProgramData\fluent-bit\test-winlog.sqlite

[OUTPUT]
    name   stdout
    match  *
    format json
"@

Set-Content "C:\ProgramData\fluent-bit\test-config.conf" $TestConfig

# Run Fluent Bit in foreground
& "C:\Program Files\fluent-bit\bin\fluent-bit.exe" -c "C:\ProgramData\fluent-bit\test-config.conf"
```

This will dump all log entries to the console as received from the winevtlog input/parser. Here's an example event:

```json
[{"date":1766099534.632749,"ProviderName":"Microsoft-Windows-Time-Service","ProviderGuid":"{06EDCFEB-0FD0-4E53-ACCA-A6F8BBF81BCB}","Qualifiers":"","EventID":37,"Version":0,"Level":4,"Task":0,"Opcode":0,"Keywords":"0x8000000000000000","TimeCreated":"2025-12-18 18:12:13 -0500","EventRecordID":2922,"ActivityID":"","RelatedActivityID":"","ProcessID":1796,"ThreadID":3340,"Channel":"System","Computer":"PF1K1CRK","UserID":"NT AUTHORITY\\LOCAL SERVICE","Message":"The time provider NtpClient is currently receiving valid time data from time.windows.com,0x9 (ntp.m|0x9|0.0.0.0:123->168.61.215.74:123).","StringInserts":["time.windows.com,0x9 (ntp.m|0x9|0.0.0.0:123->168.61.215.74:123)"]}]
```

Prettified:

```json
[
   {
      "date":1766099534.632749,
      "ProviderName":"Microsoft-Windows-Time-Service",
      "ProviderGuid":"{06EDCFEB-0FD0-4E53-ACCA-A6F8BBF81BCB}",
      "Qualifiers":"",
      "EventID":37,
      "Version":0,
      "Level":4,
      "Task":0,
      "Opcode":0,
      "Keywords":"0x8000000000000000",
      "TimeCreated":"2025-12-18 18:12:13 -0500",
      "EventRecordID":2922,
      "ActivityID":"",
      "RelatedActivityID":"",
      "ProcessID":1796,
      "ThreadID":3340,
      "Channel":"System",
      "Computer":"PF1K1CRK",
      "UserID":"NT AUTHORITY\\LOCAL SERVICE",
      "Message":"The time provider NtpClient is currently receiving valid time data from time.windows.com,0x9 (ntp.m|0x9|0.0.0.0:123->168.61.215.74:123).",
      "StringInserts":[
         "time.windows.com,0x9 (ntp.m|0x9|0.0.0.0:123->168.61.215.74:123)"
      ]
   }
]
```

Very workable format!

Here's what we get from a `logsql` query against VictoriaLogs:

```txt
PS C:\Users\liam> irm 'https://vl.lab.wporter.org/select/logsql/query?query=Computer:=PF1K1CRK AND ProviderName:="Microsoft-Windows-Time-Service"'

_time         : 12/18/2025 11:12:17 PM
_stream_id    : 0000000000000000cef4d6efe75a5fc7748a19cb0d2e5c5a
_stream       : {Channel="System",Computer="PF1K1CRK"}
_msg          : The time provider NtpClient is currently receiving valid time data from time.windows.com,0x9
                (ntp.m|0x9|0.0.0.0:123->168.61.215.74:123).
Channel       : System
Computer      : PF1K1CRK
EventID       : 37
EventRecordID : 2922
Keywords      : 0x8000000000000000
Level         : 4
Opcode        : 0
ProcessID     : 1796
ProviderGuid  : {06EDCFEB-0FD0-4E53-ACCA-A6F8BBF81BCB}
ProviderName  : Microsoft-Windows-Time-Service
StringInserts : ["time.windows.com,0x9 (ntp.m|0x9|0.0.0.0:123->168.61.215.74:123)"]
Task          : 0
ThreadID      : 3340
TimeCreated   : 2025-12-18 18:12:13 -0500
UserID        : NT AUTHORITY\LOCAL SERVICE
Version       : 0
```
