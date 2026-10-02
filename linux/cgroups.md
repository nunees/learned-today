# Linux CGROUPS

> Cgroups (control groups) is a Linux kernel feature that allows you to allocate resources—such as CPU time, system memory, network bandwidth, or combinations of these resources—among user-defined groups of tasks (processes) running on a system. Cgroups are used to manage and limit the resource usage of processes, which is particularly useful in containerization and virtualization environments.

Instalar o cgruops tools

```
sudo apt install cgroup-tools
```

Cria um grupo chamado giropops em /sys/fs/cgrup

```
cgcreate -g cpu,memory,blkio,devices,freezer:giropops
```

Linkar o PID do namespace criado anteriormente com o cgroup giropops

```
cgclassify -g cpu,memory,blkio,devices,freezer:giropops PID
```

Usar o comando ‘gstet’ para configurar os recursos, no caso abaixo limitar o uso de CPU
```
cgset -r cpu.cfs_quota_us=1000 giropops
```

References:
- [https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/cgroups.html](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/cgroups.html)
