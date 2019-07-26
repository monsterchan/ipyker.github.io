layout: post
title: kubernetes挂载卷子命令subpath
author: Pyker
categories: kubernetes
tags:
  - kubernetes
date: 2019-07-25 08:20:00
---


k8s允许我们将不同类型的volume挂载到容器的特定目录下。例如，我们可以将configmap的数据以volume的形式挂到容器下。

定义一个configmap，其中的数据以 key:value 的格式体现。
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.level: very
  special.type: |-
    property.1=value-1
    property.2=value-2
    property.3=value-3
```

创建一个Pod， 它挂载上面定义的comfigmap，并在启动时查看挂载目录 /etc/config/ 下有哪些文件。

```bash
$ cat dapi-test-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "ls /etc/config/" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        # Provide the name of the ConfigMap containing the files you want
        # to add to the container
        name: special-config
  restartPolicy: Never
```
```bash
由于pod中的command调用 ls /etc/config/ 命令，那么该容器会显示完文件自动退出！
special.level
special.type
```
可以看到pod将configmap定义的data全部挂载进来了。这里需要注意：
* 挂载目录下的文件名称，即为cm定义里的key值。
* 挂载目录下的文件的内容，即为cm定义里的value值。value可以多行定义，这在一些稍微复杂的场景下特别有用，比如 my.cnf。
* 如果挂载目录下原来有文件，挂载后将不可见（AUFS）。

这些规则其实是和`kubectl explain pod.spec.volumes.configMap.items` 的items参数有关，该参数的意思为：
如果items没有指定，则该ConfigMap的data字段中的所有`键-值对`都将作为一个文件映射到卷中，该`文件的名称是configmap的key`，`内容是configmap的value`。如果指定了items，那么指定的key将被映射到指定的路径中，没指定的key将不存在卷中。如果指定了一个在ConfigMap中不存在的key，除非它被标记为`optional: true`，否则卷设置将出错。路径必须是相对于mountPath的，并且不能包含'..路径或以“..”开头。

有的时候，我们希望将文件挂载到某个目录，但希望只是挂载该文件，不要影响挂载目录下的其他文件。有办法吗？
可以用subPath: Path within the volume from which the container's volume should be mounted。Volume中待挂载的子目录。`subPath` 的目的是为了在单一Pod中多次使用同一个volume而设计的。

例如，像下面的LAMP，可以将同一个volume下的 mysql 和 html目录，挂载到不同的挂载点上，这样就不需要为 mysql 和 html 单独创建volume了。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-lamp-site
spec:
    containers:
    - name: mysql
      image: mysql
      env:
      - name: MYSQL_ROOT_PASSWORD
        value: "rootpasswd" 
      volumeMounts:
      - mountPath: /var/lib/mysql
        name: site-data
        subPath: mysql
    - name: php
      image: php:7.0-apache
      volumeMounts:
      - mountPath: /var/www/html
        name: site-data
        subPath: html
    volumes:
    - name: site-data
      persistentVolumeClaim:
        claimName: my-lamp-site-data
```
那怎么解决我们的问题呢？很简单，如果subPath不是目录，而是一个文件，那么就可以达到单独挂载的目的了。
```yaml
containers:
- volumeMounts:
  - name: demo-config
    mountPath: /etc/special.type
    subPath: special.type
volumes:
- name: demo-config
  configMap:
    name: special-config
```
如果volumes使用的是NFS文件存储方式，subPath默认是该卷的子目录，如果想挂载NFS文件存储中的文件，则该文件必须事先存在，且后期不能在NFS文件存储中修改，如果手动修改会报`cat: can't open 'xxx': Stale NFS file handle` 的错误。 该错误可能需要重启Pod，让Pod重新挂载才行。但是可以在pod中直接修改该文件。


来看看实现。重点应该是在t.Mode()&os.ModeDir，即根据volume中的subPath指定的是目录还是文件分别动作。
```java
func doBindSubPath(mounter Interface, subpath Subpath, kubeletPid int) (hostPath string, err error) {
...
	// Create target of the bind mount. A directory for directories, empty file
	// for everything else.
	t, err := os.Lstat(subpath.Path)
	if err != nil {
		return "", fmt.Errorf("lstat %s failed: %s", subpath.Path, err)
	}
	if t.Mode()&os.ModeDir > 0 {
		if err = os.Mkdir(bindPathTarget, 0750); err != nil && !os.IsExist(err) {
			return "", fmt.Errorf("error creating directory %s: %s", bindPathTarget, err)
		}
	} else {
		// "/bin/touch <bindDir>".
		// A file is enough for all possible targets (symlink, device, pipe,
		// socket, ...), bind-mounting them into a file correctly changes type
		// of the target file.
		if err = ioutil.WriteFile(bindPathTarget, []byte{}, 0640); err != nil {
			return "", fmt.Errorf("error creating file %s: %s", bindPathTarget, err)
		}
	}
```