ZFS

Создадим виртуальную машину

В домашней директории создадим директорию zfs, в котором будут храниться настройки виртуальной машины и дополнительные диски:

[student@pv-homeworks1-10 sergsha]$ mkdir ./zfs
[student@pv-homeworks1-10 sergsha]$

Перейлём в директорию zfs:

[student@pv-homeworks1-10 sergsha]$ cd ./zfs/
[student@pv-homeworks1-10 zfs]$

Клонируем файлы zfs с github.com:

[student@pv-homeworks1-10 zfs]$ git clone https://github.com/SergSha/zfs.git
Cloning into 'zfs'...
remote: Enumerating objects: 15, done.
remote: Counting objects: 100% (15/15), done.
remote: Compressing objects: 100% (12/12), done.
remote: Total 15 (delta 1), reused 9 (delta 1), pack-reused 0
Unpacking objects: 100% (15/15), done.
[student@pv-homeworks1-10 zfs]$
 
Заходим в склонированную с github.com директорию zfs:

[student@pv-homeworks1-10 zfs]$ cd ./zfs/
[student@pv-homeworks1-10 zfs]$

Содержимое директории zfs:

[student@pv-homeworks1-10 zfs]$ ls -l
total 12
-rw-rw-r--. 1 student student  688 May 22 20:37 README.md
-rw-rw-r--. 1 student student  703 May 22 20:37 setup_zfs.sh
-rw-rw-r--. 1 student student 2994 May 22 20:37 Vagrantfile
[student@pv-homeworks1-10 zfs]$
 
С помощью редактора vi откроем файл Vagrantfile:

[student@pv-homeworks1-10 zfs]$ vi ./Vagrantfile
------------------------------
# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'open3'
require 'fileutils'

def get_vm_name(id)
  out, err = Open3.capture2e('VBoxManage list vms')
  raise out unless err.exitstatus.zero?

  path = path = File.dirname(__FILE__).split('/').last
  name = out.split(/\n/)
            .select { |x| x.start_with? "\"#{path}_#{id}" }
            .map { |x| x.tr('"', '') }
            .map { |x| x.split(' ')[0].strip }
            .first

  name
end

def controller_exists(name, controller_name)
  return false if name.nil?

  out, err = Open3.capture2e("VBoxManage showvminfo #{name}")
  raise out unless err.exitstatus.zero?
  out.split(/\n/)
     .select { |x| x.start_with? 'Storage Controller Name' }
     .map { |x| x.split(':')[1].strip }
     .any? { |x| x == controller_name }
end

def create_disks(vbox, name)
  # TODO: check that VM is created first time to avoid double vagrant up
 # unless controller_exists(name, 'SATA Controller')
 #   vbox.customize ['storagectl', :id,
 #                   '--name', 'SATA Controller',
 #                   '--add', 'sata']
 # end

  dir = "../vdisks"
  FileUtils.mkdir_p dir unless File.directory?(dir)

  disks = (1..6).map { |x| ["disk#{x}_", '1024'] }

  disks.each_with_index do |(name, size), i|
    file_to_disk = "#{dir}/#{name}.vdi"
    port = (i + 1).to_s

    unless File.exist?(file_to_disk)
      vbox.customize ['createmedium',
                      'disk',
                      '--filename',
                      file_to_disk,
                      '--size',
                      size,
                      '--format',
                      'VDI',
                      '--variant',
                      'standard']
    end

    vbox.customize ['storageattach', :id,
                    '--storagectl', 'SATA Controller',
                    '--port', port,
                    '--type', 'hdd',
                    '--medium', file_to_disk,
                    '--device', '0']

    vbox.customize ['setextradata', :id,
                    "VBoxInternal/Devices/ahci/0/Config/Port#{port}/SerialNumber",
                    name.ljust(20, '0')]
  end
end

Vagrant.configure("2") do |config|

  config.vm.box = "bento/centos-8"
  config.vm.box_version = "202112.19.0"

config.vm.define "server" do |server|

  server.vm.host_name = 'server'
  server.vm.network :private_network, ip: "10.0.0.41"

  server.vm.provider "virtualbox" do |vb|
    vb.memory = "1024"
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
  end

  server.vm.provider 'virtualbox' do |vbx|
      name = get_vm_name('server')
      create_disks(vbx, name)
    end

  server.vm.provision "shell",
    name: "Setup zfs",
    path: "setup_zfs.sh"
  end

  config.vm.define "client" do |client|
    client.vm.host_name = 'client'
    client.vm.network :private_network, ip: "10.0.0.40"
    client.vm.provider :virtualbox do |vb|
      vb.memory = "1024"
      vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    end
  end

