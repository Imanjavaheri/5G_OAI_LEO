# 5G_OAI_LEO

Test 1: CUBIC
'''
Bash
echo "=== TCP CUBIC TEST ===" >> /home/antlab/my_data.txt
iperf3 -c 192.168.70.135 -B 10.0.0.24 -C cubic -t 20 >> /home/antlab/my_data.txt
'''

Test 2: Reno

Bash
echo "=== TCP RENO TEST ===" >> /home/antlab/my_data.txt
iperf3 -c 192.168.70.135 -B 10.0.0.24 -C reno -t 20 >> /home/antlab/my_data.txt
Test 3: BBR

Bash
echo "=== TCP BBR TEST ===" >> /home/antlab/my_data.txt
iperf3 -c 192.168.70.135 -B 10.0.0.24 -C bbr -t 20 >> /home/antlab/my_data.txt
