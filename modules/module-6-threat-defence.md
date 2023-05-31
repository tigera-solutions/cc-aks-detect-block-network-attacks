### Detect zero-day attacks based on suspicious container activity using syscalls, file access, and process information 
############################################

Threat Defence > Container Threat Detection - enable

k run attacker --image ubuntu -- sleep infinity
k exec attacker -it -- /bin/bash

apt update
apt-get install nmap
nmap -sn $(hostname -i)/24
nmap -T4 -F $(hostname -i)/24

echo hacker:x:777:0:hacker:/tmp:/bin/bash >> /etc/passwd
passwd hacker

--- 

[:arrow_right: Module 7 - Quarantine Infected Workloads and KSPM](/modules/module-7-quarantine-kspm.md)  <br>

[:arrow_left: Module 5 - Configuring IDS protection and Workload-Centric WAF](/modules/module-5-ids-waf.md)  
[:leftwards_arrow_with_hook: Back to Main](/README.md)  