end
------------------------------

В строчке 
  disks = (1..6).map { |x| ["disk#{x}_", '1024'] }
количество дисков увеличим до восьми для выполнения домашнего задания, т. е.  
  disks = (1..8).map { |x| ["disk#{x}_", '1024'] }
 
Содержимое скрипта setup_zfs.sh:

[student@pv-homeworks1-10 zfs]$ vi ./setup_zfs.sh
------------------------------
#!/bin/bash


# https://www.centos.org/centos-linux-eol/
# so this is workaround to use vault
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*

yum install -y yum-utils

dnf install -y https://zfsonlinux.org/epel/zfs-release.el8_5.noarch.rpm
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-zfsonlinux

yum-config-manager --enable zfs-kmod
yum-config-manager --disable zfs
yum install -y zfs
modprobe zfs


# enable bash completion

cd /usr/share/bash-completion/completions/
curl -O https://raw.githubusercontent.com/openzfs/zfs/zfs-0.8-release/contrib/bash_completion.d/zfs
chmod +x zfs
------------------------------

Создаём и запускаем виртуальную машину server:

[[student@pv-homeworks1-10 zfs]$ vagrant up server
Bringing machine 'server' up with 'virtualbox' provider...
==> server: Importing base box 'bento/centos-8'...
==> server: Matching MAC address for NAT networking...
==> server: Checking if box 'bento/centos-8' version '202112.19.0' is up to date...
==> server: Setting the name of the VM: zfs_server_1653245472066_74660
==> server: Fixed port collision for 22 => 2222. Now on port 2204.
==> server: Clearing any previously set network interfaces...
==> server: Preparing network interfaces based on configuration...
    server: Adapter 1: nat
    server: Adapter 2: hostonly
==> server: Forwarding ports...
    server: 22 (guest) => 2204 (host) (adapter 1)
==> server: Running 'pre-boot' VM customizations...
==> server: Booting VM...
==> server: Waiting for machine to boot. This may take a few minutes...
    server: SSH address: 127.0.0.1:2204
    server: SSH username: vagrant
    server: SSH auth method: private key
    server:
    server: Vagrant insecure key detected. Vagrant will automatically replace
    server: this with a newly generated keypair for better security.
    server:
    server: Inserting generated public key within guest...
    server: Removing insecure key from the guest if it's present...
    server: Key inserted! Disconnecting and reconnecting using new SSH key...
==> server: Machine booted and ready!
==> server: Checking for guest additions in VM...
    server: The guest additions on this VM do not match the installed version of
    server: VirtualBox! In most cases this is fine, but in rare cases it can
    server: prevent things such as shared folders from working properly. If you see
    server: shared folder errors, please make sure the guest additions within the
    server: virtual machine match the version of VirtualBox you have installed on
    server: your host and reload your VM.
    server:
    server: Guest Additions Version: 6.1.30
    server: VirtualBox Version: 6.0
==> server: Setting hostname...
==> server: Configuring and enabling network interfaces...
==> server: Mounting shared folders...
    server: /vagrant => /home/student/sergsha/zfs/zfs
==> server: Running provisioner: Setup zfs (shell)...
    server: Running: script: Setup zfs
    server: CentOS Linux 8 - AppStream                      5.7 MB/s | 8.4 MB     00:01
    server: CentOS Linux 8 - BaseOS                         5.3 MB/s | 4.6 MB     00:00
    server: CentOS Linux 8 - Extras                          18 kB/s |  10 kB     00:00
    server: Package yum-utils-4.0.21-3.el8.noarch is already installed.
    server: Dependencies resolved.
    server: Nothing to do.
    server: Complete!
    server: Last metadata expiration check: 0:00:03 ago on Sun 22 May 2022 06:52:41 PM UTC.
    server: zfs-release.el8_5.noarch.rpm                     24 kB/s | 9.7 kB     00:00
    server: Dependencies resolved.
    server: ================================================================================
    server:  Package             Architecture   Version          Repository            Size
    server: ================================================================================
    server: Installing:
    server:  zfs-release         noarch         1-8.5            @commandline         9.7 k
    server:
    server: Transaction Summary
    server: ================================================================================
    server: Install  1 Package
    server:
    server: Total size: 9.7 k
    server: Installed size: 2.9 k
    server: Downloading Packages:
    server: Running transaction check
    server: Transaction check succeeded.
    server: Running transaction test
    server: Transaction test succeeded.
    server: Running transaction
    server:   Preparing        :                                                        1/1
    server:   Installing       : zfs-release-1-8.5.noarch                               1/1
    server:   Running scriptlet: zfs-release-1-8.5.noarch                               1/1
    server: error: can't create transaction lock on /var/lib/rpm/.rpm.lock (Resource temporarily unavailable)
    server: error: /etc/pki/rpm-gpg/RPM-GPG-KEY-zfsonlinux: key 1 import failed.
    server: warning: %post(zfs-release-1-8.5.noarch) scriptlet failed, exit status 1
    server:
    server: Error in POSTIN scriptlet in rpm package zfs-release
    server:   Verifying        : zfs-release-1-8.5.noarch                               1/1
    server:
    server: Installed:
    server:   zfs-release-1-8.5.noarch
    server:
    server: Complete!
    server: ZFS on Linux for EL8 - kmod                      58 kB/s | 105 kB     00:01
    server: Dependencies resolved.
    server: ================================================================================
    server:  Package            Arch   Version                              Repo       Size
    server: ================================================================================
    server: Installing:
    server:  zfs                x86_64 2.0.7-1.el8                          zfs-kmod  625 k
    server: Installing dependencies:
    server:  kmod-zfs           x86_64 2.0.7-1.el8                          zfs-kmod  1.5 M
    server:  libnvpair3         x86_64 2.0.7-1.el8                          zfs-kmod   37 k
    server:  libuutil3          x86_64 2.0.7-1.el8                          zfs-kmod   31 k
    server:  libzfs4            x86_64 2.0.7-1.el8                          zfs-kmod  227 k
    server:  libzpool4          x86_64 2.0.7-1.el8                          zfs-kmod  1.3 M
    server:  lm_sensors-libs    x86_64 3.4.0-23.20180522git70f7e08.el8      baseos     59 k
    server:  python3-pip        noarch 9.0.3-20.el8                         appstream  20 k
    server:  python3-setuptools noarch 39.2.0-6.el8                         baseos    163 k
    server:  python36           x86_64 3.6.8-38.module_el8.5.0+895+a459eca8 appstream  19 k
    server:  sysstat            x86_64 11.7.3-6.el8                         appstream 425 k
    server: Enabling module streams:
    server:  python36                  3.6
    server:
    server: Transaction Summary
    server: ================================================================================
    server: Install  11 Packages
    server:
    server: Total download size: 4.3 M
    server: Installed size: 16 M
    server: Downloading Packages:
    server: (1/11): python3-pip-9.0.3-20.el8.noarch.rpm     183 kB/s |  20 kB     00:00
    server: (2/11): python36-3.6.8-38.module_el8.5.0+895+a4 162 kB/s |  19 kB     00:00
    server: (3/11): sysstat-11.7.3-6.el8.x86_64.rpm         2.9 MB/s | 425 kB     00:00
    server: (4/11): lm_sensors-libs-3.4.0-23.20180522git70f 1.3 MB/s |  59 kB     00:00
    server: (5/11): python3-setuptools-39.2.0-6.el8.noarch. 3.4 MB/s | 163 kB     00:00
    server: (6/11): libnvpair3-2.0.7-1.el8.x86_64.rpm        66 kB/s |  37 kB     00:00
    server: (7/11): libuutil3-2.0.7-1.el8.x86_64.rpm         53 kB/s |  31 kB     00:00
    server: (8/11): libzfs4-2.0.7-1.el8.x86_64.rpm          388 kB/s | 227 kB     00:00
    server: (9/11): zfs-2.0.7-1.el8.x86_64.rpm              1.5 MB/s | 625 kB     00:00
    server: (10/11): kmod-zfs-2.0.7-1.el8.x86_64.rpm        882 kB/s | 1.5 MB     00:01
    server: (11/11): libzpool4-2.0.7-1.el8.x86_64.rpm       861 kB/s | 1.3 MB     00:01
    server: --------------------------------------------------------------------------------
    server: Total                                           1.9 MB/s | 4.3 MB     00:02
    server: Running transaction check
    server: Transaction check succeeded.
    server: Running transaction test
    server: Transaction test succeeded.
    server: Running transaction
    server:   Preparing        :                                                        1/1
    server:   Installing       : libnvpair3-2.0.7-1.el8.x86_64                         1/11
    server:   Running scriptlet: libnvpair3-2.0.7-1.el8.x86_64                         1/11
    server:   Installing       : libuutil3-2.0.7-1.el8.x86_64                          2/11
    server:   Running scriptlet: libuutil3-2.0.7-1.el8.x86_64                          2/11
    server:   Installing       : libzfs4-2.0.7-1.el8.x86_64                            3/11
    server:   Running scriptlet: libzfs4-2.0.7-1.el8.x86_64                            3/11
    server:   Installing       : libzpool4-2.0.7-1.el8.x86_64                          4/11
    server:   Running scriptlet: libzpool4-2.0.7-1.el8.x86_64                          4/11
    server:   Installing       : kmod-zfs-2.0.7-1.el8.x86_64                           5/11
    server:   Running scriptlet: kmod-zfs-2.0.7-1.el8.x86_64                           5/11
    server:   Installing       : python3-setuptools-39.2.0-6.el8.noarch                6/11
    server:   Installing       : python36-3.6.8-38.module_el8.5.0+895+a459eca8.x86_    7/11
    server:   Running scriptlet: python36-3.6.8-38.module_el8.5.0+895+a459eca8.x86_    7/11
    server:   Installing       : python3-pip-9.0.3-20.el8.noarch                       8/11
    server:   Installing       : lm_sensors-libs-3.4.0-23.20180522git70f7e08.el8.x8    9/11
    server:   Running scriptlet: lm_sensors-libs-3.4.0-23.20180522git70f7e08.el8.x8    9/11
    server:   Installing       : sysstat-11.7.3-6.el8.x86_64                          10/11
    server:   Running scriptlet: sysstat-11.7.3-6.el8.x86_64                          10/11
    server:   Installing       : zfs-2.0.7-1.el8.x86_64                               11/11
    server:   Running scriptlet: zfs-2.0.7-1.el8.x86_64                               11/11
    server:   Verifying        : python3-pip-9.0.3-20.el8.noarch                       1/11
    server:   Verifying        : python36-3.6.8-38.module_el8.5.0+895+a459eca8.x86_    2/11
    server:   Verifying        : sysstat-11.7.3-6.el8.x86_64                           3/11
    server:   Verifying        : lm_sensors-libs-3.4.0-23.20180522git70f7e08.el8.x8    4/11
    server:   Verifying        : python3-setuptools-39.2.0-6.el8.noarch                5/11
    server:   Verifying        : kmod-zfs-2.0.7-1.el8.x86_64                           6/11
    server:   Verifying        : libnvpair3-2.0.7-1.el8.x86_64                         7/11
    server:   Verifying        : libuutil3-2.0.7-1.el8.x86_64                          8/11
    server:   Verifying        : libzfs4-2.0.7-1.el8.x86_64                            9/11
    server:   Verifying        : libzpool4-2.0.7-1.el8.x86_64                         10/11
    server:   Verifying        : zfs-2.0.7-1.el8.x86_64                               11/11
    server:
    server: Installed:
    server:   kmod-zfs-2.0.7-1.el8.x86_64
    server:   libnvpair3-2.0.7-1.el8.x86_64
    server:   libuutil3-2.0.7-1.el8.x86_64
    server:   libzfs4-2.0.7-1.el8.x86_64
    server:   libzpool4-2.0.7-1.el8.x86_64
    server:   lm_sensors-libs-3.4.0-23.20180522git70f7e08.el8.x86_64
    server:   python3-pip-9.0.3-20.el8.noarch
    server:   python3-setuptools-39.2.0-6.el8.noarch
    server:   python36-3.6.8-38.module_el8.5.0+895+a459eca8.x86_64
    server:   sysstat-11.7.3-6.el8.x86_64
    server:   zfs-2.0.7-1.el8.x86_64
    server:
    server: Complete!
    server:   % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
    server:                                  Dload  Upload   Total   Spent    Left  Speed
100 11305  100 11305    0     0  50244      0 --:--:-- --:--:-- --:--:-- 50244
[student@pv-homeworks1-10 zfs]$
 
Проверим статус созданной и запущенной машины server:

[student@pv-homeworks1-10 zfs]$ vagrant status server
Current machine states:

server                    running (virtualbox)

The VM is running. To stop this VM, you can run `vagrant halt` to
shut it down forcefully, or you can run `vagrant suspend` to simply
suspend the virtual machine. In either case, to restart it again,
simply run `vagrant up`.
[student@pv-homeworks1-10 zfs]$
 
Заходим в машину server:

[student@pv-homeworks1-10 zfs]$ vagrant ssh server

This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
[vagrant@server ~]$
 
Заходим под правами root:

[vagrant@server ~]$ sudo -i
[root@server ~]#


1. Определение алгоритма с наилучшим сжатием
 
Смотрим список дисков в запущенной машине:

[root@server ~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   64G  0 disk
├─sda1   8:1    0  2.1G  0 part [SWAP]
└─sda2   8:2    0 61.9G  0 part /
sdb      8:16   0    1G  0 disk
sdc      8:32   0    1G  0 disk
sdd      8:48   0    1G  0 disk
sde      8:64   0    1G  0 disk
sdf      8:80   0    1G  0 disk
sdg      8:96   0    1G  0 disk
sdh      8:112  0    1G  0 disk
sdi      8:128  0    1G  0 disk
[root@server ~]#
 
Создаём 4 пула из двух дисков в режиме RAID 1:

[root@server ~]# zpool create pool1 mirror /dev/sdb /dev/sdc
[root@server ~]# zpool create pool2 mirror /dev/sdd /dev/sde
[root@server ~]# zpool create pool3 mirror /dev/sdf /dev/sdg
[root@server ~]# zpool create pool4 mirror /dev/sdh /dev/sdi
[root@server ~]#
 
Смотрим информацию о пулах:

[root@server ~]# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
pool1   960M   100K   960M        -         -     0%     0%  1.00x    ONLINE  -
pool2   960M   105K   960M        -         -     0%     0%  1.00x    ONLINE  -
pool3   960M   104K   960M        -         -     0%     0%  1.00x    ONLINE  -
pool4   960M   105K   960M        -         -     0%     0%  1.00x    ONLINE  -
[root@server ~]#
 
Добавим разные алгоритмы сжатия в каждую файловую систему
обавим разные алгоритмы сжатия в каждую файловую систему:
● Алгоритм lzjb: 

[root@server ~]# zfs set compression=lzjb pool1
 
● Алгоритм lz4: 

[root@server ~]# zfs set compression=lz4 pool2

● Алгоритм gzip: 

[root@server ~]# zfs set compression=gzip-9 pool3

● Алгоритм zle: 

[root@server ~]# zfs set compression=zle pool4
 
Проверим, что все файловые системы имеют разные методы сжатия:

[root@server ~]# zfs get all | grep compression
pool1  compression           lzjb                   local
pool2  compression           lz4                    local
pool3  compression           gzip-9                 local
pool4  compression           zle                    local
[root@server ~]#
  
Скачаем один и тот же текстовый файл, например, “Война и мир” во все пулы:

[root@server ~]# for i in {1..4}; do wget -O /pool$i/War_and_Peace.txt https://gutenberg.org/files/2600/2600-0.txt; done
--2022-05-22 19:20:48--  https://gutenberg.org/files/2600/2600-0.txt
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3359408 (3.2M) [text/plain]
Saving to: ‘/pool1/War_and_Peace.txt’

/pool1/War_and_Peace. 100%[======================>]   3.20M  2.71MB/s    in 1.2s

2022-05-22 19:20:50 (2.71 MB/s) - ‘/pool1/War_and_Peace.txt’ saved [3359408/3359408]

--2022-05-22 19:20:50--  https://gutenberg.org/files/2600/2600-0.txt
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3359408 (3.2M) [text/plain]
Saving to: ‘/pool2/War_and_Peace.txt’

/pool2/War_and_Peace. 100%[======================>]   3.20M  2.90MB/s    in 1.1s

2022-05-22 19:20:51 (2.90 MB/s) - ‘/pool2/War_and_Peace.txt’ saved [3359408/3359408]

--2022-05-22 19:20:51--  https://gutenberg.org/files/2600/2600-0.txt
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3359408 (3.2M) [text/plain]
Saving to: ‘/pool3/War_and_Peace.txt’

/pool3/War_and_Peace. 100%[======================>]   3.20M  3.00MB/s    in 1.1s

2022-05-22 19:20:53 (3.00 MB/s) - ‘/pool3/War_and_Peace.txt’ saved [3359408/3359408]

--2022-05-22 19:20:53--  https://gutenberg.org/files/2600/2600-0.txt
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3359408 (3.2M) [text/plain]
Saving to: ‘/pool4/War_and_Peace.txt’

/pool4/War_and_Peace. 100%[======================>]   3.20M  2.88MB/s    in 1.1s

2022-05-22 19:20:54 (2.88 MB/s) - ‘/pool4/War_and_Peace.txt’ saved [3359408/3359408]

[root@server ~]#
 
Проверим, что файл был скачан во все пулы:

[root@server ~]# ls -l /pool*
/pool1:
total 2443
-rw-r--r--. 1 root root 3359408 Jun 22  2021 War_and_Peace.txt

/pool2:
total 2041
-rw-r--r--. 1 root root 3359408 Jun 22  2021 War_and_Peace.txt

/pool3:
total 1239
-rw-r--r--. 1 root root 3359408 Jun 22  2021 War_and_Peace.txt

/pool4:
total 3287
-rw-r--r--. 1 root root 3359408 Jun 22  2021 War_and_Peace.txt
[root@server ~]#
 
Уже на этом этапе видно, что самый оптимальный метод сжатия у нас используется в пуле pool3.
Проверим, сколько места занимает один и тот же файл в разных пулах и проверим степень сжатия файлов:

[root@server ~]# zfs list
NAME    USED  AVAIL     REFER  MOUNTPOINT
pool1  2.52M   829M     2.41M  /pool1
pool2  2.12M   830M     2.02M  /pool2
pool3  1.35M   831M     1.23M  /pool3
pool4  3.35M   829M     3.23M  /pool4
[root@server ~]#
 
[root@server ~]# zfs get all | grep compressratio | grep -v refcompressratio
pool1  compressratio         1.35x                  -
pool2  compressratio         1.61x                  -
pool3  compressratio         2.62x                  -
pool4  compressratio         1.01x                  -
[root@server ~]#
 
Как видим, что алгоритм gzip-9 самый эффективный по сжатию.


2. Определение настроек пула

Скачаем архив в домашний каталог:

[root@server ~]# [root@server ~]# wget -O archive.tar.gz https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg
--2022-05-22 19:33:52--  https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg
Resolving drive.google.com (drive.google.com)... 142.250.186.174
Connecting to drive.google.com (drive.google.com)|142.250.186.174|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://drive.google.com/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg [following]
--2022-05-22 19:33:52--  https://drive.google.com/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg
Reusing existing connection to drive.google.com:443.
HTTP request sent, awaiting response... 303 See Other
Location: https://doc-0c-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/s61arprvo2k0adnbujq3te9bkl8u50h0/1653248025000/16189157874053420687/*/1KRBNW33QWqbvbVHa3hLJivOAt60yukkg [following]
Warning: wildcards not supported in HTTP.
--2022-05-22 19:33:57--  https://doc-0c-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/s61arprvo2k0adnbujq3te9bkl8u50h0/1653248025000/16189157874053420687/*/1KRBNW33QWqbvbVHa3hLJivOAt60yukkg
Resolving doc-0c-bo-docs.googleusercontent.com (doc-0c-bo-docs.googleusercontent.com)... 142.250.186.161
Connecting to doc-0c-bo-docs.googleusercontent.com (doc-0c-bo-docs.googleusercontent.com)|142.250.186.161|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7275140 (6.9M) [application/x-gzip]
Saving to: ‘archive.tar.gz’

archive.tar.gz        100%[======================>]   6.94M  37.0MB/s    in 0.2s

2022-05-22 19:33:58 (37.0 MB/s) - ‘archive.tar.gz’ saved [7275140/7275140]

[root@server ~]#
  
Разархивируем его:

[root@server ~]# tar -xzvf ./archive.tar.gz
zpoolexport/
zpoolexport/filea
zpoolexport/fileb
[root@server ~]#
 
[root@server ~]# ls -l
total 7108
-rw-r--r--. 1 root root 7275140 May 22 19:33 archive.tar.gz
drwxr-xr-x. 2 root root      32 May 15  2020 zpoolexport
[root@server ~]#
 
Проверим, возможно ли импортировать данный каталог в пул:

[root@server ~]# zpool import -d ./zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
status: Some supported features are not enabled on the pool.
 action: The pool can be imported using its name or numeric identifier, though
        some features will not be available without an explicit 'zpool upgrade'.
 config:

        otus                         ONLINE
          mirror-0                   ONLINE
            /root/zpoolexport/filea  ONLINE
            /root/zpoolexport/fileb  ONLINE
[root@server ~]#
   
Данный вывод показывает нам имя пула, тип raid и его состав.
Сделаем импорт данного пула к нам в ОС:

[root@server ~]# zpool import -d ./zpoolexport/ otus
[root@server ~]#

Команда zpool status выдаст нам информацию о составе импортированного
пула:

[root@server ~]# zpool status otus
  pool: otus
 state: ONLINE
status: Some supported features are not enabled on the pool. The pool can
        still be used, but some features are unavailable.
action: Enable all features using 'zpool upgrade'. Once this is done,
        the pool may no longer be accessible by software that does not support
        the features. See zpool-features(5) for details.
config:

        NAME                         STATE     READ WRITE CKSUM
        otus                         ONLINE       0     0     0
          mirror-0                   ONLINE       0     0     0
            /root/zpoolexport/filea  ONLINE       0     0     0
            /root/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors
[root@server ~]#

Запрос сразу всех параметров пула:

[root@server ~]# zpool get all otus
NAME  PROPERTY                       VALUE                          SOURCE
otus  size                           480M                           -
otus  capacity                       0%                             -
otus  altroot                        -                              default
otus  health                         ONLINE                         -
otus  guid                           6554193320433390805            -
otus  version                        -                              default
otus  bootfs                         -                              default
otus  delegation                     on                             default
otus  autoreplace                    off                            default
otus  cachefile                      -                              default
otus  failmode                       wait                           default
otus  listsnapshots                  off                            default
otus  autoexpand                     off                            default
otus  dedupratio                     1.00x                          -
otus  free                           478M                           -
otus  allocated                      2.09M                          -
otus  readonly                       off                            -
otus  ashift                         0                              default
otus  comment                        -                              default
otus  expandsize                     -                              -
otus  freeing                        0                              -
otus  fragmentation                  0%                             -
otus  leaked                         0                              -
otus  multihost                      off                            default
otus  checkpoint                     -                              -
otus  load_guid                      8081224005833367176            -
otus  autotrim                       off                            default
otus  feature@async_destroy          enabled                        local
otus  feature@empty_bpobj            active                         local
otus  feature@lz4_compress           active                         local
otus  feature@multi_vdev_crash_dump  enabled                        local
otus  feature@spacemap_histogram     active                         local
otus  feature@enabled_txg            active                         local
otus  feature@hole_birth             active                         local
otus  feature@extensible_dataset     active                         local
otus  feature@embedded_data          active                         local
otus  feature@bookmarks              enabled                        local
otus  feature@filesystem_limits      enabled                        local
otus  feature@large_blocks           enabled                        local
otus  feature@large_dnode            enabled                        local
otus  feature@sha512                 enabled                        local
otus  feature@skein                  enabled                        local
otus  feature@edonr                  enabled                        local
otus  feature@userobj_accounting     active                         local
otus  feature@encryption             enabled                        local
otus  feature@project_quota          active                         local
otus  feature@device_removal         enabled                        local
otus  feature@obsolete_counts        enabled                        local
otus  feature@zpool_checkpoint       enabled                        local
otus  feature@spacemap_v2            active                         local
otus  feature@allocation_classes     enabled                        local
otus  feature@resilver_defer         enabled                        local
otus  feature@bookmark_v2            enabled                        local
otus  feature@redaction_bookmarks    disabled                       local
otus  feature@redacted_datasets      disabled                       local
otus  feature@bookmark_written       disabled                       local
otus  feature@log_spacemap           disabled                       local
otus  feature@livelist               disabled                       local
otus  feature@device_rebuild         disabled                       local
otus  feature@zstd_compress          disabled                       local
[root@server ~]#

Запрос сразу всех параметром файловой системы:

[root@server ~]# zfs get all otus
NAME  PROPERTY              VALUE                  SOURCE
otus  type                  filesystem             -
otus  creation              Fri May 15  4:00 2020  -
otus  used                  2.04M                  -
otus  available             350M                   -
otus  referenced            24K                    -
otus  compressratio         1.00x                  -
otus  mounted               yes                    -
otus  quota                 none                   default
otus  reservation           none                   default
otus  recordsize            128K                   local
otus  mountpoint            /otus                  default
otus  sharenfs              off                    default
otus  checksum              sha256                 local
otus  compression           zle                    local
otus  atime                 on                     default
otus  devices               on                     default
otus  exec                  on                     default
otus  setuid                on                     default
otus  readonly              off                    default
otus  zoned                 off                    default
otus  snapdir               hidden                 default
otus  aclmode               discard                default
otus  aclinherit            restricted             default
otus  createtxg             1                      -
otus  canmount              on                     default
otus  xattr                 on                     default
otus  copies                1                      default
otus  version               5                      -
otus  utf8only              off                    -
otus  normalization         none                   -
otus  casesensitivity       sensitive              -
otus  vscan                 off                    default
otus  nbmand                off                    default
otus  sharesmb              off                    default
otus  refquota              none                   default
otus  refreservation        none                   default
otus  guid                  14592242904030363272   -
otus  primarycache          all                    default
otus  secondarycache        all                    default
otus  usedbysnapshots       0B                     -
otus  usedbydataset         24K                    -
otus  usedbychildren        2.01M                  -
otus  usedbyrefreservation  0B                     -
otus  logbias               latency                default
otus  objsetid              54                     -
otus  dedup                 off                    default
otus  mlslabel              none                   default
otus  sync                  standard               default
otus  dnodesize             legacy                 default
otus  refcompressratio      1.00x                  -
otus  written               24K                    -
otus  logicalused           1020K                  -
otus  logicalreferenced     12K                    -
otus  volmode               default                default
otus  filesystem_limit      none                   default
otus  snapshot_limit        none                   default
otus  filesystem_count      none                   default
otus  snapshot_count        none                   default
otus  snapdev               hidden                 default
otus  acltype               off                    default
otus  context               none                   default
otus  fscontext             none                   default
otus  defcontext            none                   default
otus  rootcontext           none                   default
otus  relatime              off                    default
otus  redundant_metadata    all                    default
otus  overlay               on                     default
otus  encryption            off                    default
otus  keylocation           none                   default
otus  keyformat             none                   default
otus  pbkdf2iters           0                      default
otus  special_small_blocks  0                      default
[root@server ~]#

Размер хранилища:

[root@server ~]# zfs get available otus
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -
[root@server ~]#
 
Тип:

[root@server ~]# zfs get readonly otus
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default
[root@server ~]#
 
Значение recordsize:

[root@server ~]# zfs get recordsize otus
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local
[root@server ~]#
 
Тип сжатия (или параметр отключения):

[root@server ~]# zfs get compression otus
NAME  PROPERTY     VALUE           SOURCE
otus  compression  zle             local
[root@server ~]#
 
Тип контрольной суммы:

[root@server ~]# zfs get checksum otus
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local
[root@server ~]#
 

3. Работа со снапшотом, поиск сообщения от преподавателя

Скачиваем файл из задания:

[root@server ~]# wget -O otus_task2.file --no-check-certificate 'https://drive.google.com/u/0/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download'
--2022-05-22 20:00:27--  https://drive.google.com/u/0/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download
Resolving drive.google.com (drive.google.com)... 142.250.184.206
Connecting to drive.google.com (drive.google.com)|142.250.184.206|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://drive.google.com/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download [following]
--2022-05-22 20:00:28--  https://drive.google.com/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download
Reusing existing connection to drive.google.com:443.
HTTP request sent, awaiting response... 303 See Other
Location: https://doc-00-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/p6pe8gb7uf2dfs1gf1pn0d78up76atpp/1653249600000/16189157874053420687/*/1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG?e=download [following]
Warning: wildcards not supported in HTTP.
--2022-05-22 20:00:28--  https://doc-00-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/p6pe8gb7uf2dfs1gf1pn0d78up76atpp/1653249600000/16189157874053420687/*/1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG?e=download
Resolving doc-00-bo-docs.googleusercontent.com (doc-00-bo-docs.googleusercontent.com)... 142.250.186.161
Connecting to doc-00-bo-docs.googleusercontent.com (doc-00-bo-docs.googleusercontent.com)|142.250.186.161|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 5432736 (5.2M) [application/octet-stream]
Saving to: ‘otus_task2.file’

otus_task2.file       100%[======================>]   5.18M  --.-KB/s    in 0.1s

2022-05-22 20:00:29 (34.9 MB/s) - ‘otus_task2.file’ saved [5432736/5432736]

[root@server ~]#
 
Восстанавливаем файловую систему из снапшота:

[root@server ~]# zfs receive otus/test@today < ./otus_task2.file
[root@server ~]#
 
Находим в каталоге /otus/test файл с именем “secret_message”:

[root@server ~]# find /otus/test/ -name "secret_message"
/otus/test/task1/file_mess/secret_message
[root@server ~]#
 
Смотрим содержимое найденного файла:

[root@server ~]# cat /otus/test/task1/file_mess/secret_message
https://github.com/sindresorhus/awesome
[root@server ~]#

