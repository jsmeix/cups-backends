# cups-backends

Additional backends for CUPS for special use cases

## /usr/lib/cups/backend/pipe

CUPS wrapper backend "pipe" for printing to any program.
It forwards the print job data like a pipe into another command.

It gets called by the cupsd as:
```
pipe job-id user title copies options [file]
```

Queue setup synopsis:
```
lpadmin -p queue_name -v 'pipe:/path/to/command?option1=value1&-option2&value2' -E
```

The command is called with the specified options as:
```
/path/to/command option1=value1 -option2 value2
```

The original command line parameters (... job-id user ...)
are provided as environment variables PIPE_BACKEND_ARGV[0-6]

## /usr/lib/cups/backend/monitor

Generic CUPS wrapper backend "monitor"
that runs in parallel with the actual backend to monitor it.

It will continue to actively run in parallel as parent process
of the actual backend (that could be e.g. /usr/lib/cups/backend/socket)
so that the monitor backend can do whatever is needed to watch
and control the actual backend including whatever is needed when
the actual backend finished successfully or failed with an error.

The main difference compared to the 'beh' backend error handler is that
the monitor backend is not idle waiting until the actual backend finished.
The monitor backend keeps actively running in parallel so that it can do
whatever is needed to monitor and control the actual backend while it is running
(e.g. the monitor backend terminates the actual backend when it is running for too long).

The intent is that experienced users and system admins can and should adapt or extend
this generic CUPS backend monitor to make it work for their particular needs and use cases.
Therefore it is implemented in the native language for system administration:
As bash script.

It gets called by the cupsd as:
```
monitor job-id user title copies options [file]
```

Queue setup synopsis:
```
lpadmin -p queue_name -v 'monitor:/exit_code/attempts/delay/check/actual_backend_URI' -E
```

All parameters (exit_code, attempts, delay, check, actual_backend_URI)
must be specified and separated by single slash
(double slashes are possible inside actual_backend_URI).
Parameter values must be percent encoded,
in particular space as %20 and slash as %2F
e.g. via something like:
```
echo 'parameter value' | sed -e 's/ /%20/g' -e 's/\//%2F/g'
```

* exit_code:<br/>
When the actual backend succeeded monitor exits with exit code 0 but
'inherit' lets monitor exit with the exit code of the actual backend
if the actual backend had failed or was terminated after the maximum
number of times (attempts) to run the monitoring check command.
Otherwise monitor exits with the specified exit code either a number
or specified as CUPS_BACKEND_... variable name, see `man 7 backend`.

* attempts:<br/>
Maximum number of times the monitoring check command is run
as long as the actual backend is running (0 is unlimited times)
until the actual backend gets terminated if it did not finish before.
When attempts is 1 the monitoring check command is called
one second before the actual backend is started and terminated
one second after the backend had finished or was terminated.
Otherwise the first call of the monitoring check command happens
directly after the actual backend was started.

* delay:<br/>
Number of seconds between two calls of the monitoring check command.
When attempts is 1 delay specifies the number of seconds until
the actual backend gets terminated if it did not finish before.
One can set delay to 0 to run a quick monitoring check command
that should finish within a tenth of a second (or it gets terminated)
as often as possible and even both delay and attempts set to 0 works
for special cases but the latter results unlimited busy looping.

* check:<br/>
What is called to monitor the actual backend while it is running.
Either 'sleep' waits the number of seconds specified by delay
or a whole percent encoded command (or script) can be specified.
The monitoring check command should finish within delay seconds
otherwise it gets terminated (first via SIGTERM, later via SIGKILL).

* actual_backend_URI:<br/>
The whole verbatim device URI of the actual backend
e.g. as shown by `lpstat -v` or `lpinfo -v`.

### Usage example:

Run 'tcpdump' in parallel with the CUPS socket backend to monitor
what happens on the network while the socket backend is running.

Assume the actual backend URI of the current print queue is
```
socket://192.168.101.202:9100
```

The following tcpdump command could be used to monitor what happens
when that backend sends the print data to port 9100 at host 192.168.101.202
and store the tcpdump output in /tmp/monitor.pcap
```
tcpdump -w /tmp/monitor.pcap host 192.168.101.202 and port 9100
```

That tcpdump command must be percent encoded e.g. with
```
echo 'tcpdump -w /tmp/monitor.pcap host 192.168.101.202 and port 9100' | sed -e 's/ /%20/g' -e 's/\//%2F/g'
```
that results the percent encoded monitoring check command
```
tcpdump%20-w%20%2Ftmp%2Fmonitor.pcap%20host%20192.168.101.202%20and%20port%209100
```

That percent encoded monitoring check command is finally used
as the check parameter of the monitor backend device URI
so that the current socket backend can be wrapped with the monitor backend
for example with a command like
```
lpadmin -p queue_name -v 'monitor:/CUPS_BACKEND_CANCEL/1/9/tcpdump%20-w%20%2Ftmp%2Fmonitor.pcap%20host%20192.168.101.202%20and%20port%209100/socket://192.168.101.202:9100' -E
```

This lets the monitor backend start the tcpdump command one second
before the socket backend is run and then wait up to 9 seconds for the
socket backend to finish (afterwards the socket backend gets terminated).
One second after the socked backend exited the tcpdump command gets terminated.
The return code of the monitor backend is always CUPS_BACKEND_CANCEL where
print jobs get cancelled in case of an error which avoids that the queue
becomes disabled so that one can "just proceed" when doing various tests.

By default CUPS backends are run by the cupsd as user 'lp' but
usually 'lp' is not allowed to dump arbitrary network traffic
so that the monitor backend would have to be run as 'root'.
The cupsd runs backends as 'root' when nobody except
the owner has permissions so a command like
```
chmod 0700 /usr/lib/cups/backend/monitor
```
could be needed to run the monitor backend as 'root'.

## Background information

See the openSUSE Wiki article
"SDB:Using Your Own Backends to Print with CUPS" at
https://en.opensuse.org/SDB:Using_Your_Own_Backends_to_Print_with_CUPS
