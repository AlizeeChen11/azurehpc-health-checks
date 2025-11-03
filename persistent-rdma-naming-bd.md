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

# an_index=0
an_index=8
ib_index=0

for old_device in $(ibdev2netdev -v | sort -n | cut -f2 -d' '); do

	link_layer=$(ibv_devinfo -d $old_device | grep -i 'link_layer' | awk -F: '{gsub(/^[ \t]+/, "", $2); print $2}')
	
	if [ "$link_layer" = "InfiniBand" ]; then
		
	#	$rdma_rename $old_device NAME_FIXED mlx5_ib${ib_index}
    	$rdma_rename $old_device NAME_FIXED mlx5_${ib_index}
		ib_index=$(($ib_index + 1))
		
	elif [ "$link_layer" = "Ethernet" ]; then
	
		# $rdma_rename $old_device NAME_FIXED mlx5_an${an_index}
		$rdma_rename $old_device NAME_FIXED mlx5_${an_index}
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
Once the above steps succeed, check ib device and ethernet device naming to see if it's changed: ibdev2netdev

If the device is down, run command to bring all the ib device up: 
```
for i in {0..7}; do sudo ip link set ib${i} up; done
```


