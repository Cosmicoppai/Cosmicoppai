---
title: "Docker Security Profile"
seoTitle: "Security Profiles in Docker"
datePublished: Tue Jul 02 2024 22:06:37 GMT+0000 (Coordinated Universal Time)
cuid: cly4yl0il000109me1tfl9uft
slug: docker-security-profile
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1719957888207/27efd543-13ba-469f-a055-9d95fd0f9a25.png
tags: docker, mysql, devops, kernel

---

Recently I was diagnosing a frequent crash of my [<mark>xtrabackup</mark>](https://docs.percona.com/percona-xtrabackup/2.4/index.html) container which is in a docker network along with the MySQL container.

After digging through logs for a very good minute, I came across this error:  

```plaintext
mbind: Operation not permitted
mbind: Operation not permitted
2024-06-28T15:07:48.187675Z 10 [ERROR] [MY-012592] [InnoDB] Operating system error number 5 in a file operation.
2024-06-28T15:07:48.187724Z 10 [ERROR] [MY-012596] [InnoDB] Error number 5 means 'Input/output error'
2024-06-28T15:07:48.187735Z 10 [ERROR] [MY-012646] [InnoDB] File ./#innodb_temp/temp_10.ibt: 'truncate' returned OS error 105.
```

In the Docker ecosystem, seccomp is used as a security feature to restrict the system calls that the container can make to the host kernel. The default seccomp profiles prohibit 44 system calls out of 300+. The full list can be found [<mark>here</mark>](https://docs.docker.com/engine/security/seccomp/).

The error we are encountering is because the `mbind` operation is not allowed by Docker's default seccomp configuration. The `mbind` operation is mostly used for memory access and management across nodes in [<mark>NUMA</mark>](https://en.wikipedia.org/wiki/Non-uniform_memory_access) architecture, which is important in the case of DBMS as it allows them to improve performance by storing data closer to the processor that needs it most.

So how we can override the default prevention, there are multiple ways of doing it none of them are wrong as everything is relative to the use case.

1. Custom Seccomp Profile:
    
    We can create a custom Seccomp profile which is a JSON file, in which all allowed and prohibited actions are specified.
    
    `The profile works by defining a defaultAction of SCMP_ACT_ERRNO and overriding that action only for specific system calls. The effect of SCMP_ACT_ERRNO is to cause a Permission Denied error. Next, the profile defines a specific list of system calls which are fully allowed, because their action is overridden to be SCMP_ACT_ALLOW. Finally, some specific rules are for individual system calls such as personality, and others, to allow variants of those system calls with specific arguments.`
    
    ```json
    {
      "defaultAction": "SCMP_ACT_ERRNO",
      "archMap": [
        {
          "architecture": "SCMP_ARCH_X86_64",
          "subArchitectures": [
            "SCMP_ARCH_X86",
            "SCMP_ARCH_X32"
          ]
        }
      ],
      "syscalls": [
        {
          "names": [
            "mbind"
          ],
          "action": "SCMP_ACT_ALLOW"
        }
      ]
    }
    ```
    
    We can use this profile while running our container or can specify in the compose file:
    
    ```bash
    docker run --security-opt seccomp=custom-seccomp.json ...
    
    OR
    
    services:
      mysql:
        ...
        security_opt:
          - seccomp:custom-seccomp.json
    ```
    
2. Disable Seccomp:  
    We can disable seccomp entirely for the container, but this is not recommended for production use as it reduces security:
    
    ```bash
    docker run --security-opt seccomp=unconfined ...
    
    OR
    
    services:
      mysql:
        ...
        security_opt:
          - seccomp:unconfined
    ```
    
3. Use `--cap-add SYS_NICE`:
    
    This capability allows the container to use `mbind`
    
    ```bash
    docker run --cap-add SYS_NICE ...
    
    OR
    
    services:
      mysql:
        ...
        cap_add:
          - SYS_NICE
    ```
    
4. Run in Privileged Mode:
    
    This gives the container full access to the host's devices, but should be used with extreme caution:
    
    ```bash
    docker run --privileged ...
    
    OR
    
    services:
      mysql:
        ...
        privileged: true
    ```
    
    **Conclusion**: I chose the `--cap-add SYS_NICE` option. If the least privilege option fixes the issue, it's better to use it before trying more permissive options.  
      
      
    Other Consideration:
    
    This feature is available only if Docker has been built with `seccomp` and the kernel is configured with `CONFIG_SECCOMP` enabled. To check if your kernel supports `seccomp`:
    
    ```bash
    $ grep CONFIG_SECCOMP= /boot/config-$(uname -r)
    CONFIG_SECCOMP=y
    ```