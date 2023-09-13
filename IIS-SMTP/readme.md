Sentinel IIS SMTP noodles
=========================
Noodling about with SMTP logging

## Tools provided

This folder contains:
- A Data Collection Rule to pull IIS SMTP logs from C:\Windows\System32\Logfiles\SMTPSVC[1-9]\e*.log
  - DCR directs logs to a custom table called SMTPSVC_CL, without any source transformation
- A basic parser function for Log Analytics (SMTPSVC) which splits and parses SMTPSVC_CL into named columns

This should be enough to get as much as the SMTP service is prepared to offer you, and can then be used for correlation and enrichment with more useful and interesting tables in Sentinel (IdentityInfo, ThreatIntelligenceIndicators, any mail tables you're ingesting...)

## Logging with IIS SMTP

IIS SMTP uses IIS 6 compatibility mode on IIS 7 and later.

- it runs in INETINFO.EXE (which is not required for 7+)
- it uses the Metabase emulator (and metabase.xml) for configuration storage

The default logging configuration is watery at best, so needs to be maxed out!

Once maxed out, you don't get amazing forensic levels of detail even at max settings, but it's enough to start...

## Setting IIS SMTP "sites" to log

The ADSUTIL.VBS admin scripts are not installed by default, but can be installed from Server Manager (IIS 6 Management Scripts).

Once installed, you can check current logging flags with:

`cscript adsutil.vbs get /smtpsvc/1/LogExtFileFlags`

And set all available flags with 

`cscript adsutil.vbs Set /smtpsvc/1/LogExtFileFlags 4194303`

- Or by editing `C:\Windows\System32\Inetsrv\metabase.xml` directly.
- Or by editing each SMTP Virtual Server's Logging properties in the legacy IIS Manager through the GUI.

Bulk enablement is left as an exercise to the reader.

## Future ideas
- Some fields look useless and might be tuned out at the source, but "enable all log tickboxes" is very simple guidance
- Source transformation to some extent
- Integrating deployment packages


