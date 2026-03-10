# 5G_OAI_LEO

## Fresh Start (Optional)

If you want to start completely fresh so the Python graph is not affected by old data:

```bash
rm /home/antlab/my_data.txt /home/antlab/ping_data.txt
```

## Step 1: The Ping Test (Terminal 1)

This forces the ping from the external internet container, through the 5G Core, over the satellite, to the UE.

```bash
docker exec oai-ext-dn ping -c 50 10.0.0.24 > /home/antlab/ping_data.txt
```

Wait about 50 seconds for this to finish in the background.

## Step 2: Start the iperf3 Server (Terminal 2)

Open a fresh terminal to start the receiver inside the external internet container so it listens for the UE's uploads.

```bash
docker exec -it oai-ext-dn iperf3 -s
```

Leave this terminal running. It should say `Server listening on 5201`.

## Step 3: Run the TCP Comparisons (Terminal 3)

Open a third terminal. You will act as the UE, pushing data up the satellite link to the internet container.
Run these exactly in order, and wait for each 20-second test to finish before running the next.

### 1) The CUBIC Test

```bash
echo "=== TCP CUBIC TEST ===" >> /home/antlab/my_data.txt
iperf3 -c 192.168.70.135 -B 10.0.0.24 -C cubic -t 20 >> /home/antlab/my_data.txt
```

### 2) The Reno Test

```bash
echo "=== TCP RENO TEST ===" >> /home/antlab/my_data.txt
iperf3 -c 192.168.70.135 -B 10.0.0.24 -C reno -t 20 >> /home/antlab/my_data.txt
```

### 3) The BBR Test

```bash
echo "=== TCP BBR TEST ===" >> /home/antlab/my_data.txt
iperf3 -c 192.168.70.135 -B 10.0.0.24 -C bbr -t 20 >> /home/antlab/my_data.txt
```

Just run this command from your terminal:

```bash
python3 /home/antlab/plot_tcp.py
```

It will parse both your .txt files and output a single, high-resolution image file named `tcp_satellite_results.png` right there.
