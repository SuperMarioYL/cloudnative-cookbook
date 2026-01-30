---
title: "镜像管理代码走读"
weight: 03
---


本章深入分析镜像管理的源码实现，帮助你理解镜像拉取、存储和解压的内部工作机制。

## 代码结构

```
containerd/
├── core/images/
│   ├── image.go            # Image 接口和结构
│   ├── handlers.go         # Handler 模式实现
│   └── mediatypes.go       # MediaType 常量
├── core/transfer/
│   ├── transfer.go         # Transfer 接口
│   └── local/              # 本地传输实现
├── core/unpack/
│   └── unpacker.go         # Unpack 实现
├── remotes/
│   ├── resolver.go         # Resolver 接口
│   ├── docker/             # Docker Registry 实现
│   │   └── resolver.go
│   └── handlers.go         # 远程 Handlers
└── client/
    ├── pull.go             # Pull 客户端实现
    └── image.go            # Image 客户端
```

## Client Pull 入口

### client/pull.go

```go
// client/pull.go

func (c *Client) Pull(ctx context.Context, ref string, opts ...RemoteOpt) (Image, error) {
    // 1. 解析选项
    pullCtx := defaultRemoteContext()
    for _, o := range opts {
        if err := o(c, pullCtx); err != nil {
            return nil, err
        }
    }

    // 2. 解析镜像引用
    ctx, done, err := c.WithLease(ctx)
    if err != nil {
        return nil, err
    }
    defer done(ctx)

    // 3. 创建 Resolver
    resolver := pullCtx.Resolver
    if resolver == nil {
        resolver, err = c.defaultResolver(ctx)
        if err != nil {
            return nil, err
        }
    }

    // 4. 解析镜像
    name, desc, err := resolver.Resolve(ctx, ref)
    if err != nil {
        return nil, err
    }

    // 5. 创建 Fetcher
    fetcher, err := resolver.Fetcher(ctx, name)
    if err != nil {
        return nil, err
    }

    // 6. 执行拉取
    store := c.ContentStore()
    if err := remotes.Fetch(ctx, store, fetcher, desc, pullCtx.PlatformMatcher); err != nil {
        return nil, err
    }

    // 7. 可选: Unpack
    if pullCtx.Unpack {
        if err := c.unpack(ctx, desc, pullCtx); err != nil {
            return nil, err
        }
    }

    // 8. 创建 Image 记录
    img := images.Image{
        Name:   name,
        Target: desc,
        Labels: pullCtx.Labels,
    }

    is := c.ImageService()
    for {
        if created, err := is.Create(ctx, img); err != nil {
            if !errdefs.IsAlreadyExists(err) {
                return nil, err
            }
            updated, err := is.Update(ctx, img)
            if err != nil {
                return nil, err
            }
            img = updated
        } else {
            img = created
        }
        break
    }

    return NewImage(c, img), nil
}
```

**调试断点建议**：
- 第 4 步：观察解析结果
- 第 6 步：观察下载过程
- 第 7 步：观察解压过程

## Fetch 实现

### remotes/handlers.go

```go
// remotes/handlers.go

func Fetch(ctx context.Context, store content.Store, fetcher Fetcher, desc ocispec.Descriptor, platform platforms.MatchComparer) error {
    // 创建 Handler 链
    handler := images.Handlers(
        // 首先尝试从本地读取
        remotes.FetchHandler(store, fetcher),
        // 处理子描述符
        images.ChildrenHandler(store),
    )

    // 如果指定平台，添加过滤
    if platform != nil {
        handler = images.Handlers(
            images.FilterPlatforms(images.ChildrenHandler(store), platform),
            remotes.FetchHandler(store, fetcher),
        )
    }

    // 遍历镜像内容树
    return images.Dispatch(ctx, handler, nil, desc)
}

// FetchHandler 返回一个获取内容的 Handler
func FetchHandler(store content.Ingester, fetcher Fetcher) images.HandlerFunc {
    return func(ctx context.Context, desc ocispec.Descriptor) ([]ocispec.Descriptor, error) {
        // 检查是否已存在
        ctx, done, err := store.Writer(ctx,
            content.WithRef(desc.Digest.String()),
            content.WithDescriptor(desc),
        )
        if err != nil {
            if errdefs.IsAlreadyExists(err) {
                return nil, nil // 已存在，跳过
            }
            return nil, err
        }

        // 获取远程内容
        rc, err := fetcher.Fetch(ctx, desc)
        if err != nil {
            done.Close()
            return nil, err
        }

        // 写入本地
        if _, err := io.Copy(done, rc); err != nil {
            done.Close()
            rc.Close()
            return nil, err
        }
        rc.Close()

        // 提交
        if err := done.Commit(ctx, desc.Size, desc.Digest); err != nil {
            return nil, err
        }

        return nil, nil
    }
}
```

