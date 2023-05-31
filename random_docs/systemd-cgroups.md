# cgroups, kubepods slices and systemd

Some commands and tricks about interacting cgroups and systemd. How it works the filesystem, etc.

Pods runs locally in the host using cgroups. There are different cgroups drivers (like cgroupsfs), modern Kubernetes/Openshift uses Systemd.

Basically everything lives under: ''

## Understanding Systemd and cgroups directories

Everything lives under '/sys/fs/cgroup/'

```bash
# ls -l /sys/fs/cgroup/
total 0
dr-xr-xr-x. 7 root root  0 Jul 26 06:52 blkio
lrwxrwxrwx. 1 root root 11 Jul 26 06:52 cpu -> cpu,cpuacct
dr-xr-xr-x. 7 root root  0 Jul 26 06:52 cpu,cpuacct
lrwxrwxrwx. 1 root root 11 Jul 26 06:52 cpuacct -> cpu,cpuacct
dr-xr-xr-x. 5 root root  0 Jul 26 06:52 cpuset
dr-xr-xr-x. 7 root root  0 Jul 26 06:52 devices
dr-xr-xr-x. 4 root root  0 Jul 26 06:52 freezer
dr-xr-xr-x. 5 root root  0 Jul 26 06:52 hugetlb
dr-xr-xr-x. 7 root root  0 Jul 26 06:52 memory
lrwxrwxrwx. 1 root root 16 Jul 26 06:52 net_cls -> net_cls,net_prio
dr-xr-xr-x. 5 root root  0 Jul 26 06:52 net_cls,net_prio
lrwxrwxrwx. 1 root root 16 Jul 26 06:52 net_prio -> net_cls,net_prio
dr-xr-xr-x. 4 root root  0 Jul 26 06:52 perf_event
dr-xr-xr-x. 7 root root  0 Jul 26 06:52 pids
dr-xr-xr-x. 3 root root  0 Jul 26 06:52 rdma
dr-xr-xr-x. 7 root root  0 Jul 26 06:52 systemd
```

You can check all your pods managed by systemd/cgroup:

