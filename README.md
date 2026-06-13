Full packet capture tools like Wireshark and tcpdump record everything — which is often more than you need. When the question is purely about HTTP activity, **Httpry** gives you exactly what matters: hostnames, URLs, HTTP methods, status codes, and timestamps. Nothing more, nothing less.

This project walks through the complete Httpry workflow:

- Building Httpry from source on Kali Linux
- Running passive monitoring against a `.pcap` file
- Writing HTTP logs to disk and inspecting them
- Converting Httpry logs to **Common Log Format (CLF)** with a Perl script
- Running live HTTP capture on a network interface
- Repeating the CLF conversion on live capture output

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
|---|---|
| Httpry 0.1.8 | HTTP-layer packet sniffer |
| libpcap | Packet capture library (dependency) |
| Perl | CLF conversion scripting |
| curl | HTTP traffic generation for live testing |
| daemonlogger | pcap file source (from prior capture session) |
| Kali Linux | Operating environment |

---

## ⚙️ Installation

### 1. Update system packages

```bash
sudo apt-get update
```

![System update](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/z03stqqy2xfl3wortu35.png)

### 2. Install build dependencies

```bash
sudo apt install libpcap-dev make gcc
```

### 3. Clone and build from source

```bash
git clone https://github.com/jbittel/httpry.git
cd httpry
make
sudo make install
```

![Build and install](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nzl7nzn3jriq5jn2nnun.png)

### 4. Verify installation

```bash
httpry --version
which httpry
```

![Verify install](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/669u0urkcd5ea25ingq4.png)

---

## 📁 Part i — Locate the Captured pcap File

```bash
cd /var/log/daemonlogger
ls -lh capture.1777551302.pcap
```

![pcap file listing](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hmdomml2kp4iaaqo4r8t.png)

---

## 📡 Part ii — Passive Monitoring on pcap File

The `-r` flag reads from an existing pcap file instead of a live interface — no active capture needed.

```bash
sudo httpry -r capture.1777551302.pcap
```

Httpry extracts every HTTP transaction and prints it in a structured, tab-separated format showing timestamp, source/destination IPs, direction, method, host, URI, HTTP version, and status code.

![Passive monitoring output](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9t3ctqhfkhhlhqoff2dw.png)

---

## 📝 Part iii — Create and Inspect a Log File

```bash
sudo httpry -r capture.1777551302.pcap -o httpry_log.txt
cat httpry_log.txt
```

The log file starts with a comment header:

```
# httpry version 0.1.8
# Fields: timestamp,source-ip,dest-ip,direction,method,host,request-uri,http-version,status-code,reason-phrase
```

Followed by tab-separated data rows for every HTTP event found in the pcap.

![Log file content](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0hqwf4iqb2s7hifbexhr.png)

---

## 🔄 Part iv — Convert to Common Log Format

Httpry's native format is useful but not directly compatible with Apache/Nginx-style log parsers. This Perl script converts it to **Common Log Format (CLF)**:

```bash
cat > ~/log2clf.pl << 'EOF'
#!/usr/bin/perl
use strict;
use warnings;

while (<STDIN>) {
    chomp;
    next if /^#/;
    next if /^\s*$/;

    my @fields = split(/\t/, $_);
    next unless @fields >= 7;

    my ($timestamp, $direction, $src_ip, $dst_ip, $method, $host, $path, $proto) = @fields;

    next unless defined $method && $method =~ /GET|POST|HEAD/;

    my $url = "http://" . ($host // "-") . ($path // "/");
    my $time_fmt = $timestamp;

    print "$src_ip - - [$time_fmt] \"$method $url $proto\" - -\n";
}
EOF

cat ~/httpry_log.txt | perl ~/log2clf.pl > ~/httpry_clf.txt
cat ~/httpry_clf.txt
```

![CLF conversion output](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pu2jbp483n0vdtlp29a7.png)

---

## 🌐 Part v — Live Monitoring While Generating HTTP Traffic

