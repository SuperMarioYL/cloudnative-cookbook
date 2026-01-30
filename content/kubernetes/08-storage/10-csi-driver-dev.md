---
title: "CSI 驱动开发指南"
weight: 10
---
## 概述

CSI（Container Storage Interface）驱动是连接 Kubernetes 和存储后端的桥梁。本章详细介绍如何从零开始开发一个 CSI 驱动，包括项目结构、核心接口实现、部署配置等完整流程。

## 项目结构

### 目录布局

```
csi-driver/
├── cmd/
│   └── csi-driver/
│       └── main.go              # 入口点
├── pkg/
│   ├── driver/
│   │   ├── driver.go            # 驱动主结构
│   │   ├── controller.go        # Controller 服务
│   │   ├── node.go              # Node 服务
│   │   └── identity.go          # Identity 服务
│   ├── backend/
│   │   └── storage.go           # 存储后端接口
│   └── utils/
│       └── mount.go             # 挂载工具
├── deploy/
│   ├── kubernetes/
│   │   ├── controller.yaml      # Controller 部署
│   │   ├── node.yaml            # Node DaemonSet
│   │   ├── csidriver.yaml       # CSIDriver 对象
│   │   └── rbac.yaml            # RBAC 配置
│   └── helm/
│       └── csi-driver/          # Helm Chart
├── test/
│   ├── sanity/                  # CSI Sanity 测试
│   └── e2e/                     # E2E 测试
├── go.mod
├── go.sum
├── Makefile
└── Dockerfile
```

### 依赖管理

```go
// go.mod
module github.com/example/csi-driver

go 1.22

require (
    github.com/container-storage-interface/spec v1.9.0
    google.golang.org/grpc v1.59.0
    k8s.io/klog/v2 v2.110.1
    k8s.io/mount-utils v0.29.0
    k8s.io/utils v0.0.0-20231127182322-b307cd553661
)
```

## 驱动主结构

### 入口点

```go
// cmd/csi-driver/main.go
package main

import (
    "flag"
    "os"

    "github.com/example/csi-driver/pkg/driver"
    "k8s.io/klog/v2"
)

var (
    endpoint   = flag.String("endpoint", "unix:///csi/csi.sock", "CSI endpoint")
    nodeID     = flag.String("node-id", "", "Node ID")
    driverName = flag.String("driver-name", "csi.example.com", "Driver name")

    // 存储后端配置
    backendURL = flag.String("backend-url", "", "Storage backend URL")
    backendKey = flag.String("backend-key", "", "Storage backend API key")
)

func main() {
    klog.InitFlags(nil)
    flag.Parse()

    if *nodeID == "" {
        // 从环境变量获取
        *nodeID = os.Getenv("NODE_ID")
        if *nodeID == "" {
            klog.Fatal("Node ID is required")
        }
    }

    // 创建驱动
    d, err := driver.NewDriver(driver.Config{
        Endpoint:   *endpoint,
        NodeID:     *nodeID,
        DriverName: *driverName,
        BackendURL: *backendURL,
        BackendKey: *backendKey,
    })
    if err != nil {
        klog.Fatalf("Failed to create driver: %v", err)
    }

    // 运行驱动
    if err := d.Run(); err != nil {
        klog.Fatalf("Failed to run driver: %v", err)
    }
}
```

### 驱动结构

```go
// pkg/driver/driver.go
package driver

import (
    "net"
    "net/url"
    "os"
    "sync"

    "github.com/container-storage-interface/spec/lib/go/csi"
    "github.com/example/csi-driver/pkg/backend"
    "google.golang.org/grpc"
    "k8s.io/klog/v2"
    "k8s.io/mount-utils"
)

// Config 驱动配置
type Config struct {
    Endpoint   string
    NodeID     string
    DriverName string
    BackendURL string
    BackendKey string
}

// Driver CSI 驱动主结构
type Driver struct {
    config   Config
    server   *grpc.Server
    backend  backend.Interface
    mounter  mount.Interface

    // 服务实现
    identity   *IdentityServer
    controller *ControllerServer
    node       *NodeServer

    // 状态
    ready bool
    mu    sync.Mutex
}

// NewDriver 创建驱动实例
func NewDriver(config Config) (*Driver, error) {
    // 创建存储后端客户端
    be, err := backend.NewClient(config.BackendURL, config.BackendKey)
    if err != nil {
        return nil, err
    }

    d := &Driver{
        config:  config,
        backend: be,
        mounter: mount.New(""),
    }

    // 创建服务实现
    d.identity = NewIdentityServer(d)
    d.controller = NewControllerServer(d)
    d.node = NewNodeServer(d)

    return d, nil
}

// Run 启动驱动
func (d *Driver) Run() error {
    // 解析 endpoint
    u, err := url.Parse(d.config.Endpoint)
    if err != nil {
        return err
    }

    var listener net.Listener
    switch u.Scheme {
    case "unix":
        // 删除旧的 socket 文件
        if err := os.Remove(u.Path); err != nil && !os.IsNotExist(err) {
            return err
        }
        listener, err = net.Listen("unix", u.Path)
    case "tcp":
        listener, err = net.Listen("tcp", u.Host)
    default:
        return fmt.Errorf("unsupported scheme: %s", u.Scheme)
    }
    if err != nil {
        return err
    }

    // 创建 gRPC 服务器
    d.server = grpc.NewServer(
        grpc.UnaryInterceptor(loggingInterceptor),
    )

    // 注册服务
    csi.RegisterIdentityServer(d.server, d.identity)
    csi.RegisterControllerServer(d.server, d.controller)
    csi.RegisterNodeServer(d.server, d.node)

    d.setReady(true)
    klog.Infof("CSI driver started on %s", d.config.Endpoint)

    // 启动服务
    return d.server.Serve(listener)
}

// Stop 停止驱动
func (d *Driver) Stop() {
    d.setReady(false)
    if d.server != nil {
        d.server.GracefulStop()
    }
}

func (d *Driver) setReady(ready bool) {
    d.mu.Lock()
    defer d.mu.Unlock()
    d.ready = ready
}

func (d *Driver) isReady() bool {
    d.mu.Lock()
    defer d.mu.Unlock()
    return d.ready
}

// loggingInterceptor gRPC 日志拦截器
func loggingInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler) (interface{}, error) {

    klog.V(4).Infof("GRPC call: %s", info.FullMethod)
    klog.V(5).Infof("GRPC request: %+v", req)

    resp, err := handler(ctx, req)

    if err != nil {
        klog.Errorf("GRPC error: %v", err)
    } else {
        klog.V(5).Infof("GRPC response: %+v", resp)
    }

    return resp, err
}
```

