# Debugging different test failure
Run "sudo sh ./run-health-checks.sh.2 -c conf/nd96isr_h100_v5.conf" would encounter different failures.
We can troubleshoot by running sample container, and run the respected failure test in container to debug it.
```
sudo docker run --name=aznhc --net=host --rm --gpus all --privileged --shm-size=8g -it mcr.microsoft.com/aznhc/aznhc-nv bash
```
## Scenario 1: nccl_allreduce failure
NCCL allreduce test failed with return code 3
```
ERROR:  nhc:  Health check failed:  collect_nccl_allreduce_data: nccl_allreduce returned error code 3. FaultCode: NHCNA
Health checks completed with exit code: 1.

```
Run below test inside container:
```
/opt/openmpi/bin/mpirun -np 8  --map-by ppr:8:node -bind-to numa -mca coll_hcoll_enable 0 --allow-run-as-root -x LD_LIBRARY_PATH=/azure-nhc/lib:/usr/local/nvidia/lib:/usr/local/nvidia/lib64:/opt/openmpi/lib -x NCCL_IB_PCI_RELAXED_ORDERING=1 -x UCX_IB_PCI_RELAXED_ORDERING=on -x UCX_TLS=tcp -x UCX_NET_DEVICES=eth0 -x CUDA_DEVICE_ORDER=PCI_BUS_ID -x NCCL_SOCKET_IFNAME=eth0 -x NCCL_TOPO_FILE=/azure-nhc/topofiles/ndv5-topo.xml /opt/nccl-tests/build/all_reduce_perf -b 16G -f 2 -g 1 -e 16G -c 1
```
I can get below detailed error mesage. This is due to mlx5_1 is NIC device. The mpirun test uses mlx_1 devices.
```
[1750832176.378614] [elsah100:40   :0]           ib_md.c:1537 UCX  WARN  mlx5_1: relaxed order memory access requested, but unsupported
...
--------------------------------------------------------------------------
Primary job  terminated normally, but 1 process returned
a non-zero exit code. Per user-direction, the job has been aborted.
--------------------------------------------------------------------------
--------------------------------------------------------------------------
mpirun detected that one or more processes exited with non-zero status, thus causing
the job to be terminated. The first process to do so was:

  Process name: [[56171,1],6]
  Exit code:    3
```
If i enable debugging logs by adding "-x NCCL_DEBUG=INFO -x NCCL_DEBUG_SUBSYS=INIT,NET", i can see all 8 mlx5_x devices are used for RDMA:
```
elsah100:186:303 [6] NCCL INFO NET/IB : Using [0]mlx5_0:1/IB [1]mlx5_1:1/RoCE [2]mlx5_2:1/IB [3]mlx5_3:1/IB [4]mlx5_4:1/IB [5]mlx5_5:1/IB [6]mlx5_6:1/IB [7]mlx5_7:1/IB [8]mlx5_8:1/IB [RO]; OOB eth0:10.0.0.4<0>

elsah100:176:302 [1] NCCL INFO NET/IB : GPU Direct RDMA Enabled for HCA 0 'mlx5_0'
elsah100:176:302 [1] NCCL INFO NET/IB : GPU Direct RDMA Enabled for HCA 1 'mlx5_1'

elsah100:176:302 [1] graph/xml.h:85 NCCL WARN Attribute busid of node nic not found

```
To resolve this, we have to remove nic from mlx5_x.




