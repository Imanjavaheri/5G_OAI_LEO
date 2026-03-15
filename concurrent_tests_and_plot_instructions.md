# Concurrent Test + Plot Workflow (TCP CUBIC/RENO/BBR + UDP)

This guide provides a clean, repeatable way to:
1. Run four `iperf3` load tests (TCP CUBIC, TCP RENO, TCP BBR, and UDP).
2. Capture simultaneous ping latency during each load test.
3. Generate one combined plot for throughput, congestion window (TCP), and RTT.

---

## Prerequisites

- Docker container `oai-ext-dn` is running.
- `iperf3` server destination is reachable at `192.168.70.135`.
- Client bind/interface IP is `10.0.0.24`.
- Python 3 with `matplotlib` installed.

---

## 1) Clean Previous Output

```bash
rm -f /home/antlab/my_data.txt /home/antlab/ping_*.txt
```

---

## 2) Start `iperf3` Server (separate terminal)

Run this in a second terminal and keep it running:

```bash
docker exec -it oai-ext-dn iperf3 -s
```

---

## 3) Run the 4 Load Tests (main terminal)

> Execute the blocks **in order**.
> Wait until each 20-second test finishes before starting the next block.

### 3.1 TCP CUBIC

```bash
echo "=== TCP CUBIC TEST ===" >> /home/antlab/my_data.txt
docker exec oai-ext-dn ping -c 20 10.0.0.24 > /home/antlab/ping_cubic.txt &
iperf3 -c 192.168.70.135 -B 10.0.0.24 -C cubic -t 20 >> /home/antlab/my_data.txt
```

### 3.2 TCP RENO

```bash
echo "=== TCP RENO TEST ===" >> /home/antlab/my_data.txt
docker exec oai-ext-dn ping -c 20 10.0.0.24 > /home/antlab/ping_reno.txt &
iperf3 -c 192.168.70.135 -B 10.0.0.24 -C reno -t 20 >> /home/antlab/my_data.txt
```

### 3.3 TCP BBR

```bash
echo "=== TCP BBR TEST ===" >> /home/antlab/my_data.txt
docker exec oai-ext-dn ping -c 20 10.0.0.24 > /home/antlab/ping_bbr.txt &
iperf3 -c 192.168.70.135 -B 10.0.0.24 -C bbr -t 20 >> /home/antlab/my_data.txt
```

### 3.4 UDP stress test

This step intentionally pushes UDP at `20M` to stress the 25 PRB satellite link and expose latency inflation (bufferbloat behavior).

```bash
echo "=== UDP TEST ===" >> /home/antlab/my_data.txt
docker exec oai-ext-dn ping -c 20 10.0.0.24 > /home/antlab/ping_udp.txt &
iperf3 -c 192.168.70.135 -B 10.0.0.24 -u -b 20M -t 20 >> /home/antlab/my_data.txt
```

---

## 4) Replace `plot_tcp.py` with the script below