## Identity 服务

```go
// pkg/driver/identity.go
package driver

import (
    "context"

    "github.com/container-storage-interface/spec/lib/go/csi"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

// IdentityServer Identity 服务实现
type IdentityServer struct {
    csi.UnimplementedIdentityServer
    driver *Driver
}

// NewIdentityServer 创建 Identity 服务
func NewIdentityServer(d *Driver) *IdentityServer {
    return &IdentityServer{driver: d}
}

// GetPluginInfo 返回插件信息
func (s *IdentityServer) GetPluginInfo(
    ctx context.Context,
    req *csi.GetPluginInfoRequest) (*csi.GetPluginInfoResponse, error) {

    return &csi.GetPluginInfoResponse{
        Name:          s.driver.config.DriverName,
        VendorVersion: "v1.0.0",
    }, nil
}

// GetPluginCapabilities 返回插件能力
func (s *IdentityServer) GetPluginCapabilities(
    ctx context.Context,
    req *csi.GetPluginCapabilitiesRequest) (*csi.GetPluginCapabilitiesResponse, error) {

    return &csi.GetPluginCapabilitiesResponse{
        Capabilities: []*csi.PluginCapability{
            // Controller 服务
            {
                Type: &csi.PluginCapability_Service_{
                    Service: &csi.PluginCapability_Service{
                        Type: csi.PluginCapability_Service_CONTROLLER_SERVICE,
                    },
                },
            },
            // 卷访问性拓扑
            {
                Type: &csi.PluginCapability_Service_{
                    Service: &csi.PluginCapability_Service{
                        Type: csi.PluginCapability_Service_VOLUME_ACCESSIBILITY_CONSTRAINTS,
                    },
                },
            },
            // 卷扩展（在线）
            {
                Type: &csi.PluginCapability_VolumeExpansion_{
                    VolumeExpansion: &csi.PluginCapability_VolumeExpansion{
                        Type: csi.PluginCapability_VolumeExpansion_ONLINE,
                    },
                },
            },
        },
    }, nil
}

// Probe 健康检查
func (s *IdentityServer) Probe(
    ctx context.Context,
    req *csi.ProbeRequest) (*csi.ProbeResponse, error) {

    if !s.driver.isReady() {
        return nil, status.Error(codes.FailedPrecondition, "Driver not ready")
    }

    // 检查后端连接
    if err := s.driver.backend.Ping(ctx); err != nil {
        return nil, status.Errorf(codes.Unavailable, "Backend unavailable: %v", err)
    }

    return &csi.ProbeResponse{
        Ready: &wrappers.BoolValue{Value: true},
    }, nil
}
```

## Controller 服务

