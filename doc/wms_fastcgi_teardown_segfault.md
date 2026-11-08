# WMS (wms.cgi) segfault at FastCGI process exit: investigation and fixes

Production investigation of the recurring `wms.cgi` SIGSEGV (mod_fcgid `get signal 11`), the host root-disk exhaustion it caused, and the configuration changes that address both. Kept verbatim from the operational report.

```text
ONEARTH WMS (PROD) - HOST SITUATION REPORT
Prepared 2026-11-06 ~05:30 UTC from read-only SSM Run Command output and
systemd journal data supplied by O. Pariser; analysis by Devin.

=====================================================================
1. SEVERITY: CRITICAL - prod ECS host is about to run out of disk
=====================================================================
Host: ip-10-13-21-169.us-west-2.compute.internal (ECS EC2 instance running
the onearth-wms task; 95 GB RAM, 32 GB root volume, up since 2026-11-03 17:02 UTC)

  Root disk (/)            32 GB, 99 % used, 370 MB free   (was 97 %/1.2 GB free
                                                           40 minutes earlier)
  Load average             18 (1 min) / 17 / 16
  wms.cgi segfaults        ~250-430 per hour, 10,828 since 2026-11-04 11:49 UTC
                           (kernel log); 33 in the first 3 minutes of 05:00 UTC
  systemd-coredump workers 10 running at 05:03 UTC, 342 running at ~05:25 UTC
  Core store               /var/lib/systemd/coredump empty; every coredumpctl
                           entry marked "missing"

Where the disk went (du vs df):
  /var/lib/docker   14 GB
  /var/log/journal  355 MB
  /var/log          41 MB
  /tmp              80 KB
  accounted         ~14.4 GB      vs   32 GB used by df
  => ~17 GB is held by DELETED-BUT-OPEN core files: each of the 342 stuck
     systemd-coredump workers still has a vacuumed core open. That space is
     invisible to du and is not released until the workers exit.

What happens when / reaches 100 %:
  - docker cannot write container layers or logs -> onearth-wms task fails
    health checks and is restarted or stopped; ECS may not be able to start a
    replacement on this host.
  - awslogs agent cannot buffer -> CloudWatch log gap (blind spot during the
    outage).
  - Every further crash (several per minute) spawns another coredump worker
    that competes for the remaining space and CPU.
Time to full at the observed rate: minutes to a few hours, not days.

Client impact so far: none visible. Crashes occur after the last response has
been delivered (process teardown), so no 5xx are produced by them. The risk is
the host, not individual requests.

=====================================================================
2. ROOT CAUSE (confirmed by symbolized stack trace from the prod journal)
=====================================================================
Kernel: wms.cgi[...]: segfault at <heap addr> ip ... error 4 in libc.so.6
Offset is identical in every record: libc.so.6+0x44b6d = getenv+109.

Stack (systemd-coredump journal entry, symbols from glibc-2.39-145.el10_2 and
gibs-gdal-3.13.2-1.el10 binaries):
  #0 getenv+109                           libc.so.6
  #1 CPLGetConfigOption+250               libgdal.so.39.3.13.2  (reads CPL_DEBUG)
  #2 CPLvDebug+444
  #3 CPLDebug+162                         ("GDAL: GDALClose(<GDAL_WMS>...)")
  #4 GDALDataset::~GDALDataset()+1075
  #5 GDALWMSDataset::~GDALWMSDataset()+21
  #6 GDALCloseEx+67
  #7 msGDALCleanup                        libmapserver.so.8.6.5
  #8 msCleanup
  #9 main                                 wms.cgi (mapserv 8.6.5), after the
                                           final FCGI_Accept() returned -1
Mechanism:
  mapserv's FastCGI loop is  while (FCGI_Accept() >= 0) {...}; msCleanup();
  libfcgi's FCGI_Accept() sets environ to the request's parameter array and
  frees that array at the start of the NEXT call. When mod_fcgid closes the
  socket (process retirement: 50th request, FcgidProcessLifetime, graceful
  restart at every :00 logrotate) the next accept fails and returns -1 WITHOUT
  restoring environ. msCleanup() then closes the GDAL_WMS datasets that
  MapServer kept open for the life of the process (CLOSE_CONNECTION=DEFER
  default for raster layers); each destructor calls CPLDebug -> getenv(),
  which walks the freed environment. It faults when the heap has been trimmed
  below the stale strings, which is the normal case for a process that frees
  hundreds of MB of deferred datasets at exit.
  => A MapServer/libfcgi use-after-free. GDAL is only the caller of getenv.
  => Not triggered by any request; the BBOX/epsilon/GOES suspects are cleared.

Contributing factor (measured in SIT): with the default deferred close, each
wms.cgi grows ~4 MB per distinct GetMap (99 -> 287 MB over 45 requests), so
retiring processes are large and the teardown frees a lot -> fault is
near-certain. With PROCESSING "CLOSE_CONNECTION=NORMAL" growth is ~0.3 MB per
request and nothing is left for msGDALCleanup() to close.

=====================================================================
3. RECOMMENDED STEPS, IN ORDER
=====================================================================
--- A. NOW (minutes): stop the bleeding on the prod host (SSM Run Command, root)

A1. Stop writing cores and release the 17 GB held by stuck workers:
    mkdir -p /etc/systemd/coredump.conf.d && printf '[Coredump]\nStorage=none\nProcessSizeMax=0\n' > /etc/systemd/coredump.conf.d/50-no-cores.conf && cat /etc/systemd/coredump.conf.d/50-no-cores.conf; systemctl stop 'systemd-coredump@*'; sleep 5; echo workers=$(systemctl list-units 'systemd-coredump@*' --no-legend | wc -l); df -h / | tail -1; true
    Expected: workers=0 (or a few that exit immediately), df back to ~15 GB used.
    Safe: the dumped processes are already dead; nothing user-facing stops.
    ProcessSizeMax=0 makes systemd-coredump discard new cores unread, so the
    ~300 crashes/hour no longer cost disk, CPU or journal space. The stack
    trace we need is already captured (section 2).

A2. If df does not drop, find who still holds the deleted files:
    ls -la /var/lib/systemd/coredump | head; find /proc/[0-9]*/fd -lname '*(deleted)' 2>/dev/null | wc -l; true

A3. Verify after the next :00 retirement wave (one hour later):
    coredumpctl list --since "-1 hour" --no-pager 2>&1 | tail -3; echo segfaults=$(journalctl -k --since "-1 hour" --no-pager | grep -c segfault); echo workers=$(systemctl list-units 'systemd-coredump@*' --no-legend | wc -l); df -h / | tail -1; true
    Expected: segfaults still > 0 (bug not fixed yet), no new coredumpctl
    entries, workers=0, free space stable.

--- B. THIS WEEK: make A durable and reduce the crash rate without a rebuild

B1. The drop-in is lost when the ASG replaces the instance. Put the same two
    lines into the ECS launch-template user-data, or set on the onearth-wms
    container definition (terraform/onearth-tf ecs.tf / ecs2.tf):
      ulimits = [{ name = "core", softLimit = 0, hardLimit = 0 }]
    so wms.cgi never produces a core in the first place. Either one; the
    task-definition route also covers SIT/dev and survives AMI changes.

B2. Mapfile workaround (config-only, no C rebuild): emit
      PROCESSING "CLOSE_CONNECTION=NORMAL"
    for raster (GDAL_WMS) layers in docker/wms_service/oe2_wms_configure.py.
    Measured in SIT: RSS growth 4.2 -> 0.3 MB/request, no latency change
    (45 renders: 26 s vs 25 s). With no deferred datasets, msGDALCleanup()
    has nothing to close, so the crashing stack cannot occur. It does not
    remove the dangling environ pointer, so a residual (much lower) crash
    rate from other getenv() callers in GDALDestroy() is possible.

--- C. NEXT RELEASE: fix the bug

C1. Patch mapserv.c (MapServer 8.6.5, built from tarball in
    docker/wms_service/Dockerfile and docker/test/Dockerfile): save environ
    before the FCGI_Accept() loop and restore it before msCleanup().
    Draft patch: mapserv-environ.patch (~20 lines, 3 hunks); apply with
    `patch -p1` between `tar xf` and `cmake`. First compile happens in CI.
C2. Report upstream to MapServer (github.com/MapServer/MapServer) with the
    stack in section 2 and the fcgi_stdio.c note that environ must not be
    used after a failed FCGI_Accept(); offer the patch. Optionally also to
    fcgi2 (FCGI_Accept could restore environ on failure).
C3. Keep FcgidMaxRequestsPerProcess at 50 until B2/C1 are in prod; then it is
    a pure tuning knob (100-500 feasible, ~0.3 MB/request growth). Do NOT
    raise it before B2: at ~4 MB/request, 500 means ~2 GB per process.
C4. After C1 ships, confirm on the host: journalctl -k | grep -c segfault
    over an hour should be 0, and the mod_fcgid "get signal 11" lines should
    disappear from the WMS CloudWatch stream.

--- D. Monitoring to add (so this is caught next time)
  - CloudWatch alarm on the ECS host root-volume free space (< 20 %).
  - CloudWatch Logs metric filter on the WMS stream for "signal 11" and
    "possible coredump".
  - Optional: metric on `systemctl list-units 'systemd-coredump@*' | wc -l`
    via the CloudWatch agent (any value > 5 means cores are piling up).

=====================================================================
4. EVIDENCE INDEX
=====================================================================
  - Prod ci/*.zip logs (commit 8b7ba118): 16 mod_fcgid "get signal 11" lines,
    Nov 3 17:30-19:00 UTC; no request correlation. (wms_segfault_report.md §1-4)
  - Prod host journal: coredumpctl list (all "missing"), kernel segfault lines
    with constant libc offset, full stack trace for PID 368168. (§6, §8)
  - Offline symbolization: glibc-2.39-145.el10_2 (almalinux:10),
    gibs-gdal-3.13.2-1.el10 RPM. (§7, §9)
  - SIT measurements: RSS slope with/without CLOSE_CONNECTION=NORMAL,
    GDAL_CACHEMAX disproved, 0 crashes in 53 h of low traffic. (runbook)

=====================================================================
5. UPDATE 2026-11-06 ~06:30 UTC - mitigation applied, second disk leak found
=====================================================================
Applied on the prod host (Run Command):
  /etc/systemd/coredump.conf.d/50-no-cores.conf  (Storage=none, ProcessSizeMax=0)
  systemctl stop 'systemd-coredump@*'            (342 stuck workers ended)
Result: 370 MB free -> 6.8 GB free (79-80 %), workers=0 thereafter,
segfaults continue at ~300/h but each new coredump worker exits immediately.
Host is no longer at risk of filling overnight.

Second leak (independent of the segfaults):
  du -x / = 18 GB vs df used = 26 GB. The ~8 GB gap is deleted-but-open
  httpd error logs inside the onearth-wms container:
    /var/log/httpd/error_log-2026110603  571 MB  held by 7 httpd children
    /var/log/httpd/error_log-2026110510  480 MB  held by 6 httpd children
    /var/log/httpd/error_log-2026110418  445 MB  held by 2+ httpd children
    ... (~10 generations)
  Cause: after each hourly `httpd -k graceful` (logrotate postrotate), some
  old-generation worker children never exit and keep the previous hour's
  error_log open; logrotate deletes the file later, blocks stay allocated.
  Aggravating factor: the WMS error_log is 445-571 MB PER HOUR (mod_fcgid
  warnings, broken pipes, mapserver/GDAL messages) - review LogLevel / noise.

Release (no process impact; files are already deleted):
  n=0; for l in /proc/[0-9]*/fd/*; do case "$(readlink "$l" 2>/dev/null)" in /var/log/httpd/*"(deleted)") : > "$l" && n=$((n+1));; esac; done; echo truncated=$n; df -h / | tail -1; true
Diagnose lingering children:
  ps -o pid,ppid,etimes,stat,wchan:25,args -p <pids>  -> blocked in pipe/socket
  read = stuck FastCGI/client connection (fix: bounded Timeout /
  GracefulShutdownTimeout); idle in poll = keepalive (fix: KeepAliveTimeout).
Permanent options (container config, docker/wms_service + docker/logrotate):
  - MaxConnectionsPerChild so a child cannot live across many rotations, or
  - logrotate `copytruncate` for httpd logs so a lingering child cannot pin a
    deleted file, and
  - reduce error_log volume (LogLevel warn, silence known-noise messages).
Add to section 3-D monitoring: alert when df-used minus du-used on / exceeds
2 GB (deleted-open files), and on error_log growth > 100 MB/h.

=====================================================================
6. UPDATE ~07:00 UTC - crash trigger pinned to debug logging config
=====================================================================
Prod error_log top messages are GDAL debug output ("WMS: ReadBlockFromCache",
"GDAL: Selecting overview", "HTTP: Request [n] http://onearth-tile-services/...").
The prod image's mapfile headers (pre PR #1) contain:
    Debug 3
    CONFIG "CPL_DEBUG" "ON"
    CONFIG "PROJ_DEBUG" "ON"
=> 0.45-0.57 GB of error_log per hour, and the crash path:
   disassembly of libgdal.so.39.3.13.2 shows CPLvDebug+444 (frame #2) is the
   return from CPLGetConfigOption("CPL_TIMESTAMP","NO"), executed only after
   CPL_DEBUG tested true. CPL_TIMESTAMP is not in GDAL's option table, so the
   lookup falls through to getenv() on the freed FastCGI environ.
   Each exiting wms.cgi logs "GDALClose(<GDAL_WMS>)" per deferred dataset, so
   every retirement crashes (~300/h observed).

PR #1 (2.10.1-1, in SIT) sets Debug 0 but REMOVED the CONFIG "CPL_DEBUG"
lines and passes CPL_DEBUG=OFF only via FcgidInitialEnv. GDAL lookup order is
thread table -> global table -> getenv(); with nothing in the table the first
lookup (CPL_DEBUG itself) calls getenv() on the dangling pointer. The crash
path is therefore still reachable in SIT; low traffic explains 0 crashes.

Config-only fix set (replaces the need for the mapserv.c patch):
  1. Headers keep CONFIG "CPL_DEBUG" "OFF" and CONFIG "PROJ_DEBUG" "OFF";
     start_wms_service.sh inserts them when absent (real headers come from
     the config bucket, not sample_configs). With CPL_DEBUG in the table,
     CPLvDebug returns before any getenv().
  2. PROCESSING "CLOSE_CONNECTION=NORMAL" on raster layers (memory, and no
     deferred datasets to close at exit).
  3. logrotate: copytruncate instead of postrotate "httpd -k graceful":
     removes the hourly graceful that creates stale httpd children
     (~95 of 110 children are old generations, 4 threads each parked in
     futex_lock_pi = deadlocked on graceful exit), the pinned deleted logs,
     and the :00 mass retirement of wms.cgi.
  4. Optional backstop: MaxConnectionsPerChild; optional: mapserv.c environ
     restore patch (true root cause, upstream).
```
