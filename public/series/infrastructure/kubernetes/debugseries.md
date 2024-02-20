# https://iximiuz.com/

## 容器

### 方法一 直接安装

docker exec安装软件debug

### 方法二 换个镜像

换成有工具的镜像

### 方法三 挂载工具

缺点：需要重启容器

```shell
# 1. Prepare the debugger "image" (not quite):
$ docker create --name debugger busybox
$ mkdir debugger
$ docker export debugger | tar -xC debugger


# 2. Start the guinea-pig container (distroless):
$ docker run -d --rm \
  -v $(pwd)/debugger:/.debugger \
  --name my-distroless gcr.io/distroless/nodejs \
  -e 'setTimeout(() => console.log("Done"), 99999999)'


# 3. Start the debugging session:
$ docker exec -it my-distroless /.debugger/bin/sh
```

Appending the .debugger/bin folder to the $PATH env var has a different effect than prepending it! In the case of a name collision, the binaries from the target container will have precedence over the ones residing on the .debugger/bin mount.

If the opposite is desirable, you can inverse the order: `export PATH=/.debugger/bin:${PATH}`

nixery可以指定需要的工具

```shell
# 1. Prepare the debugger - it'll contain a shell (bash) and tcpdump!
$ docker create --name debugger nixery.dev/shell/tcpdump
$ mkdir debugger
$ docker export debugger | tar -xC debugger

# 2. Start the container that needs to be debugged
#    (notice how the volumes are slightly more complex)
$ docker run -d --rm \
  -v $(pwd)/debugger/nix:/nix \
  -v $(pwd)/debugger/bin:/.debugger/bin \
  --name my-distroless gcr.io/distroless/nodejs \
  -e 'setTimeout(() => console.log("Done"), 99999999)'

# 3. Start the debugging session:
$ docker exec -it my-distroless /.debugger/bin/bash
bash-5.1$# export PATH=${PATH}:/.debugger/bin
bash-5.1$# tcpdump
```

### 方法四 共享命名空间

缺点：无法共享mnt空间，ipc空间需要以对应选项启动

```shell
# Preparing the guinea pig container (distroless/nodejs):
$ docker run -d --rm \
  --name my-distroless gcr.io/distroless/nodejs \
  -e 'setTimeout(() => console.log("Done"), 99999999)'

# Starting the debugger container (busybox)
$ docker run --rm -it \
  --name debugger \
  --pid container:my-distroless \
  --network container:my-distroless \
  busybox \
  sh

# With the pid namespace shared, you can still access the target's container filesystem using the following trick:
/ $# ls -l /proc/1/root/  # or any other PID that belongs to the target container

```
```
Using docker run, shared namespaces, and chroot

I find this trick so handy that I even automated it in my new container debugger tool iximiuz/cdebug. With cdebug, exec-ing into a container becomes as simple as just:

cdebug exec -it <target-container-name-or-id>
The above command starts a debugger "sidecar" container using the busybox:latest image. But if you need something more powerful, you can always use the --image flag:

cdebug exec --privileged -it --image nixery.dev/shell/ps/vim/tshark <target>
'''

```