```go
// pkg/driver/controller.go
package driver

import (
    "context"
    "fmt"
    "strconv"

    "github.com/container-storage-interface/spec/lib/go/csi"
    "github.com/example/csi-driver/pkg/backend"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
    "k8s.io/klog/v2"
)

// ControllerServer Controller 服务实现
type ControllerServer struct {
    csi.UnimplementedControllerServer
    driver *Driver
}

// NewControllerServer 创建 Controller 服务
func NewControllerServer(d *Driver) *ControllerServer {
    return &ControllerServer{driver: d}
}

// ControllerGetCapabilities 返回 Controller 能力
func (s *ControllerServer) ControllerGetCapabilities(
    ctx context.Context,
    req *csi.ControllerGetCapabilitiesRequest) (*csi.ControllerGetCapabilitiesResponse, error) {

    capabilities := []csi.ControllerServiceCapability_RPC_Type{
        csi.ControllerServiceCapability_RPC_CREATE_DELETE_VOLUME,
        csi.ControllerServiceCapability_RPC_PUBLISH_UNPUBLISH_VOLUME,
        csi.ControllerServiceCapability_RPC_LIST_VOLUMES,
        csi.ControllerServiceCapability_RPC_GET_CAPACITY,
        csi.ControllerServiceCapability_RPC_EXPAND_VOLUME,
        csi.ControllerServiceCapability_RPC_CREATE_DELETE_SNAPSHOT,
        csi.ControllerServiceCapability_RPC_LIST_SNAPSHOTS,
        csi.ControllerServiceCapability_RPC_CLONE_VOLUME,
    }

    var caps []*csi.ControllerServiceCapability
    for _, cap := range capabilities {
        caps = append(caps, &csi.ControllerServiceCapability{
            Type: &csi.ControllerServiceCapability_Rpc{
                Rpc: &csi.ControllerServiceCapability_RPC{
                    Type: cap,
                },
            },
        })
    }

    return &csi.ControllerGetCapabilitiesResponse{Capabilities: caps}, nil
}

// CreateVolume 创建卷
func (s *ControllerServer) CreateVolume(
    ctx context.Context,
    req *csi.CreateVolumeRequest) (*csi.CreateVolumeResponse, error) {

    // 1. 参数验证
    name := req.GetName()
    if name == "" {
        return nil, status.Error(codes.InvalidArgument, "Volume name required")
    }

    caps := req.GetVolumeCapabilities()
    if len(caps) == 0 {
        return nil, status.Error(codes.InvalidArgument, "Volume capabilities required")
    }

    // 验证能力
    if err := s.validateVolumeCapabilities(caps); err != nil {
        return nil, status.Errorf(codes.InvalidArgument, "Invalid capabilities: %v", err)
    }

    // 2. 计算容量
    capacity := req.GetCapacityRange()
    size := int64(10 * 1024 * 1024 * 1024) // 默认 10Gi
    if capacity != nil {
        if capacity.RequiredBytes > 0 {
            size = capacity.RequiredBytes
        }
    }

    // 3. 检查幂等性
    existingVolume, err := s.driver.backend.GetVolumeByName(ctx, name)
    if err == nil && existingVolume != nil {
        // 卷已存在，验证参数
        if existingVolume.SizeBytes != size {
            return nil, status.Error(codes.AlreadyExists,
                "Volume exists with different size")
        }
        // 返回已有卷
        return s.buildCreateVolumeResponse(existingVolume), nil
    }

    // 4. 处理数据源（快照或克隆）
    var sourceSnapshotID, sourceVolumeID string
    if source := req.GetVolumeContentSource(); source != nil {
        if snapshot := source.GetSnapshot(); snapshot != nil {
            sourceSnapshotID = snapshot.SnapshotId
        } else if volume := source.GetVolume(); volume != nil {
            sourceVolumeID = volume.VolumeId
        }
    }

    // 5. 获取参数
    params := req.GetParameters()
    storageType := params["type"]
    if storageType == "" {
        storageType = "standard"
    }

    // 6. 确定拓扑
    var topology *csi.Topology
    if requirements := req.GetAccessibilityRequirements(); requirements != nil {
        if len(requirements.Preferred) > 0 {
            topology = requirements.Preferred[0]
        } else if len(requirements.Requisite) > 0 {
            topology = requirements.Requisite[0]
        }
    }

    zone := ""
    if topology != nil {
        zone = topology.Segments["topology.kubernetes.io/zone"]
    }

    // 7. 创建卷
    createReq := &backend.CreateVolumeRequest{
        Name:             name,
        SizeBytes:        size,
        StorageType:      storageType,
        Zone:             zone,
        SourceSnapshotID: sourceSnapshotID,
        SourceVolumeID:   sourceVolumeID,
    }

    volume, err := s.driver.backend.CreateVolume(ctx, createReq)
    if err != nil {
        klog.Errorf("Failed to create volume: %v", err)
        return nil, status.Errorf(codes.Internal, "Failed to create volume: %v", err)
    }

    klog.Infof("Created volume %s (ID: %s)", name, volume.ID)

    return s.buildCreateVolumeResponse(volume), nil
}

func (s *ControllerServer) buildCreateVolumeResponse(vol *backend.Volume) *csi.CreateVolumeResponse {
    resp := &csi.CreateVolumeResponse{
        Volume: &csi.Volume{
            VolumeId:      vol.ID,
            CapacityBytes: vol.SizeBytes,
            VolumeContext: map[string]string{
                "storage_type": vol.StorageType,
            },
        },
    }

    // 设置拓扑
    if vol.Zone != "" {
        resp.Volume.AccessibleTopology = []*csi.Topology{
            {
                Segments: map[string]string{
                    "topology.kubernetes.io/zone": vol.Zone,
                },
            },
        }
    }

    // 设置数据源
    if vol.SourceSnapshotID != "" {
        resp.Volume.ContentSource = &csi.VolumeContentSource{
            Type: &csi.VolumeContentSource_Snapshot{
                Snapshot: &csi.VolumeContentSource_SnapshotSource{
                    SnapshotId: vol.SourceSnapshotID,
                },
            },
        }
    }

    return resp
}

// DeleteVolume 删除卷
func (s *ControllerServer) DeleteVolume(
    ctx context.Context,
    req *csi.DeleteVolumeRequest) (*csi.DeleteVolumeResponse, error) {

    volumeID := req.GetVolumeId()
    if volumeID == "" {
        return nil, status.Error(codes.InvalidArgument, "Volume ID required")
    }

    // 删除卷（幂等操作）
    err := s.driver.backend.DeleteVolume(ctx, volumeID)
    if err != nil {
        if backend.IsNotFoundError(err) {
            // 卷不存在，视为成功
            return &csi.DeleteVolumeResponse{}, nil
        }
        return nil, status.Errorf(codes.Internal, "Failed to delete volume: %v", err)
    }

    klog.Infof("Deleted volume %s", volumeID)
    return &csi.DeleteVolumeResponse{}, nil
}

// ControllerPublishVolume 附着卷到节点
func (s *ControllerServer) ControllerPublishVolume(
    ctx context.Context,
    req *csi.ControllerPublishVolumeRequest) (*csi.ControllerPublishVolumeResponse, error) {

    volumeID := req.GetVolumeId()
    if volumeID == "" {
        return nil, status.Error(codes.InvalidArgument, "Volume ID required")
    }

    nodeID := req.GetNodeId()
    if nodeID == "" {
        return nil, status.Error(codes.InvalidArgument, "Node ID required")
    }

    // 检查卷是否存在
    volume, err := s.driver.backend.GetVolume(ctx, volumeID)
    if err != nil {
        if backend.IsNotFoundError(err) {
            return nil, status.Error(codes.NotFound, "Volume not found")
        }
        return nil, status.Errorf(codes.Internal, "Failed to get volume: %v", err)
    }

    // 检查只读标志
    readOnly := req.GetReadonly()

    // 附着卷到节点
    attachment, err := s.driver.backend.AttachVolume(ctx, &backend.AttachVolumeRequest{
        VolumeID: volumeID,
        NodeID:   nodeID,
        ReadOnly: readOnly,
    })
    if err != nil {
        return nil, status.Errorf(codes.Internal, "Failed to attach volume: %v", err)
    }

    klog.Infof("Attached volume %s to node %s (device: %s)", volumeID, nodeID, attachment.DevicePath)

    return &csi.ControllerPublishVolumeResponse{
        PublishContext: map[string]string{
            "device_path": attachment.DevicePath,
            "lun_id":      strconv.Itoa(attachment.LunID),
        },
    }, nil
}

// ControllerUnpublishVolume 从节点分离卷
func (s *ControllerServer) ControllerUnpublishVolume(
    ctx context.Context,
    req *csi.ControllerUnpublishVolumeRequest) (*csi.ControllerUnpublishVolumeResponse, error) {

    volumeID := req.GetVolumeId()
    if volumeID == "" {
        return nil, status.Error(codes.InvalidArgument, "Volume ID required")
    }

    nodeID := req.GetNodeId()
    if nodeID == "" {
        return nil, status.Error(codes.InvalidArgument, "Node ID required")
    }

    // 分离卷
    err := s.driver.backend.DetachVolume(ctx, volumeID, nodeID)
    if err != nil {
        if backend.IsNotFoundError(err) {
            return &csi.ControllerUnpublishVolumeResponse{}, nil
        }
        return nil, status.Errorf(codes.Internal, "Failed to detach volume: %v", err)
    }

    klog.Infof("Detached volume %s from node %s", volumeID, nodeID)
    return &csi.ControllerUnpublishVolumeResponse{}, nil
}

// ControllerExpandVolume 扩展卷
func (s *ControllerServer) ControllerExpandVolume(
    ctx context.Context,
    req *csi.ControllerExpandVolumeRequest) (*csi.ControllerExpandVolumeResponse, error) {

    volumeID := req.GetVolumeId()
    if volumeID == "" {
        return nil, status.Error(codes.InvalidArgument, "Volume ID required")
    }

    capacity := req.GetCapacityRange()
    if capacity == nil || capacity.RequiredBytes == 0 {
        return nil, status.Error(codes.InvalidArgument, "Capacity required")
    }

    newSize := capacity.RequiredBytes

    // 获取当前卷
    volume, err := s.driver.backend.GetVolume(ctx, volumeID)
    if err != nil {
        return nil, status.Errorf(codes.Internal, "Failed to get volume: %v", err)
    }

    // 检查是否需要扩展
    if volume.SizeBytes >= newSize {
        return &csi.ControllerExpandVolumeResponse{
            CapacityBytes:         volume.SizeBytes,
            NodeExpansionRequired: false,
        }, nil
    }

    // 执行扩展
    err = s.driver.backend.ExpandVolume(ctx, volumeID, newSize)
    if err != nil {
        return nil, status.Errorf(codes.Internal, "Failed to expand volume: %v", err)
    }

    klog.Infof("Expanded volume %s to %d bytes", volumeID, newSize)

    return &csi.ControllerExpandVolumeResponse{
        CapacityBytes:         newSize,
        NodeExpansionRequired: true, // 需要节点端扩展文件系统
    }, nil
}

// CreateSnapshot 创建快照
func (s *ControllerServer) CreateSnapshot(
    ctx context.Context,
    req *csi.CreateSnapshotRequest) (*csi.CreateSnapshotResponse, error) {

    name := req.GetName()
    if name == "" {
        return nil, status.Error(codes.InvalidArgument, "Snapshot name required")
    }

    sourceVolumeID := req.GetSourceVolumeId()
    if sourceVolumeID == "" {
        return nil, status.Error(codes.InvalidArgument, "Source volume ID required")
    }

    // 检查幂等性
    existing, err := s.driver.backend.GetSnapshotByName(ctx, name)
    if err == nil && existing != nil {
        if existing.SourceVolumeID != sourceVolumeID {
            return nil, status.Error(codes.AlreadyExists,
                "Snapshot exists with different source volume")
        }
        return s.buildCreateSnapshotResponse(existing), nil
    }

    // 创建快照
    snapshot, err := s.driver.backend.CreateSnapshot(ctx, &backend.CreateSnapshotRequest{
        Name:           name,
        SourceVolumeID: sourceVolumeID,
        Parameters:     req.GetParameters(),
    })
    if err != nil {
        return nil, status.Errorf(codes.Internal, "Failed to create snapshot: %v", err)
    }

    klog.Infof("Created snapshot %s from volume %s", name, sourceVolumeID)

    return s.buildCreateSnapshotResponse(snapshot), nil
}

func (s *ControllerServer) buildCreateSnapshotResponse(snap *backend.Snapshot) *csi.CreateSnapshotResponse {
    return &csi.CreateSnapshotResponse{
        Snapshot: &csi.Snapshot{
            SnapshotId:     snap.ID,
            SourceVolumeId: snap.SourceVolumeID,
            CreationTime:   timestamppb.New(snap.CreationTime),
            SizeBytes:      snap.SizeBytes,
            ReadyToUse:     snap.ReadyToUse,
        },
    }
}

// DeleteSnapshot 删除快照
func (s *ControllerServer) DeleteSnapshot(
    ctx context.Context,
    req *csi.DeleteSnapshotRequest) (*csi.DeleteSnapshotResponse, error) {

    snapshotID := req.GetSnapshotId()
    if snapshotID == "" {
        return nil, status.Error(codes.InvalidArgument, "Snapshot ID required")
    }

    err := s.driver.backend.DeleteSnapshot(ctx, snapshotID)
    if err != nil {
        if backend.IsNotFoundError(err) {
            return &csi.DeleteSnapshotResponse{}, nil
        }
        return nil, status.Errorf(codes.Internal, "Failed to delete snapshot: %v", err)
    }

    klog.Infof("Deleted snapshot %s", snapshotID)
    return &csi.DeleteSnapshotResponse{}, nil
}

// validateVolumeCapabilities 验证卷能力
func (s *ControllerServer) validateVolumeCapabilities(caps []*csi.VolumeCapability) error {
    for _, cap := range caps {
        // 检查访问模式
        accessMode := cap.GetAccessMode()
        if accessMode == nil {
            return fmt.Errorf("access mode required")
        }

        switch accessMode.GetMode() {
        case csi.VolumeCapability_AccessMode_SINGLE_NODE_WRITER,
            csi.VolumeCapability_AccessMode_SINGLE_NODE_READER_ONLY:
            // 支持的模式
        default:
            return fmt.Errorf("unsupported access mode: %v", accessMode.GetMode())
        }

        // 检查卷类型
        if mount := cap.GetMount(); mount != nil {
            // 支持文件系统挂载
        } else if block := cap.GetBlock(); block != nil {
            // 支持块设备
        } else {
            return fmt.Errorf("must specify mount or block access type")
        }
    }
    return nil
}
```