```bash
sh-4.4$> systemd-cgls -u kubepods.slice
Unit kubepods.slice (/kubepods.slice):
├─kubepods-burstable.slice
│ ├─kubepods-burstable-podb34ebfaf_8cdf_4ed1_abb1_96a43a4507e1.slice
│ │ ├─crio-conmon-f4a306feb51067fd5bf09c30300a9bd5727b090ff9849e063d19106d4eb374b4.scope
│ │ │ └─9587 /usr/bin/conmon -b /run/containers/storage/overlay-containers/f4a306feb51067fd5bf09c30300a9bd5727b090ff9849e063d19106d4eb374b4/userdata -c f4a306feb51067fd5bf09c30300a9bd5727b090ff9849e063d19106d4eb374b4 --exit-dir /var/run/crio/exits -l /var/log/pods/openshift-machine-config-operator_machine-config-ser>
│ │ └─crio-f4a306feb51067fd5bf09c30300a9bd5727b090ff9849e063d19106d4eb374b4.scope
│ │   └─9617 /usr/bin/machine-config-server start --apiserver-url=https://api-int.el8k.hpecloud.org:6443
│ ├─kubepods-burstable-podf3a2b4bd_1ba1_4876_ac83_545305bea03e.slice
│ │ ├─crio-conmon-f78ceaaf45813e3979066e130108a138e9269f681e4705477e18bdce1ead83d7.scope
│ │ │ └─21823 /usr/bin/conmon -b /run/containers/storage/overlay-containers/f78ceaaf45813e3979066e130108a138e9269f681e4705477e18bdce1ead83d7/userdata -c f78ceaaf45813e3979066e130108a138e9269f681e4705477e18bdce1ead83d7 --exit-dir /var/run/crio/exits -l /var/log/pods/openshift-ingress_router-default-f4bdc774f-tsscb_f3>
│ │ └─crio-f78ceaaf45813e3979066e130108a138e9269f681e4705477e18bdce1ead83d7.scope
│ │   ├─  21836 /usr/bin/openshift-router --v=2
│ │   └─2038497 /usr/sbin/haproxy -f /var/lib/haproxy/conf/haproxy.config -p /var/lib/haproxy/run/haproxy.pid -x /var/lib/haproxy/run/haproxy.sock -sf 24420
│ ├─kubepods-burstable-pod256974ab_704f_4674_b954_4201bd5f0dc7.slice
│ │ ├─crio-9421f9e102eaa2afad00a25e534bd8c369eb4b795452456f005edfd64e651918.scope
│ │ │ └─1623621 /work agent --spoke-cluster-name=local-cluster --hub-kubeconfig=/spoke/hub-kubeconfig/kubeconfig
│ │ └─crio-conmon-9421f9e102eaa2afad00a25e534bd8c369eb4b795452456f005edfd64e651918.scope
│ │   └─1623582 /usr/bin/conmon -b /run/containers/storage/overlay-containers/9421f9e102eaa2afad00a25e534bd8c369eb4b795452456f005edfd64e651918/userdata -c 9421f9e102eaa2afad00a25e534bd8c369eb4b795452456f005edfd64e651918 --exit-dir /var/run/crio/exits -l /var/log/pods/open-cluster-management-agent_klusterlet-work-age>
│ ├─kubepods-burstable-pod2b3e0230_3062_45dd_b45b_5a2a9bd79cde.slice
│ │ ├─crio-conmon-769b6c988d4064edc53517cca97deacba93e94c97fd0bcfc165aa521b5f5716a.scope
│ │ │ └─498768 /usr/bin/conmon -b /run/containers/storage/overlay-containers/769b6c988d4064edc53517cca97deacba93e94c97fd0bcfc165aa521b5f5716a/userdata -c 769b6c988d4064edc53517cca97deacba93e94c97fd0bcfc165aa521b5f5716a --exit-dir /var/run/crio/exits -l /var/log/pods/openshift-service-ca_service-ca-846b9d5557-drrgn_2>
│ │ └─crio-769b6c988d4064edc53517cca97deacba93e94c97fd0bcfc165aa521b5f5716a.scope
│ │   └─498794 service-ca-operator controller -v=2
│ ├─kubepods-burstable-podfa023a18_6b17_4fc3_9f22_c30ce3a9041d.slice
│ │ ├─crio-conmon-f5dbbaf5f981a11699df3f0a66b66564accde9118107a5f10677f6a47d896697.scope
│ │ │ └─1622060 /usr/bin/conmon -b /run/containers/storage/overlay-containers/f5dbbaf5f981a11699df3f0a66b66564accde9118107a5f10677f6a47d896697/userdata -c f5dbbaf5f981a11699df3f0a66b66564accde9118107a5f10677f6a47d896697 --exit-dir /var/run/crio/exits -l /var/log/pods/open-cluster-management-agent_klusterlet-work-age>
│ │ └─crio-f5dbbaf5f981a11699df3f0a66b66564accde9118107a5f10677f6a47d896697.scope
│ │   └─1622082 /work agent --spoke-cluster-name=local-cluster --hub-kubeconfig=/spoke/hub-kubeconfig/kubeconfig
│ ├─kubepods-burstable-pod7bde77bf_8d0f_4df7_91d3_451b1c0c2b1d.slice
│ │ ├─crio-conmon-a912eda748731e91e221264d70d8db89ffd8349e487bcc5a2812aa9694e606bb.scope
│ │ │ └─1575487 /usr/bin/conmon -b /run/containers/storage/overlay-containers/a912eda748731e91e221264d70d8db89ffd8349e487bcc5a2812aa9694e606bb/userdata -c a912eda748731e91e221264d70d8db89ffd8349e487bcc5a2812aa9694e606bb --exit-dir /var/run/crio/exits -l /var/log/pods/open-cluster-management-hub_cluster-manager-work->
│ │ └─crio-a912eda748731e91e221264d70d8db89ffd8349e487bcc5a2812aa9694e606bb.scope
│ │   └─1575568 /work webhook --secure-port=6443 --tls-cert-file=/serving-cert/tls.crt --tls-private-key-file=/serving-cert/tls.key
<REDACTED>
```