## Docker Resolver 实现

### remotes/docker/resolver.go

```go
// remotes/docker/resolver.go

type dockerResolver struct {
    hosts      RegistryHosts
    auth       Authorizer
    httpClient *http.Client
    tracker    docker.StatusTracker
}

func (r *dockerResolver) Resolve(ctx context.Context, ref string) (string, ocispec.Descriptor, error) {
    // 解析引用
    refspec, err := reference.Parse(ref)
    if err != nil {
        return "", ocispec.Descriptor{}, err
    }

    // 获取 Registry Host 配置
    hosts, err := r.hosts(refspec.Hostname())
    if err != nil {
        return "", ocispec.Descriptor{}, err
    }

    // 遍历可用的 host（支持镜像站）
    var lastErr error
    for _, host := range hosts {
        desc, err := r.resolveFromHost(ctx, host, refspec)
        if err == nil {
            return refspec.String(), desc, nil
        }
        lastErr = err
    }

    return "", ocispec.Descriptor{}, lastErr
}

func (r *dockerResolver) resolveFromHost(ctx context.Context, host RegistryHost, refspec reference.Spec) (ocispec.Descriptor, error) {
    // 构建 URL
    base := host.Scheme + "://" + host.Host + host.Path
    manifestURL := base + "/v2/" + refspec.Locator + "/manifests/" + refspec.Object

    // 发送 HEAD 请求
    req, _ := http.NewRequestWithContext(ctx, "HEAD", manifestURL, nil)
    req.Header.Set("Accept", strings.Join(acceptedMediaTypes, ","))

    // 添加认证
    if err := r.auth.AddAuth(ctx, host, req); err != nil {
        return ocispec.Descriptor{}, err
    }

    resp, err := r.httpClient.Do(req)
    if err != nil {
        return ocispec.Descriptor{}, err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return ocispec.Descriptor{}, fmt.Errorf("unexpected status: %d", resp.StatusCode)
    }

    // 解析响应头
    return ocispec.Descriptor{
        MediaType: resp.Header.Get("Content-Type"),
        Digest:    digest.Digest(resp.Header.Get("Docker-Content-Digest")),
        Size:      resp.ContentLength,
    }, nil
}
```

### Fetcher 实现

```go
func (r *dockerResolver) Fetcher(ctx context.Context, ref string) (remotes.Fetcher, error) {
    refspec, _ := reference.Parse(ref)
    hosts, _ := r.hosts(refspec.Hostname())

    return &dockerFetcher{
        refspec: refspec,
        hosts:   hosts,
        client:  r.httpClient,
        auth:    r.auth,
    }, nil
}

type dockerFetcher struct {
    refspec reference.Spec
    hosts   []RegistryHost
    client  *http.Client
    auth    Authorizer
}

func (f *dockerFetcher) Fetch(ctx context.Context, desc ocispec.Descriptor) (io.ReadCloser, error) {
    for _, host := range f.hosts {
        rc, err := f.fetchFromHost(ctx, host, desc)
        if err == nil {
            return rc, nil
        }
    }
    return nil, fmt.Errorf("fetch failed")
}

func (f *dockerFetcher) fetchFromHost(ctx context.Context, host RegistryHost, desc ocispec.Descriptor) (io.ReadCloser, error) {
    // 判断内容类型
    var path string
    switch desc.MediaType {
    case ocispec.MediaTypeImageManifest,
        ocispec.MediaTypeImageIndex,
        images.MediaTypeDockerSchema2Manifest,
        images.MediaTypeDockerSchema2ManifestList:
        // Manifest
        path = "/v2/" + f.refspec.Locator + "/manifests/" + desc.Digest.String()
    default:
        // Blob
        path = "/v2/" + f.refspec.Locator + "/blobs/" + desc.Digest.String()
    }

    url := host.Scheme + "://" + host.Host + host.Path + path

    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    f.auth.AddAuth(ctx, host, req)

    resp, err := f.client.Do(req)
    if err != nil {
        return nil, err
    }

    if resp.StatusCode != http.StatusOK {
        resp.Body.Close()
        return nil, fmt.Errorf("unexpected status: %d", resp.StatusCode)
    }

    return resp.Body, nil
}
```