## Node 服务

```go
// pkg/driver/node.go
package driver

import (
    "context"
    "fmt"
    "os"
    "path/filepath"

    "github.com/container-storage-interface/spec/lib/go/csi"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
    "k8s.io/klog/v2"
    "k8s.io/mount-utils"
)

// NodeServer Node 服务实现
type NodeServer struct {
    csi.UnimplementedNodeServer
    driver  *Driver
    mounter mount.Interface
}

// NewNodeServer 创建 Node 服务
func NewNodeServer(d *Driver) *NodeServer {
    return &NodeServer{
        driver:  d,
        mounter: d.mounter,
    }
}

// NodeGetCapabilities 返回 Node 能力
func (s *NodeServer) NodeGetCapabilities(
    ctx context.Context,
    req *csi.NodeGetCapabilitiesRequest) (*csi.NodeGetCapabilitiesResponse, error) {

    capabilities := []csi.NodeServiceCapability_RPC_Type{
        csi.NodeServiceCapability_RPC_STAGE_UNSTAGE_VOLUME,
        csi.NodeServiceCapability_RPC_EXPAND_VOLUME,
        csi.NodeServiceCapability_RPC_GET_VOLUME_STATS,
    }

    var caps []*csi.NodeServiceCapability
    for _, cap := range capabilities {
        caps = append(caps, &csi.NodeServiceCapability{
            Type: &csi.NodeServiceCapability_Rpc{
                Rpc: &csi.NodeServiceCapability_RPC{
                    Type: cap,
                },
            },
        })
    }

    return &csi.NodeGetCapabilitiesResponse{Capabilities: caps}, nil
}

// NodeGetInfo 返回节点信息
func (s *NodeServer) NodeGetInfo(
    ctx context.Context,
    req *csi.NodeGetInfoRequest) (*csi.NodeGetInfoResponse, error) {

    // 获取节点拓扑信息
    zone := os.Getenv("NODE_ZONE")
    region := os.Getenv("NODE_REGION")

    topology := &csi.Topology{
        Segments: map[string]string{
            "topology.kubernetes.io/zone":   zone,
            "topology.kubernetes.io/region": region,
        },
    }

    return &csi.NodeGetInfoResponse{
        NodeId:             s.driver.config.NodeID,
        MaxVolumesPerNode:  256,
        AccessibleTopology: topology,
    }, nil
}

// NodeStageVolume 阶段挂载卷
func (s *NodeServer) NodeStageVolume(
    ctx context.Context,
    req *csi.NodeStageVolumeRequest) (*csi.NodeStageVolumeResponse, error) {

    volumeID := req.GetVolumeId()
    if volumeID == "" {
        return nil, status.Error(codes.InvalidArgument, "Volume ID required")
    }

    stagingPath := req.GetStagingTargetPath()
    if stagingPath == "" {
        return nil, status.Error(codes.InvalidArgument, "Staging path required")
    }

    cap := req.GetVolumeCapability()
    if cap == nil {
        return nil, status.Error(codes.InvalidArgument, "Volume capability required")
    }

    // 获取发布上下文
    publishContext := req.GetPublishContext()
    devicePath := publishContext["device_path"]
    if devicePath == "" {
        return nil, status.Error(codes.InvalidArgument, "Device path not found in publish context")
    }

    // 等待设备就绪
    if err := s.waitForDevice(devicePath); err != nil {
        return nil, status.Errorf(codes.Internal, "Device not ready: %v", err)
    }

    // 检查是否已挂载
    mounted, err := s.mounter.IsMountPoint(stagingPath)
    if err != nil {
        if !os.IsNotExist(err) {
            return nil, status.Errorf(codes.Internal, "Failed to check mount point: %v", err)
        }
        // 创建目录
        if err := os.MkdirAll(stagingPath, 0750); err != nil {
            return nil, status.Errorf(codes.Internal, "Failed to create staging path: %v", err)
        }
    }

    if mounted {
        klog.Infof("Volume %s already staged at %s", volumeID, stagingPath)
        return &csi.NodeStageVolumeResponse{}, nil
    }

    // 处理块设备或文件系统
    if block := cap.GetBlock(); block != nil {
        // 块设备模式：创建符号链接
        if err := os.Symlink(devicePath, stagingPath); err != nil {
            return nil, status.Errorf(codes.Internal, "Failed to create symlink: %v", err)
        }
    } else if mnt := cap.GetMount(); mnt != nil {
        // 文件系统模式：格式化并挂载
        fsType := mnt.GetFsType()
        if fsType == "" {
            fsType = "ext4"
        }

        // 格式化（如果需要）
        formatted, err := s.isFormatted(devicePath)
        if err != nil {
            return nil, status.Errorf(codes.Internal, "Failed to check format: %v", err)
        }
        if !formatted {
            if err := s.formatDevice(devicePath, fsType); err != nil {
                return nil, status.Errorf(codes.Internal, "Failed to format device: %v", err)
            }
        }

        // 挂载
        mountOptions := mnt.GetMountFlags()
        if err := s.mounter.Mount(devicePath, stagingPath, fsType, mountOptions); err != nil {
            return nil, status.Errorf(codes.Internal, "Failed to mount: %v", err)
        }
    }

    klog.Infof("Staged volume %s at %s", volumeID, stagingPath)
    return &csi.NodeStageVolumeResponse{}, nil
}

// NodeUnstageVolume 取消阶段挂载
func (s *NodeServer) NodeUnstageVolume(
    ctx context.Context,
    req *csi.NodeUnstageVolumeRequest) (*csi.NodeUnstageVolumeResponse, error) {

    volumeID := req.GetVolumeId()
    if volumeID == "" {
        return nil, status.Error(codes.InvalidArgument, "Volume ID required")
    }

    stagingPath := req.GetStagingTargetPath()
    if stagingPath == "" {
        return nil, status.Error(codes.InvalidArgument, "Staging path required")
    }

    // 检查是否挂载
    mounted, err := s.mounter.IsMountPoint(stagingPath)
    if err != nil {
        if os.IsNotExist(err) {
            return &csi.NodeUnstageVolumeResponse{}, nil
        }
        return nil, status.Errorf(codes.Internal, "Failed to check mount point: %v", err)
    }

    if mounted {
        // 卸载
        if err := s.mounter.Unmount(stagingPath); err != nil {
            return nil, status.Errorf(codes.Internal, "Failed to unmount: %v", err)
        }
    }

    // 删除目录
    if err := os.Remove(stagingPath); err != nil && !os.IsNotExist(err) {
        return nil, status.Errorf(codes.Internal, "Failed to remove staging path: %v", err)
    }

    klog.Infof("Unstaged volume %s from %s", volumeID, stagingPath)
    return &csi.NodeUnstageVolumeResponse{}, nil
}

// NodePublishVolume 发布卷到目标路径
func (s *NodeServer) NodePublishVolume(
    ctx context.Context,
    req *csi.NodePublishVolumeRequest) (*csi.NodePublishVolumeResponse, error) {

    volumeID := req.GetVolumeId()
    if volumeID == "" {
        return nil, status.Error(codes.InvalidArgument, "Volume ID required")
    }

    targetPath := req.GetTargetPath()
    if targetPath == "" {
        return nil, status.Error(codes.InvalidArgument, "Target path required")
    }

    stagingPath := req.GetStagingTargetPath()
    cap := req.GetVolumeCapability()
    readOnly := req.GetReadonly()

    // 检查是否已挂载
    mounted, err := s.mounter.IsMountPoint(targetPath)
    if err != nil {
        if !os.IsNotExist(err) {
            return nil, status.Errorf(codes.Internal, "Failed to check mount point: %v", err)
        }
    }

    if mounted {
        klog.Infof("Volume %s already published at %s", volumeID, targetPath)
        return &csi.NodePublishVolumeResponse{}, nil
    }

    // 创建目标目录
    if block := cap.GetBlock(); block != nil {
        // 块设备：创建父目录
        if err := os.MkdirAll(filepath.Dir(targetPath), 0750); err != nil {
            return nil, status.Errorf(codes.Internal, "Failed to create target dir: %v", err)
        }
    } else {
        // 文件系统：创建挂载点
        if err := os.MkdirAll(targetPath, 0750); err != nil {
            return nil, status.Errorf(codes.Internal, "Failed to create target path: %v", err)
        }
    }

    // 执行 bind mount
    mountOptions := []string{"bind"}
    if readOnly {
        mountOptions = append(mountOptions, "ro")
    }
    if mnt := cap.GetMount(); mnt != nil {
        mountOptions = append(mountOptions, mnt.GetMountFlags()...)
    }

    if err := s.mounter.Mount(stagingPath, targetPath, "", mountOptions); err != nil {
        return nil, status.Errorf(codes.Internal, "Failed to bind mount: %v", err)
    }

    klog.Infof("Published volume %s at %s", volumeID, targetPath)
    return &csi.NodePublishVolumeResponse{}, nil
}

// NodeUnpublishVolume 取消发布卷
func (s *NodeServer) NodeUnpublishVolume(
    ctx context.Context,
    req *csi.NodeUnpublishVolumeRequest) (*csi.NodeUnpublishVolumeResponse, error) {

    volumeID := req.GetVolumeId()
    if volumeID == "" {
        return nil, status.Error(codes.InvalidArgument, "Volume ID required")
    }

    targetPath := req.GetTargetPath()
    if targetPath == "" {
        return nil, status.Error(codes.InvalidArgument, "Target path required")
    }

    // 卸载
    if err := mount.CleanupMountPoint(targetPath, s.mounter, true); err != nil {
        return nil, status.Errorf(codes.Internal, "Failed to cleanup mount point: %v", err)
    }

    klog.Infof("Unpublished volume %s from %s", volumeID, targetPath)
    return &csi.NodeUnpublishVolumeResponse{}, nil
}

// NodeExpandVolume 节点端扩展卷
func (s *NodeServer) NodeExpandVolume(
    ctx context.Context,
    req *csi.NodeExpandVolumeRequest) (*csi.NodeExpandVolumeResponse, error) {

    volumeID := req.GetVolumeId()
    if volumeID == "" {
        return nil, status.Error(codes.InvalidArgument, "Volume ID required")
    }

    volumePath := req.GetVolumePath()
    if volumePath == "" {
        return nil, status.Error(codes.InvalidArgument, "Volume path required")
    }

    // 获取设备路径
    devicePath, _, err := mount.GetDeviceNameFromMount(s.mounter, volumePath)
    if err != nil {
        return nil, status.Errorf(codes.Internal, "Failed to get device: %v", err)
    }

    // 扩展文件系统
    resizer := mount.NewResizeFs(s.mounter.(*mount.SafeFormatAndMount).Exec)
    if _, err := resizer.Resize(devicePath, volumePath); err != nil {
        return nil, status.Errorf(codes.Internal, "Failed to resize filesystem: %v", err)
    }

    // 获取新大小
    newSize, err := s.getFilesystemSize(volumePath)
    if err != nil {
        return nil, status.Errorf(codes.Internal, "Failed to get filesystem size: %v", err)
    }

    klog.Infof("Expanded volume %s filesystem to %d bytes", volumeID, newSize)

    return &csi.NodeExpandVolumeResponse{
        CapacityBytes: newSize,
    }, nil
}

// NodeGetVolumeStats 获取卷统计信息
func (s *NodeServer) NodeGetVolumeStats(
    ctx context.Context,
    req *csi.NodeGetVolumeStatsRequest) (*csi.NodeGetVolumeStatsResponse, error) {

    volumeID := req.GetVolumeId()
    if volumeID == "" {
        return nil, status.Error(codes.InvalidArgument, "Volume ID required")
    }

    volumePath := req.GetVolumePath()
    if volumePath == "" {
        return nil, status.Error(codes.InvalidArgument, "Volume path required")
    }

    // 获取文件系统统计
    var stat unix.Statfs_t
    if err := unix.Statfs(volumePath, &stat); err != nil {
        return nil, status.Errorf(codes.Internal, "Failed to statfs: %v", err)
    }

    return &csi.NodeGetVolumeStatsResponse{
        Usage: []*csi.VolumeUsage{
            {
                Available: int64(stat.Bavail) * int64(stat.Bsize),
                Total:     int64(stat.Blocks) * int64(stat.Bsize),
                Used:      int64(stat.Blocks-stat.Bfree) * int64(stat.Bsize),
                Unit:      csi.VolumeUsage_BYTES,
            },
            {
                Available: int64(stat.Ffree),
                Total:     int64(stat.Files),
                Used:      int64(stat.Files - stat.Ffree),
                Unit:      csi.VolumeUsage_INODES,
            },
        },
    }, nil
}

// 辅助方法
func (s *NodeServer) waitForDevice(devicePath string) error {
    // 等待设备出现
    for i := 0; i < 30; i++ {
        if _, err := os.Stat(devicePath); err == nil {
            return nil
        }
        time.Sleep(time.Second)
    }
    return fmt.Errorf("device %s not found", devicePath)
}

func (s *NodeServer) isFormatted(devicePath string) (bool, error) {
    // 使用 blkid 检查
    output, err := exec.Command("blkid", devicePath).CombinedOutput()
    if err != nil {
        // 未格式化
        return false, nil
    }
    return len(output) > 0, nil
}

func (s *NodeServer) formatDevice(devicePath, fsType string) error {
    var cmd *exec.Cmd
    switch fsType {
    case "ext4":
        cmd = exec.Command("mkfs.ext4", "-F", devicePath)
    case "xfs":
        cmd = exec.Command("mkfs.xfs", "-f", devicePath)
    default:
        return fmt.Errorf("unsupported filesystem type: %s", fsType)
    }
    return cmd.Run()
}

func (s *NodeServer) getFilesystemSize(path string) (int64, error) {
    var stat unix.Statfs_t
    if err := unix.Statfs(path, &stat); err != nil {
        return 0, err
    }
    return int64(stat.Blocks) * int64(stat.Bsize), nil
}
```