systemd manages other services (apart of cgroup) that interacts with the kernel. This is why we are specifying the kubepods.slice. Pods created by Kubernetes.

## Exploring the filesystem of Systemd

All these pods are divided into slices (burstable, besteffort, etc). In the local fs you can find all the slices managed by Systemd:

```bash
sh-4.4$> ls /sys/fs/cgroup/cpu/ | grep slice
kubepods.slice
machine.slice
system.slice
user.slice
```

For us, the important slice is the one about kubepods.slice. There you can find all the, for example, burstable pods:

```bash
sh-4.4# ls /sys/fs/cgroup/cpu/kubepods.slice/kubepods-besteffort.slice/ | grep kubepods
kubepods-besteffort-pod495f1778_9cf1_49af_b124_4cb23e579a4e.slice
kubepods-besteffort-pod7304b8c0_d3bb_4f69_a104_646ecfee1876.slice
kubepods-besteffort-pod7aecee56_5e76_4630_a907_ee0189db7e8b.slice
kubepods-besteffort-pod89b82275_f415_4b33_af20_6be9e18d615c.slice
kubepods-besteffort-pod971140cd_4f3a_4b43_bcd7_dc8c2f26a5ad.slice
kubepods-besteffort-poddd9abcc7_6050_48cc_91e3_fa73811d0144.slice
kubepods-besteffort-poddf02f0c5_dfca_4a4a_90b9_fc59e5637978.slice
kubepods-besteffort-podf3b9edb3_05a6_4693_ba24_130138ee01a9.slice
kubepods-besteffort-podf841fc0b_c784_473a_a1fe_95f8a5c637af.slice
```

The filesystem is divided about the resources assigned to each pod under the cgroup. So, similar can be done for memory:

```bash
sh-4.4$> ls /sys/fs/cgroup/memory/kubepods.slice/kubepods-burstable.slice/ | grep burstable
kubepods-burstable-pod0185634f_eef8_427e_a2c5_884d45bc5503.slice
kubepods-burstable-pod02d8af75_be7d_4cd2_99bf_f4ea0a711570.slice
kubepods-burstable-pod045c4312_9b56_456b_939b_ea74a06ecc25.slice
kubepods-burstable-pod06addf8c_d1e5_4d89_83c7_b4a9be793c23.slice
kubepods-burstable-pod144001ed_1155_4d1e_b946_1db7e8163f95.slice
kubepods-burstable-pod16065c6a_4ee4_47c5_bebd_56e4e20cfc6d.slice
kubepods-burstable-pod169ec7c8_af83_46f0_b13e_b8c7f7b1d40d.slice
kubepods-burstable-pod2188548b_4550_48bd_a729_96271f805da5.slice
kubepods-burstable-pod2401dd0b724625aff2816035e0229d46.slice
kubepods-burstable-pod256974ab_704f_4674_b954_4201bd5f0dc7.slice
kubepods-burstable-pod267d0739_1e00_4452_81ff_3b421b782aad.slice
kubepods-burstable-pod272b61d4_dafd_46f9_a8ae_867dde2aeb98.slice
kubepods-burstable-pod2a4a24f19b938a7999cb57d788f523c5.slice
kubepods-burstable-pod2aa26341_bd90_4c9b_a5d5_404d134ca333.slice
kubepods-burstable-pod2b3e0230_3062_45dd_b45b_5a2a9bd79cde.slice
kubepods-burstable-pod2e77cf65_0632_47f8_ab96_318af6f24c3f.slice
kubepods-burstable-pod315b496b_db6a_4e73_97ab_9aaca649b7d3.slice
kubepods-burstable-pod32682616_9293_47d5_bb7d_1ce5af1dabfb.slice
kubepods-burstable-pod3348cd3a_94e2_4b40_82d7_6fd0fc615253.slice
```

Inside the directory for each pod, there are other directories for each container on that pod:

```bash
sh-4.4$> ls /sys/fs/cgroup/memory/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-podfffd64fe_49ce_4656_b1a1_637bb776126c.slice | grep crio 
crio-4160da6d39d46e8a6548431694af7c8e3550cb438f5f0ebcab540057b7ee6846.scope
crio-8b65d6de577dc9fd0083867c879ed6e53db37d1d0000d46dbf08e87ff49faedf.scope
crio-conmon-8b65d6de577dc9fd0083867c879ed6e53db37d1d0000d46dbf08e87ff49faedf.scope
```