**Terminal 1 — start Httpry on the active interface:**

```bash
sudo httpry -i eth0
```

**Terminal 2 — generate HTTP traffic:**

```bash
curl http://example.com
curl http://google.com
```

Press `Ctrl+C` to stop. Httpry prints a packet summary on shutdown.

![Live monitoring](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/t9dwsye30o50qydnxyyp.png)

![curl traffic generation](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pdud00o37ff8cvug60pm.png)

![Live capture output](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bo119mg0j5qhuqgmwfon.png)

---

## 💾 Part vi — Save Live Capture to File

```bash
sudo httpry -i eth0 -o live_capture.txt
cat live_capture.txt
```

![Live capture file](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zo7blh5bav66x0kw8piy.png)

---

## 🔄 Part vii — Convert Live Capture to CLF

```bash
cat > ~/log2clf.pl << 'EOF'
#!/usr/bin/perl
while (<STDIN>) {
    chomp;
    next if /^#/;
    next if /^\s*$/;
    my @f = split(/\t/, $_);
    next unless @f >= 7;
    my ($ts, $dir, $src, $dst, $method, $host, $path, $proto) = @f;
    next unless defined $method && $method =~ /GET|POST|HEAD/;
    $path //= "/";
    $proto //= "HTTP/1.1";
    print "$src - - [$ts] \"$method http://$host$path $proto\" - -\n";
}
EOF

cat ~/live_capture.txt | perl ~/log2clf.pl > ~/live_clf.txt
cat ~/live_clf.txt
```

**Sample CLF output:**

```
172.66.147.243 - - [2026-05-04 12:44:34.618] "GET http://example.com/ HTTP/1.1" - -
142.250.187.78 - - [2026-05-04 12:44:38.835] "GET http://google.com/ HTTP/1.1" - -
```

![Live CLF output](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/85plt2loeq4ybwirvhlk.png)

---

## ✅ Verification Checklist

| Check | Command | Expected Result |
|---|---|---|
| Httpry installed | `which httpry` | `/usr/sbin/httpry` |
| Passive read | `sudo httpry -r file.pcap` | HTTP lines + packets parsed count |
| Log file created | `cat httpry_log.txt` | Header + tab-separated data rows |
| CLF conversion | `cat httpry_clf.txt` | `IP - - [timestamp] "METHOD url proto" - -` |
| Live capture | `sudo httpry -i eth0` | Real-time HTTP events in terminal |
| Live log saved | `cat live_capture.txt` | Same format as passive log |
| Live CLF done | `cat live_clf.txt` | CLF lines from live session |

---

## ⚠️ Common Mistakes

| Mistake | Why It Happens | Fix |
|---|---|---|
| Running without `sudo` | Packet capture needs root | Always use `sudo httpry ...` |
| Wrong interface name | `eth0` vs `eth1` vs `ens33` varies | Check with `ip a` first |
| Perl script not filtering `#` lines | Log starts with comment headers | Add `next if /^#/;` to the loop |
| Empty CLF output | Tab-split field index off | Print raw `@fields` to verify alignment |
| `make install` man page error | Kali may lack `/usr/man/man1/` | Cosmetic only — binary installs correctly |
| No HTTP traffic in pcap | Capture was HTTPS only | Httpry only sees port 80 plaintext traffic |
| Log file empty | Forgot `-o filename.txt` | Always specify output file explicitly |

---

## 📖 Full Walkthrough

Read the complete step-by-step article on Dev.to:

👉 [Monitoring HTTP Traffic with Httpry on Kali Linux](https://dev.to/almahmudkhalif/lab-task-11-monitoring-http-traffic-with-httpry-on-kali-linux-passive-capture-live-monitoring-3f83)

---

## 🌐 Connect With Me

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?logo=linkedin)](https://www.linkedin.com/in/almahmudkhalif/)
[![Dev.to](https://img.shields.io/badge/Dev.to-Articles-black?logo=devdotto)](https://dev.to/almahmudkhalif/)