## 部署配置

### Controller 部署

```yaml
# deploy/kubernetes/controller.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: csi-controller
  namespace: kube-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: csi-controller
  template:
    metadata:
      labels:
        app: csi-controller
    spec:
      serviceAccountName: csi-controller-sa
      priorityClassName: system-cluster-critical
      containers:
        # CSI 驱动
        - name: csi-driver
          image: example/csi-driver:v1.0.0
          args:
            - --endpoint=unix:///csi/csi.sock
            - --driver-name=csi.example.com
            - --backend-url=$(BACKEND_URL)
          env:
            - name: BACKEND_URL
              valueFrom:
                secretKeyRef:
                  name: csi-backend-secret
                  key: url
            - name: BACKEND_KEY
              valueFrom:
                secretKeyRef:
                  name: csi-backend-secret
                  key: key
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
          resources:
            limits:
              memory: 256Mi
            requests:
              cpu: 10m
              memory: 64Mi

        # external-provisioner
        - name: csi-provisioner
          image: registry.k8s.io/sig-storage/csi-provisioner:v3.6.0
          args:
            - --csi-address=/csi/csi.sock
            - --feature-gates=Topology=true
            - --leader-election
            - --leader-election-namespace=kube-system
          volumeMounts:
            - name: socket-dir
              mountPath: /csi

        # external-attacher
        - name: csi-attacher
          image: registry.k8s.io/sig-storage/csi-attacher:v4.4.0
          args:
            - --csi-address=/csi/csi.sock
            - --leader-election
            - --leader-election-namespace=kube-system
          volumeMounts:
            - name: socket-dir
              mountPath: /csi

        # external-snapshotter
        - name: csi-snapshotter
          image: registry.k8s.io/sig-storage/csi-snapshotter:v6.3.0
          args:
            - --csi-address=/csi/csi.sock
            - --leader-election
            - --leader-election-namespace=kube-system
          volumeMounts:
            - name: socket-dir
              mountPath: /csi

        # external-resizer
        - name: csi-resizer
          image: registry.k8s.io/sig-storage/csi-resizer:v1.9.0
          args:
            - --csi-address=/csi/csi.sock
            - --leader-election
            - --leader-election-namespace=kube-system
          volumeMounts:
            - name: socket-dir
              mountPath: /csi

      volumes:
        - name: socket-dir
          emptyDir: {}
```

