# Install Persistent RDMA naming
This document walks through the steps to setup RDMA persistent device naming.

## Referred:
```
https://github.com/Azure/azhpc-images/blob/master/common/install_azure_persistent_rdma_naming.sh
https://github.com/Azure/azhpc-images/blob/master/common/install_azure_persistent_rdma_naming_monitor.sh
```

### Build rdma-core
Run below commands to build rdma-core:
```
wget https://github.com/linux-rdma/rdma-core/archive/refs/tags/v56.2.tar.gz
tar -xvf v56.2.tar.gz
cd rdma-core-56.2
sudo dnf install cmake libnl3-devel -y
bash build.sh
sudo cp build/bin/rdma_rename /usr/sbin/rdma_rename_56.2
```

### Setup systemd service
Follow below steps to setup systemd service:

```
sudo -i
cat <<EOF >/usr/sbin/azure_persistent_rdma_naming.sh
#!/bin/bash

rdma_rename=/usr/sbin/rdma_rename_56.2

an_index=0
ib_index=0

for old_device in $(ibdev2netdev -v | sort -n | cut -f2 -d' '); do

	link_layer=$(ibv_devinfo -d $old_device | grep -i 'link_layer' | awk -F: '{gsub(/^[ \t]+/, "", $2); print $2}')
	
	if [ "$link_layer" = "InfiniBand" ]; then
		
		$rdma_rename $old_device NAME_FIXED mlx5_ib${ib_index}
		ib_index=$(($ib_index + 1))
		
	elif [ "$link_layer" = "Ethernet" ]; then
	
		$rdma_rename $old_device NAME_FIXED mlx5_an${an_index}
		an_index=$(($an_index + 1))
		
	else
	
		echo "Unknown device type for $old_device - $device_type."
		
	fi
	
done
EOF

chmod 755 /usr/sbin/azure_persistent_rdma_naming.sh

cat <<EOF >/etc/systemd/system/azure_persistent_rdma_naming.service
[Unit]
Description=Azure persistent RDMA naming
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/azure_persistent_rdma_naming.sh
RemainAfterExit=true
StandardOutput=journal

[Install]
WantedBy=multi-user.target
EOF

systemctl enable azure_persistent_rdma_naming.service
systemctl start azure_persistent_rdma_naming.service
systemctl status azure_persistent_rdma_naming.service
```
Once the above steps succeed, the ib device name will be changed:
```
[root@elsah100 ~]# ibdev2netdev
mlx5_an0 port 1 ==> eth1 (Up)
mlx5_ib0 port 1 ==> ib0 (Down)
mlx5_ib1 port 1 ==> ib1 (Down)
mlx5_ib2 port 1 ==> ib2 (Down)
mlx5_ib3 port 1 ==> ib3 (Down)
mlx5_ib4 port 1 ==> ib4 (Down)
mlx5_ib5 port 1 ==> ib5 (Down)
mlx5_ib6 port 1 ==> ib6 (Down)
mlx5_ib7 port 1 ==> ib7 (Down)
```
Run command to bring all the ib device up: 
```
for i in {0..7}; do sudo ip link set ib${i} up; done
```
Check device status:
```
[root@elsah100 ~]# ibdev2netdev
mlx5_an0 port 1 ==> eth1 (Up)
mlx5_ib0 port 1 ==> ib0 (Up)
mlx5_ib1 port 1 ==> ib1 (Up)
mlx5_ib2 port 1 ==> ib2 (Up)
mlx5_ib3 port 1 ==> ib3 (Up)
mlx5_ib4 port 1 ==> ib4 (Up)
mlx5_ib5 port 1 ==> ib5 (Up)
mlx5_ib6 port 1 ==> ib6 (Up)
mlx5_ib7 port 1 ==> ib7 (Up)
```
### Install RMDA naming monitoring
```
vi /usr/sbin/azure_persistent_rdma_naming_monitor.sh
```
Enter below content:
```
# referred https://github.com/Azure/azhpc-images/blob/master/common/install_azure_persistent_rdma_naming_monitor.sh
# need to make some format changes to the file.
#!/bin/bash

# monitoring service to check that hca_id's are named correctly
# if incorrect, restart azure_persistent_rdma_naming.service

while true; do 

    for device in $(ibdev2netdev -v | sort -n | cut -f2 -d' '); do
        
	link_layer=$(ibv_devinfo -d $device | grep -i 'link_layer' | awk -F: '{gsub(/^[ \t]+/, "", $2); print $2}')

        if [[ $device != *"an"* && $device != *"ib"* ]]; then 
            systemctl enable azure_persistent_rdma_naming.service
            systemctl restart azure_persistent_rdma_naming.service
            sleep 60
            break
        fi
        
    done

    sleep 60 

done
```
Then run:
```
chmod 755 /usr/sbin/azure_persistent_rdma_naming_monitor.sh
cat <<EOF >/etc/systemd/system/azure_persistent_rdma_naming_monitor.service
[Unit]
Description=Azure persistent RDMA naming Monitor
After=network.target

[Service]
Type=simple
ExecStart=/usr/sbin/azure_persistent_rdma_naming_monitor.sh
RemainAfterExit=true
StandardOutput=journal

[Install]
WantedBy=multi-user.target
EOF

systemctl enable azure_persistent_rdma_naming_monitor.service
systemctl start azure_persistent_rdma_naming_monitor.service
systemctl status azure_persistent_rdma_naming_monitor.service
```