Two containers on that pod. 

In general container level cgroup is managed by CRIO, pod level by Kubelet, though overall Systemd. 

### Listing containers for a pod

You can enter any slice directory (cgroup) of a pod, to check all containers and processes:

```bash
sh-4.4$> systemd-cgls 
Working directory /sys/fs/cgroup/memory/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-podfffd64fe_49ce_4656_b1a1_637bb776126c.slice:
├─crio-conmon-8b65d6de577dc9fd0083867c879ed6e53db37d1d0000d46dbf08e87ff49faedf.scope
│ └─4145903 /usr/bin/conmon -b /run/containers/storage/overlay-containers/8b65d6de577dc9fd0083867c879ed6e53db37d1d0000d46dbf08e87ff49faedf/userdata -c 8b65d6de577dc9fd0083867c879ed6e53db37d1d0000d46dbf08e87ff49faedf --exit-dir /var/run/crio/exits -l /var>
└─crio-8b65d6de577dc9fd0083867c879ed6e53db37d1d0000d46dbf08e87ff49faedf.scope
  └─4145926 /usr/bin/adapter --prometheus-auth-config=/etc/prometheus-config/prometheus-config.yaml --config=/etc/adapter/config.yaml --logtostderr=true --metrics-relist-interval=1m --prometheus-url=https://prometheus-k8s.openshift-monitoring.svc:9091 --
```

In this case, only one container with one process.

## Listing Pods by consumption

```bash
sh-4.4$> systemd-cgtop -1              

Control Group                                                                                                                                                                                                                                                                          Tasks   %CPU   Memory  Input/s Output/s
/                                                                                                                                                                                                                                                                                       7216      -    64.2G        -        -
/init.scope                                                                                                                                                                                                                                                                                1      -   119.1M        -        -
/kubepods.slice                                                                                                                                                                                                                                                                         6306      -    42.5G        -        -
/kubepods.slice/kubepods-besteffort.slice                                                                                                                                                                                                                                                284      -     1.2G        -        -
/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pod25232883_3c96_4a34_a74f_e78a540afde1.slice                                                                                                                                                                                5      -    47.0M        -        -
/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pod495f1778_9cf1_49af_b124_4cb23e579a4e.slice                                                                                                                                                                               56      -   228.4M        -        -
/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pod7304b8c0_d3bb_4f69_a104_646ecfee1876.slice                                                                                                                                                                                4      -     2.9M        -        -
/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pod7aecee56_5e76_4630_a907_ee0189db7e8b.slice                                                                                                                                                                               53      -    80.9M        -        -
/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pod89b82275_f415_4b33_af20_6be9e18d615c.slice                                                                                                                                                                               57      -   645.6M        -        -
/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-poddd9abcc7_6050_48cc_91e3_fa73811d0144.slice                                                                                                                                                                                4      -    11.1M        -        -
/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-poddf02f0c5_dfca_4a4a_90b9_fc59e5637978.slice                                                                                                                                                                                5      -    25.9M        -        -
/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-podf3b9edb3_05a6_4693_ba24_130138ee01a9.slice                                                                                                                                                                                5      -    29.7M        -        -
/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-podf841fc0b_c784_473a_a1fe_95f8a5c637af.slice                                                                                                                                                                               95      -   178.1M        -        -
/kubepods.slice/kubepods-burstable.slice                                                                                                                                                                                                                                                6022      -    41.3G        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod0185634f_eef8_427e_a2c5_884d45bc5503.slice                                                                                                                                                                                  4      -     2.3M        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod02d8af75_be7d_4cd2_99bf_f4ea0a711570.slice                                                                                                                                                                                282      -     4.3G        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod045c4312_9b56_456b_939b_ea74a06ecc25.slice                                                                                                                                                                                215      -   243.3M        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod06addf8c_d1e5_4d89_83c7_b4a9be793c23.slice                                                                                                                                                                                 48      -    97.9M        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod144001ed_1155_4d1e_b946_1db7e8163f95.slice                                                                                                                                                                                 13      -   110.2M        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod16065c6a_4ee4_47c5_bebd_56e4e20cfc6d.slice  
```