### Node DaemonSet

```yaml
# deploy/kubernetes/node.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: csi-node
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: csi-node
  template:
    metadata:
      labels:
        app: csi-node
    spec:
      serviceAccountName: csi-node-sa
      priorityClassName: system-node-critical
      hostNetwork: true
      containers:
        # CSI 驱动
        - name: csi-driver
          image: example/csi-driver:v1.0.0
          args:
            - --endpoint=unix:///csi/csi.sock
            - --driver-name=csi.example.com
            - --node-id=$(NODE_ID)
          env:
            - name: NODE_ID
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: NODE_ZONE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.labels['topology.kubernetes.io/zone']
          securityContext:
            privileged: true
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
            - name: pods-mount-dir
              mountPath: /var/lib/kubelet
              mountPropagation: Bidirectional
            - name: device-dir
              mountPath: /dev
          resources:
            limits:
              memory: 256Mi
            requests:
              cpu: 10m
              memory: 64Mi

        # node-driver-registrar
        - name: node-driver-registrar
          image: registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.9.0
          args:
            - --csi-address=/csi/csi.sock
            - --kubelet-registration-path=/var/lib/kubelet/plugins/csi.example.com/csi.sock
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
            - name: registration-dir
              mountPath: /registration

      volumes:
        - name: socket-dir
          hostPath:
            path: /var/lib/kubelet/plugins/csi.example.com
            type: DirectoryOrCreate
        - name: pods-mount-dir
          hostPath:
            path: /var/lib/kubelet
            type: Directory
        - name: device-dir
          hostPath:
            path: /dev
            type: Directory
        - name: registration-dir
          hostPath:
            path: /var/lib/kubelet/plugins_registry
            type: Directory
```

