---
title: 079-自定义Rke2 Containerd Config.toml 支持 V2 规范
urlname: rke2-containerd-config-v2
date: '2022-09-27 19:35:21 +0800'
tags:
  - rancher
  - k8s
  - containerd
  - rke2
  - rke
categories:
  - k8s
  - 云原生
---

> 这是坚持技术写作计划（含翻译）的第 79 篇，定个小目标 999，每周最少 2 篇。

之前 Rancher 集群比较老 v2.6.5，当时的 rke2 stable 版本是 [v1.21.12+rke2r1](https://github.com/rancher/rke2/releases/tag/v1.21.12+rke2r1)，而这个版本的 Containerd 是[v1.4.13-k3s1](https://github.com/k3s-io/containerd/releases/tag/v1.4.13-k3s1) 而 Containerd 在 1.5.x 之前只支持单镜像库加速，在 1.5.x 后支持多镜像库加速（文档 [Registry Configuration](https://github.com/containerd/containerd/blob/main/docs/cri/config.md#registry-configuration)）。

因业务需要，给集群升级 Rancher 版本到 [v2.6.8](https://github.com/rancher/rancher/releases/tag/v2.6.8) ,Rke2 升级到 [v1.24.4+rke2r1](https://github.com/rancher/rke2/releases/tag/v1.24.4+rke2r1) 。集群升级不是本文重点，不多描述。本文主要讲解升级后如何 自定义 rke2 的 `/var/lib/rancher/rke2/agent/etc/containerd/config.toml` 实现 v2 规范。比如 多镜像库通过一个 d7y 集群加速(通过 header 区分)。

<!-- more -->

根据 Rke2 文档 [Advanced Options and Configuration > Configuring containerd](https://docs.rke2.io/advanced/#configuring-containerd) 可知，官方支持通过创建 `/var/lib/rancher/rke2/agent/etc/containerd/config.toml.tmpl`来自定义生成 `/var/lib/rancher/rke2/agent/etc/containerd/config.toml`

`/var/lib/rancher/rke2/agent/etc/containerd/config.toml.tmpl` 如下

```bash
version = 2

[plugins."io.containerd.internal.v1.opt"]
	path = "{{ .NodeConfig.Containerd.Opt }}"

[plugins."io.containerd.grpc.v1.cri"]
  stream_server_address = "127.0.0.1"
  stream_server_port = "10010"
  enable_selinux = {{ .NodeConfig.SELinux }}
  enable_unprivileged_ports = {{ .EnableUnprivileged }}
  enable_unprivileged_icmp = {{ .EnableUnprivileged }}
{{- if .DisableCgroup}}
  disable_cgroup = true
{{end}}
{{- if .IsRunningInUserNS }}
  disable_apparmor = true
  restrict_oom_score_adj = true
{{end}}
{{- if .NodeConfig.AgentConfig.PauseImage }}
  sandbox_image = "{{ .NodeConfig.AgentConfig.PauseImage }}"
{{end}}
{{- if .NodeConfig.AgentConfig.Snapshotter }}
[plugins."io.containerd.grpc.v1.cri".containerd]
  snapshotter = "{{ .NodeConfig.AgentConfig.Snapshotter }}"
  disable_snapshot_annotations = {{ if eq .NodeConfig.AgentConfig.Snapshotter "stargz" }}false{{else}}true{{end}}
{{ if eq .NodeConfig.AgentConfig.Snapshotter "stargz" }}
{{ if .NodeConfig.AgentConfig.ImageServiceSocket }}
[plugins.stargz]
cri_keychain_image_service_path = "{{ .NodeConfig.AgentConfig.ImageServiceSocket }}"
[plugins.stargz.cri_keychain]
enable_keychain = true
{{end}}
{{ if .PrivateRegistryConfig }}
{{ if .PrivateRegistryConfig.Mirrors }}
[plugins.stargz.registry.mirrors]{{end}}
{{range $k, $v := .PrivateRegistryConfig.Mirrors }}
[plugins.stargz.registry.mirrors."{{$k}}"]
  endpoint = [{{range $i, $j := $v.Endpoints}}{{if $i}}, {{end}}{{printf "%q" .}}{{end}}]
{{if $v.Rewrites}}
  [plugins.stargz.registry.mirrors."{{$k}}".rewrite]
{{range $pattern, $replace := $v.Rewrites}}
    "{{$pattern}}" = "{{$replace}}"
{{end}}
{{end}}
{{end}}
{{range $k, $v := .PrivateRegistryConfig.Configs }}
{{ if $v.Auth }}
[plugins.stargz.registry.configs."{{$k}}".auth]
  {{ if $v.Auth.Username }}username = {{ printf "%q" $v.Auth.Username }}{{end}}
  {{ if $v.Auth.Password }}password = {{ printf "%q" $v.Auth.Password }}{{end}}
  {{ if $v.Auth.Auth }}auth = {{ printf "%q" $v.Auth.Auth }}{{end}}
  {{ if $v.Auth.IdentityToken }}identitytoken = {{ printf "%q" $v.Auth.IdentityToken }}{{end}}
{{end}}
{{ if $v.TLS }}
[plugins.stargz.registry.configs."{{$k}}".tls]
  {{ if $v.TLS.CAFile }}ca_file = "{{ $v.TLS.CAFile }}"{{end}}
  {{ if $v.TLS.CertFile }}cert_file = "{{ $v.TLS.CertFile }}"{{end}}
  {{ if $v.TLS.KeyFile }}key_file = "{{ $v.TLS.KeyFile }}"{{end}}
  {{ if $v.TLS.InsecureSkipVerify }}insecure_skip_verify = true{{end}}
{{end}}
{{end}}
{{end}}
{{end}}
{{end}}
{{- if not .NodeConfig.NoFlannel }}
[plugins."io.containerd.grpc.v1.cri".cni]
  bin_dir = "{{ .NodeConfig.AgentConfig.CNIBinDir }}"
  conf_dir = "{{ .NodeConfig.AgentConfig.CNIConfDir }}"
{{end}}
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  runtime_type = "io.containerd.runc.v2"
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
	SystemdCgroup = {{ .SystemdCgroup }}
{{ if .PrivateRegistryConfig }}

[plugins."io.containerd.grpc.v1.cri".registry]
  config_path = "/var/lib/rancher/rke2/agent/etc/containerd/certs.d"

{{range $k, $v := .PrivateRegistryConfig.Configs }}
{{ if $v.Auth }}
[plugins."io.containerd.grpc.v1.cri".registry.configs."{{$k}}".auth]
  {{ if $v.Auth.Username }}username = {{ printf "%q" $v.Auth.Username }}{{end}}
  {{ if $v.Auth.Password }}password = {{ printf "%q" $v.Auth.Password }}{{end}}
  {{ if $v.Auth.Auth }}auth = {{ printf "%q" $v.Auth.Auth }}{{end}}
  {{ if $v.Auth.IdentityToken }}identitytoken = {{ printf "%q" $v.Auth.IdentityToken }}{{end}}
{{end}}
{{ if $v.TLS }}
[plugins."io.containerd.grpc.v1.cri".registry.configs."{{$k}}".tls]
  {{ if $v.TLS.CAFile }}ca_file = "{{ $v.TLS.CAFile }}"{{end}}
  {{ if $v.TLS.CertFile }}cert_file = "{{ $v.TLS.CertFile }}"{{end}}
  {{ if $v.TLS.KeyFile }}key_file = "{{ $v.TLS.KeyFile }}"{{end}}
  {{ if $v.TLS.InsecureSkipVerify }}insecure_skip_verify = true{{end}}
{{end}}
{{end}}
{{end}}
{{range $k, $v := .ExtraRuntimes}}
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes."{{$k}}"]
  runtime_type = "{{$v.RuntimeType}}"
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes."{{$k}}".options]
  BinaryName = "{{$v.BinaryName}}"
{{end}}
```

如果要配置 RKE2 的镜像仓库，参考 [Containerd Registry Configuration](https://docs.rke2.io/install/containerd_registry_configuration/)
`/etc/rancher/rke2/registries.yaml `配置如下所示：

```yaml
mirrors:
  docker.io:
    endpoint:
      - "http://127.0.0.1:65001"
  registry.example.com:
    endpoint:
      - "http://127.0.0.1:65001"
```

创建 registry 目录

```bash
# 如果有镜像库配置，则创建 /var/lib/rancher/rke2/agent/etc/containerd/certs.d 目录
sudo mkdir -p /var/lib/rancher/rke2/agent/etc/containerd/certs.d/
# 或者也可以用下面这行 一次性创建 docker.io,ghcr.io,quay.io 等子目录
sudo mkdir -p /var/lib/rancher/rke2/agent/etc/containerd/certs.d/{docker.io,ghcr.io,quay.io}

# 根据 https://github.com/containerd/containerd/blob/main/docs/hosts.md 创建 hosts.toml 文件
```

如果用了 [d7y ](https://d7y.io/zh/docs/getting-started/quick-start/kubernetes/),也可以直接用他的脚本

```bash
wget https://github.com/dragonflyoss/Dragonfly2/blob/main/hack/gen-containerd-hosts.sh
chmod +x gen-containerd-hosts.sh

sudo CONTAINED_CONFIG_DIR=/var/lib/rancher/rke2/agent/etc/containerd/certs.d/ gen-containerd-hosts.sh docker.io

sudo cat /var/lib/rancher/rke2/agent/etc/containerd/certs.d/docker.io/hosts.toml

server = "https://docker.io"
[host."http://127.0.0.1:65001"]
  capabilities = ["pull", "resolve"]
  [host."http://127.0.0.1:65001".header]
    X-Dragonfly-Registry = ["https://docker.io"]
```

最后别忘了 restart rke2-server 或者 rke2-agent。

校验下是否生效

```bash
# 查看 mirrors 和 configs.auth 是否生效
sudo /var/lib/rancher/rke2/bin/crictl --config=/var/lib/rancher/rke2/agent/etc/crictl.yaml info

# 拉取镜像
sudo /var/lib/rancher/rke2/bin/crictl --config=/var/lib/rancher/rke2/agent/etc/crictl.yaml -D pull nginx:alpine

# 查看日志，是否走镜像加速器
sudo tail -f /var/lib/rancher/rke2/agent/containerd/containerd.log

```

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/gongsi/98c1ccdd9decf9791XR539y5GFA~.html) , 一起搞事情。
长期招聘，Java 程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2022/09/27/rke2-containerd-config-v2/)
- [我的掘金](https://juejin.cn/post/7148244954928644110/)