You can specify which kids of pods to list, for example only burstables ones

```bash
sh-4.4$> systemd-cgtop -1 /kubepods.slice/kubepods-burstable.slice

Control Group                                                                                                                                                                                                                                                                           Tasks   %CPU   Memory  Input/s Output/s
/kubepods.slice/kubepods-burstable.slice                                                                                                                                                                                                                                                 6048      -    41.5G        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod0185634f_eef8_427e_a2c5_884d45bc5503.slice                                                                                                                                                                                   4      -     2.3M        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod0185634f_eef8_427e_a2c5_884d45bc5503.slice/crio-78283d52a8505805e4138f05dcecc6e90fd1c3f6fbc0261b4654f3d763c5ed97.scope                                                                                                       2      -     1.4M        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod0185634f_eef8_427e_a2c5_884d45bc5503.slice/crio-conmon-78283d52a8505805e4138f05dcecc6e90fd1c3f6fbc0261b4654f3d763c5ed97.scope                                                                                                2      -   488.0K        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod02d8af75_be7d_4cd2_99bf_f4ea0a711570.slice                                                                                                                                                                                 282      -     4.3G        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod02d8af75_be7d_4cd2_99bf_f4ea0a711570.slice/crio-1197154cd8e6a6699eaff544a906ed53d071f83ac9fce476ffd58ac6ffacc0c9.scope                                                                                                      41      -    33.4M        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod02d8af75_be7d_4cd2_99bf_f4ea0a711570.slice/crio-14038cd94849234944840c56649b874472641f58403011321adf3c3f50512579.scope                                                                                                      44      -    48.9M        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod02d8af75_be7d_4cd2_99bf_f4ea0a711570.slice/crio-348f9bbfed4a2fe857a399256bf91d02ddc8aa70e6f85a6e37e457bea1ab6143.scope                                                                                                      88      -     4.0G        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod02d8af75_be7d_4cd2_99bf_f4ea0a711570.slice/crio-3964be904f5d8006707dcaafdbf45406edb968a93920a3f5bc41ef5efcbea5ae.scope                                                                                                      54      -    88.6M        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod02d8af75_be7d_4cd2_99bf_f4ea0a711570.slice/crio-892fe98078a5419d638ede4f73dc89267d6d17e35c29b3e686c0741f39a61e99.scope                                                                                                      45      -    51.8M        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod02d8af75_be7d_4cd2_99bf_f4ea0a711570.slice/crio-conmon-1197154cd8e6a6699eaff544a906ed53d071f83ac9fce476ffd58ac6ffacc0c9.scope                                                                                                2      -   468.0K        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod02d8af75_be7d_4cd2_99bf_f4ea0a711570.slice/crio-conmon-14038cd94849234944840c56649b874472641f58403011321adf3c3f50512579.scope                                                                                                2      -   472.0K        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod02d8af75_be7d_4cd2_99bf_f4ea0a711570.slice/crio-conmon-348f9bbfed4a2fe857a399256bf91d02ddc8aa70e6f85a6e37e457bea1ab6143.scope                                                                                                2      -    30.1M        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod02d8af75_be7d_4cd2_99bf_f4ea0a711570.slice/crio-conmon-3964be904f5d8006707dcaafdbf45406edb968a93920a3f5bc41ef5efcbea5ae.scope                                                                                                2      -   476.0K        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod02d8af75_be7d_4cd2_99bf_f4ea0a711570.slice/crio-conmon-892fe98078a5419d638ede4f73dc89267d6d17e35c29b3e686c0741f39a61e99.scope                                                                                                2      -    15.5M        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod045c4312_9b56_456b_939b_ea74a06ecc25.slice                                                                                                                                                                                 215      -   241.0M        -        -
/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod045c4312_9b56_456b_939b_ea74a06ecc25.slice/crio-0bfbab25535eb0d5989cc80662b5631da35cbf86ec939409573d3ee525d0c33d.scope                                  
```