### CSIDriver 对象

```yaml
# deploy/kubernetes/csidriver.yaml
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: csi.example.com
spec:
  # 附着操作是否需要
  attachRequired: true
  # 是否支持 Pod 信息传递
  podInfoOnMount: true
  # 卷生命周期模式
  volumeLifecycleModes:
    - Persistent
    - Ephemeral
  # 存储容量追踪
  storageCapacity: true
  # 文件系统组策略
  fsGroupPolicy: File
```

## 测试

### CSI Sanity 测试

```go
// test/sanity/sanity_test.go
package sanity

import (
    "testing"

    "github.com/kubernetes-csi/csi-test/v5/pkg/sanity"
)

func TestSanity(t *testing.T) {
    config := sanity.NewTestConfig()
    config.Address = "unix:///tmp/csi.sock"
    config.TargetPath = "/tmp/csi-target"
    config.StagingPath = "/tmp/csi-staging"

    // 运行测试
    sanity.Test(t, config)
}
```

### 运行测试

```bash
# 启动驱动
./csi-driver --endpoint=unix:///tmp/csi.sock --node-id=test-node &

# 运行 sanity 测试
go test -v ./test/sanity/...
```

## 总结

CSI 驱动开发涉及：
- **三个核心服务**：Identity、Controller、Node
- **gRPC 接口实现**：遵循 CSI 规范
- **Sidecar 容器**：与 Kubernetes 集成
- **部署配置**：Controller Deployment + Node DaemonSet

开发 CSI 驱动时需要注意：
- 实现幂等性，支持重复调用
- 正确处理错误码（gRPC status codes）
- 使用 CSI Sanity 测试验证实现
- 考虑拓扑感知和容量追踪