## Unpack 实现

### core/unpack/unpacker.go

```go
// core/unpack/unpacker.go

type Unpacker struct {
    content       content.Provider
    snapshotterFn func(string) (snapshots.Snapshotter, error)
    platforms     platforms.MatchComparer
    duplicationSuppressor DuplicationSuppressor
}

func (u *Unpacker) Unpack(ctx context.Context, config UnpackConfig) error {
    // 获取 Manifest
    manifest, err := images.Manifest(ctx, u.content, config.Desc, u.platforms)
    if err != nil {
        return err
    }

    // 获取 Image Config
    configData, err := content.ReadBlob(ctx, u.content, manifest.Config)
    if err != nil {
        return err
    }

    var imageConfig ocispec.Image
    if err := json.Unmarshal(configData, &imageConfig); err != nil {
        return err
    }

    // 获取 Snapshotter
    sn, err := u.snapshotterFn(config.Snapshotter)
    if err != nil {
        return err
    }

    // 解压每个层
    var parent string
    for i, layer := range manifest.Layers {
        diffID := imageConfig.RootFS.DiffIDs[i]

        // 计算 ChainID
        chainID := identity.ChainID(imageConfig.RootFS.DiffIDs[:i+1])

        // 检查是否已存在
        if _, err := sn.Stat(ctx, chainID.String()); err == nil {
            parent = chainID.String()
            continue
        }

        // 使用去重机制
        unlock, err := u.duplicationSuppressor.Lock(ctx, chainID.String())
        if err != nil {
            return err
        }

        // 再次检查（可能已被其他请求解压）
        if _, err := sn.Stat(ctx, chainID.String()); err == nil {
            unlock()
            parent = chainID.String()
            continue
        }

        // 解压层
        if err := u.unpackLayer(ctx, sn, layer, diffID, chainID.String(), parent); err != nil {
            unlock()
            return err
        }

        unlock()
        parent = chainID.String()
    }

    return nil
}

func (u *Unpacker) unpackLayer(ctx context.Context, sn snapshots.Snapshotter, layer ocispec.Descriptor, diffID digest.Digest, key, parent string) error {
    extractKey := key + "-extract"

    // 创建快照
    mounts, err := sn.Prepare(ctx, extractKey, parent,
        snapshots.WithLabels(map[string]string{
            "containerd.io/uncompressed": diffID.String(),
        }),
    )
    if err != nil {
        return err
    }

    // 挂载并解压
    if err := mount.WithTempMount(ctx, mounts, func(root string) error {
        // 获取层内容
        ra, err := u.content.ReaderAt(ctx, layer)
        if err != nil {
            return err
        }
        defer ra.Close()

        // 解压缩
        sr := io.NewSectionReader(ra, 0, ra.Size())
        r, err := compression.DecompressStream(sr)
        if err != nil {
            return err
        }
        defer r.Close()

        // 应用到目录
        if _, err := archive.Apply(ctx, root, r, archive.WithFilter(func(hdr *tar.Header) (bool, error) {
            // 处理 Whiteout 文件
            return !strings.HasPrefix(path.Base(hdr.Name), archive.WhiteoutPrefix), nil
        })); err != nil {
            return err
        }

        return nil
    }); err != nil {
        sn.Remove(ctx, extractKey)
        return err
    }

    // 提交快照
    return sn.Commit(ctx, key, extractKey)
}
```

