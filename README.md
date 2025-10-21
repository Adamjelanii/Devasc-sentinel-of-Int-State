# Devasc-sentinel-of-Int-State
    Program Python ini dirancang untuk berjalan di lingkungan DevAsc VM (Oracle VirtualBox) yang berbasis Linux. Program ini memiliki dua fungsi: pertama, menampilkan status semua antarmuka jaringan sistem, dan kedua, secara otomatis menonaktifkan antarmuka yang tidak terpakai untuk meningkatkan keamanan.

**#Perintah untuk menampilkan status interface di sistem**
    devasc@labvm:~$ ip link show

**#membuat library python untuk menyimpan program**
    devasc@labvm:~$ nano disable_linux_ports.py

**#program yang digunakan**

    import subprocess
    import json
    import sys

    def get_interfaces_to_disable():
    print("Mengecek status interface lokal...")
    try:
        # Jalankan 'ip -j link show' untuk mendapatkan output JSON
        # Output JSON jauh lebih aman untuk diproses daripada teks biasa
        result = subprocess.run(
            ['ip', '-j', 'link', 'show'], 
            capture_output=True, 
            text=True, 
            check=True
        )
        
        interfaces = json.loads(result.stdout)
        interfaces_to_shutdown = []
        
        print("--- Status Interface SAAT INI ---")
        
        for iface in interfaces:
            ifname = iface.get('ifname')
            flags = iface.get('flags', [])
            operstate = iface.get('operstate') # Status operasional (mis: UP, DOWN, LOWERLAYERDOWN)

            # Cetak status untuk perbandingan
            print(f"  Interface: {ifname}, Flags: {flags}, Operstate: {operstate}")

            # Abaikan 'lo' (loopback)
            if ifname == 'lo':
                continue
                
            if 'UP' in flags and operstate != 'UP':
                interfaces_to_shutdown.append(ifname)
                
        print("-----------------------------------")
        return interfaces_to_shutdown

    except FileNotFoundError:
        print("Error: Perintah 'ip' tidak ditemukan. Pastikan Anda menjalankan ini di Linux.")
        sys.exit(1)
    except subprocess.CalledProcessError as e:
        print(f"Error saat menjalankan 'ip link show': {e}")
        sys.exit(1)
    except Exception as e:
        print(f"Error: {e}")
        sys.exit(1)

def disable_interfaces(interfaces):
    if not interfaces:
        print("\nTidak ada interface yang perlu dinonaktifkan.")
        return

    print(f"\nInterface berikut akan dinonaktifkan: {', '.join(interfaces)}")

    for iface in interfaces:
        print(f"Menjalankan: sudo ip link set {iface} down")
        try:
            subprocess.run(
                ['sudo', 'ip', 'link', 'set', iface, 'down'], 
                check=True
            )
            print(f"Berhasil menonaktifkan {iface}.")
        except subprocess.CalledProcessError as e:
            print(f"Gagal menonaktifkan {iface}: {e}")
            print("PASTIKAN Anda menjalankan skrip ini menggunakan 'sudo'.")
        except FileNotFoundError:
            print("Error: Perintah 'sudo' atau 'ip' tidak ditemukan.")
            break
            
def main():
    import os
    if os.geteuid() != 0:
        print("Error: Skrip ini perlu dijalankan dengan hak sudo untuk mengubah pengaturan jaringan.")
        print("Silakan jalankan dengan: sudo python3 disable_linux_ports.py")
        sys.exit(1)

    interfaces = get_interfaces_to_disable()
    disable_interfaces(interfaces)
    
    if interfaces:
        print("\nVerifikasi: Menjalankan 'ip link show' lagi...")
        subprocess.run(['ip', 'link', 'show'])

if __name__ == "__main__":
    main()
  
**#Menjalankan Program**
devasc@labvm:~$ sudo python3 disable_linux_ports.py

#Output program
Mengecek status interface lokal...
--- Status Interface SAAT INI (dari 'ip link show') ---
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:e9:3d:e6 brd ff:ff:ff:ff:ff:ff
3: dummy0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 66:6a:76:ba:a0:2b brd ff:ff:ff:ff:ff:ff
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default 
    link/ether 02:42:a9:20:d1:e8 brd ff:ff:ff:ff:ff:ff
-------------------------------------------------------
[ ] Interface diabaikan: enp0s3 (Status: BROADCAST, MULTICAST, UP, LOWER_UP)
[ ] Interface diabaikan: dummy0 (Status: BROADCAST, MULTICAST, UP, LOWER_UP)
[*] Target ditemukan: docker0 (Status: UP, tapi tidak LOWER_UP)

Interface berikut akan dinonaktifkan: docker0
Menjalankan: sudo ip link set docker0 down
Berhasil menonaktifkan docker0.

--- Verifikasi: Status Interface SETELAH dinonaktifkan ---
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:e9:3d:e6 brd ff:ff:ff:ff:ff:ff
3: dummy0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 66:6a:76:ba:a0:2b brd ff:ff:ff:ff:ff:ff
4: docker0: <BROADCAST,MULTICAST> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default 
    link/ether 02:42:a9:20:d1:e8 brd ff:ff:ff:ff:ff:ff
---------------------------------------------------------

    