```python
import matplotlib.pyplot as plt
import re
import os

iperf_file = "/home/antlab/my_data.txt"


def parse_iperf(filename):
    data = {
        'CUBIC': {'time': [], 'tput': [], 'cwnd': []},
        'RENO':  {'time': [], 'tput': [], 'cwnd': []},
        'BBR':   {'time': [], 'tput': [], 'cwnd': []},
        'UDP':   {'time': [], 'tput': []}
    }

    current_algo = None
    if not os.path.exists(filename):
        return data

    with open(filename, 'r') as f:
        for line in f:
            if "=== TCP CUBIC TEST ===" in line:
                current_algo = 'CUBIC'
            elif "=== TCP RENO TEST ===" in line:
                current_algo = 'RENO'
            elif "=== TCP BBR TEST ===" in line:
                current_algo = 'BBR'
            elif "=== UDP TEST ===" in line:
                current_algo = 'UDP'

            if current_algo in ['CUBIC', 'RENO', 'BBR']:
                match = re.search(
                    r'\]\s+([\d\.]+)-([\d\.]+)\s+sec\s+[\d\.]+\s+[KMG]?Bytes\s+'
                    r'([\d\.]+)\s+([KMG]bits/sec)\s+\d+\s+([\d\.]+)\s+([KMG]?Bytes)',
                    line,
                )
                if match:
                    start_time = float(match.group(1))
                    tput_val = float(match.group(3))
                    if match.group(4) == 'Kbits/sec':
                        tput_val /= 1000

                    cwnd_val = float(match.group(5))
                    if match.group(6) == 'MBytes':
                        cwnd_val *= 1024
                    elif match.group(6) == 'Bytes':
                        cwnd_val /= 1024

                    data[current_algo]['time'].append(start_time)
                    data[current_algo]['tput'].append(tput_val)
                    data[current_algo]['cwnd'].append(cwnd_val)

            elif current_algo == 'UDP':
                match = re.search(
                    r'\]\s+([\d\.]+)-([\d\.]+)\s+sec\s+[\d\.]+\s+[KMG]?Bytes\s+'
                    r'([\d\.]+)\s+([KMG]bits/sec)',
                    line,
                )
                if match:
                    start_time = float(match.group(1))
                    tput_val = float(match.group(3))
                    if match.group(4) == 'Kbits/sec':
                        tput_val /= 1000

                    data['UDP']['time'].append(start_time)
                    data['UDP']['tput'].append(tput_val)

    return data


def parse_ping(filename):
    times = []
    if os.path.exists(filename):
        with open(filename, 'r') as f:
            for line in f:
                match = re.search(r'time=([\d\.]+)\s*ms', line)
                if match:
                    times.append(float(match.group(1)))
    return times


# Parse all data
iperf_data = parse_iperf(iperf_file)
ping_cubic = parse_ping("/home/antlab/ping_cubic.txt")
ping_reno = parse_ping("/home/antlab/ping_reno.txt")
ping_bbr = parse_ping("/home/antlab/ping_bbr.txt")
ping_udp = parse_ping("/home/antlab/ping_udp.txt")

# Create figure
fig, (ax1, ax2, ax3) = plt.subplots(3, 1, figsize=(10, 14))
fig.suptitle('5G Satellite Link: TCP vs UDP & Bufferbloat Latency', fontsize=16)

colors = {'CUBIC': 'blue', 'RENO': 'red', 'BBR': 'green', 'UDP': 'orange'}

# 1) Throughput
for algo in ['CUBIC', 'RENO', 'BBR', 'UDP']:
    if iperf_data[algo]['time']:
        style = '--' if algo == 'UDP' else '-'
        ax1.plot(
            iperf_data[algo]['time'],
            iperf_data[algo]['tput'],
            label=algo,
            color=colors[algo],
            linestyle=style,
            marker='o',
            markersize=3,
        )
ax1.set_ylabel('Throughput (Mbps)')
ax1.set_title('Throughput over Time')
ax1.grid(True, linestyle='--', alpha=0.7)
ax1.legend()

# 2) TCP congestion window
for algo in ['CUBIC', 'RENO', 'BBR']:
    if iperf_data[algo]['time']:
        ax2.plot(
            iperf_data[algo]['time'],
            iperf_data[algo]['cwnd'],
            label=algo,
            color=colors[algo],
            marker='s',
            markersize=3,
        )
ax2.set_ylabel('Congestion Window (KBytes)')
ax2.set_title('TCP Congestion Window (cwnd)')
ax2.grid(True, linestyle='--', alpha=0.7)
ax2.legend()

# 3) RTT under load (bufferbloat view)
if ping_cubic:
    ax3.plot(range(1, len(ping_cubic) + 1), ping_cubic, label='CUBIC Load RTT', color='blue', marker='x')
if ping_reno:
    ax3.plot(range(1, len(ping_reno) + 1), ping_reno, label='RENO Load RTT', color='red', marker='x')
if ping_bbr:
    ax3.plot(range(1, len(ping_bbr) + 1), ping_bbr, label='BBR Load RTT', color='green', marker='x')
if ping_udp:
    ax3.plot(range(1, len(ping_udp) + 1), ping_udp, label='UDP Load RTT', color='orange', marker='o', linestyle='--')

ax3.set_xlabel('Ping Sequence Number')
ax3.set_ylabel('RTT Delay (ms)')
ax3.set_title('Real-time Latency During Downloads (Bufferbloat Check)')
ax3.grid(True, linestyle='--', alpha=0.7)
ax3.legend()

plt.tight_layout(rect=[0, 0.03, 1, 0.95])
output_file = "/home/antlab/tcp_udp_satellite_results.png"
plt.savefig(output_file, dpi=300)
print(f"Graph saved as {output_file}")
```

---

## 5) Generate the Plot

```bash
python3 /home/antlab/plot_tcp.py
```

Expected output image:

- `/home/antlab/tcp_udp_satellite_results.png`