## Image Store 实现

### core/metadata/images.go

```go
// core/metadata/images.go

type imageStore struct {
    db *DB
}

func (s *imageStore) Get(ctx context.Context, name string) (images.Image, error) {
    namespace, err := namespaces.NamespaceRequired(ctx)
    if err != nil {
        return images.Image{}, err
    }

    var image images.Image

    if err := view(ctx, s.db, func(tx *bolt.Tx) error {
        bkt := getImagesBucket(tx, namespace)
        if bkt == nil {
            return fmt.Errorf("image %q: %w", name, errdefs.ErrNotFound)
        }

        ibkt := bkt.Bucket([]byte(name))
        if ibkt == nil {
            return fmt.Errorf("image %q: %w", name, errdefs.ErrNotFound)
        }

        image.Name = name
        return readImage(&image, ibkt)
    }); err != nil {
        return images.Image{}, err
    }

    return image, nil
}

func (s *imageStore) Create(ctx context.Context, image images.Image) (images.Image, error) {
    namespace, _ := namespaces.NamespaceRequired(ctx)

    if err := update(ctx, s.db, func(tx *bolt.Tx) error {
        bkt, err := createImagesBucket(tx, namespace)
        if err != nil {
            return err
        }

        // 检查是否已存在
        if ibkt := bkt.Bucket([]byte(image.Name)); ibkt != nil {
            return fmt.Errorf("image %q: %w", image.Name, errdefs.ErrAlreadyExists)
        }

        // 创建 bucket
        ibkt, err := bkt.CreateBucket([]byte(image.Name))
        if err != nil {
            return err
        }

        // 写入数据
        image.CreatedAt = time.Now().UTC()
        image.UpdatedAt = image.CreatedAt
        return writeImage(ibkt, &image)
    }); err != nil {
        return images.Image{}, err
    }

    return image, nil
}
```

## 调试技巧

### 使用 ctr 调试

```bash
# 拉取镜像（带详细输出）
ctr images pull --platform linux/amd64 docker.io/library/nginx:latest

# 查看镜像内容
ctr images check docker.io/library/nginx:latest

# 导出镜像内容
ctr images export nginx.tar docker.io/library/nginx:latest

# 查看 Content Store
ctr content ls

# 获取特定内容
ctr content get sha256:abc...
```

### 关键断点

| 文件 | 函数 | 用途 |
|------|------|------|
| `client/pull.go` | `Pull()` | 拉取入口 |
| `remotes/handlers.go` | `FetchHandler()` | 下载逻辑 |
| `remotes/docker/resolver.go` | `Resolve()` | 解析引用 |
| `remotes/docker/resolver.go` | `Fetch()` | 获取内容 |
| `core/unpack/unpacker.go` | `Unpack()` | 解压入口 |
| `core/unpack/unpacker.go` | `unpackLayer()` | 层解压 |

### 日志调试

```bash
# 启用 debug 日志
containerd --log-level debug

# 查看拉取相关日志
journalctl -u containerd | grep -E "(pull|fetch|unpack)"
```

## 常见问题

### 问题 1: "failed to resolve"

```go
// 检查点
// 1. DNS 解析
// 2. Registry 可达性
// 3. 认证配置
```

### 问题 2: "digest mismatch"

```go
// 下载内容与预期 digest 不匹配
// 可能原因：网络问题、Registry 数据损坏
```

### 问题 3: "layer already exists"

```go
// 正常行为，表示层已经被下载过
// 去重机制生效
```

## 小结

镜像管理代码的关键路径：

1. **Client.Pull()**：入口，协调整个流程
2. **Resolver.Resolve()**：解析镜像引用
3. **Fetcher.Fetch()**：获取远程内容
4. **Unpacker.Unpack()**：解压到 Snapshotter

调试建议：
1. 从 `client/pull.go` 开始跟踪
2. 注意 Handler 链的处理顺序
3. 理解 Content Store 和 Snapshotter 的交互

下一章我们将学习 [Runtime 运行时模块](../06-runtime/01-runtime-architecture.md)。
