# Jarkom_Modul5_Lapres_B03

### A. Membuat topologi jaringan 
```
#Switch
uml_switch -unix switch1 > /dev/null < /dev/null &
uml_switch -unix switch2 > /dev/null < /dev/null &
uml_switch -unix switch3 > /dev/null < /dev/null & #Sidoarjo-Batu
uml_switch -unix switch4 > /dev/null < /dev/null & #Batu-Surabaya
uml_switch -unix switch5 > /dev/null < /dev/null & #Surabaya-Kediri
uml_switch -unix switch6 > /dev/null < /dev/null & #Kediri-Gresik

#Router
xterm -T SURABAYA -e linux ubd0=SURABAYA,jarkom umid=SURABAYA eth0=tuntap,,,10.151.74.17 eth1=daemon,,,switch4 eth2=daemon,,,switch5 mem=96M &
xterm -T BATU -e linux ubd0=BATU,jarkom umid=BATU eth0=daemon,,,switch4 eth1=daemon,,,switch2 eth2=daemon,,,switch3 mem=96M &
xterm -T KEDIRI -e linux ubd0=KEDIRI,jarkom umid=KEDIRI eth0=daemon,,,switch5 eth1=daemon,,,switch1 eth2=daemon,,,switch6 mem=96M &

#Server
xterm -T PROBOLINGGO -e linux ubd0=PROBOLINGGO,jarkom umid=PROBOLINGGO eth0=daemon,,,switch1 mem=128M &
xterm -T MADIUN -e linux ubd0=MADIUN,jarkom umid=MADIUN eth0=daemon,,,switch1 mem=128M &
xterm -T MALANG -e linux ubd0=MALANG,jarkom umid=MALANG eth0=daemon,,,switch2 mem=128M &
xterm -T MOJOKERTO -e linux ubd0=MOJOKERTO,jarkom umid=MOJOKERTO eth0=daemon,,,switch2 mem=128M &

#Client
xterm -T GRESIK -e linux ubd0=GRESIK,jarkom umid=GRESIK eth0=daemon,,,switch6 mem=96M &
xterm -T SIDOARJO -e linux ubd0=SIDOARJO,jarkom umid=SIDOARJO eth0=daemon,,,switch3 mem=96M &
```

### B. CIDR
![CIDR](modul5/CIDR.PNG)
![CIDR_TREE](modul5/CIDR_TREE.PNG)

### C. Melakukan Routing
```
#Surabaya Routing

ip route add 192.168.0.0/22 via 192.168.2.2
ip route add 192.168.4.0/23 via 192.168.5.2
ip route add 10.151.83.32/29 via 192.168.5.2
```

### 1. Mengkonfigurasi SURABAYA menggunakan iptables tanpa menggunakan MASQUERADE.
```
iptables -t nat -A POSTROUTING -s 192.168.0.0/16 -o eth0 -j SNAT --to-source 10.151.74.18
```


### 2. Mendrop semua akses SSH dari luar Topologi pada server yang memiliki ip DMZ pada SURABAYA
```
iptables -A FORWARD -d 10.151.83.32/29 -i eth0 -p tcp --dport 22 -j DROP
```

### 3. Membatasi DHCP dan DNS server hanya boleh menerima maksimal 3 koneksi ICMP secara bersamaan.
```
iptables -A INPUT -p icmp -m connlimit --connlimit-above 3 --connlimit-mask 0 -j DROP
```

### 4. Akses ke MALANG yang berasal dari SUBNET SIDOARJO hanya diperbolehkan pada pukul 07.00 - 17.00 pada hari Senin sampai Jumat.
```
iptables -A INPUT -s 192.168.4.0/24 -m time --timestart 07:00 --timestop 17:00 --weekdays Mon,Tue,Wed,Thu,Fri -j ACCEPT
iptables -A INPUT -s 192.168.4.0/24 -j REJECT
```


### 5. Akses ke MALANG yang berasal dari SUBNET GRESIK hanya diperbolehkan pada pukul 17.00 hingga pukul 07.00 setiap harinya.
```
iptables -A INPUT -s 192.168.4.0/24 -m time --timestart 07:00 --timestop 17:00 --weekdays Mon,Tue,Wed,Thu,Fri -j ACCEPT
iptables -A INPUT -s 192.168.4.0/24 -j REJECT
```

### 6. SURABAYA disetting sehingga setiap request dari client yang mengakses DNS Server akan didistribusikan secara bergantian pada PROBOLINGGO port 80 dan MADIUN port 80.
```
iptables -A PREROUTING -t nat -p tcp -d 10.151.83.34\
         -m statistic --mode nth --every 2 --packet 0\
         -j DNAT --to-destination 192.168.1.3:80
iptables -A PREROUTING -t nat -p tcp -d 10.151.83.34\         
         -j DNAT --to-destination 192.168.1.2:80
```

### 7. Semua paket didrop oleh firewall (dalam topologi) tercatat dalam log pada setiap UML yang memiliki aturan drop.
```
iptables -N LOGGING
iptables -A INPUT -j LOGGING
iptables -A LOGGING -m limit --limit 2/min -j LOG --log-prefix "#####IPTABLES#####" --log-level 4
iptables -A LOGGING -j DROP
```
