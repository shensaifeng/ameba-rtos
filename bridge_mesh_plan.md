# Matter Bridge + BLE Mesh 集成开发计划 (5 步) — 修订版

基于 Ameba RTL8721F (AmebaGreen2) 的 `bridge_dm` 示例。  
Bridge 以 BLE Mesh **Device** 角色加入 Mesh 网络，通过外部注入获取 Mesh 节点列表，
将这些节点动态注册为 Matter 虚拟端点，并实现双向命令/状态桥接。

---

## ✅ Step 1 — Matter + BLE Mesh 共同编译 & 启动（已完成）

- Kconfig：`CONFIG_BT_MESH_DEVICE_MATTER`、`CONFIG_BLE_MATTER_ADAPTER`
- 修复 CMake target name 冲突：`ble_mesh_device` → `ble_matter_adapter_peripheral`
- 关闭 WiFi fast reconnect（`fast_reconnect_en = 0`）避免初始化时序冲突
- `matter_blemesh_post_commissioning_init()`：commissioning 完成后自动切到 Mesh Device 模式
- `FabricCount()` 判断：重启后已 commissioned 则直接起 Mesh

---

## 🔜 Step 2 — 实现 Mesh 节点注入接口

Bridge 作为 Mesh Device 无法主动枚举网络拓扑，需由外部（AT命令 / 串口 / Provisioner vendor message）
将 Mesh 节点信息注入 Bridge。

- 定义数据结构：`MeshNodeInfo { uint16_t unicast_addr; uint8_t device_type; char name[32]; }`
- 实现公共 API：
  - `matter_blemesh_node_inject(const MeshNodeInfo *node)`
  - `matter_blemesh_node_remove(uint16_t unicast_addr)`
- 节点列表全局静态数组（最多 `MESH_MAX_NODES` 条），线程安全（FreeRTOS mutex）
- 注入后通过 FreeRTOS queue 通知 `matter_customer_bridge_code`
- **涉及文件**：新增 `common/bluetooth/matter_blemesh_node_table.c/h`

---

## 🔜 Step 3 — 动态创建 / 删除 Matter 虚拟端点

根据注入的节点列表，驱动 MatterBridge 创建或删除虚拟端点。

- 修改 `matter_customer_bridge_code`：从 queue 读取注入/删除事件
- 注入事件：
  - `new MatterBridgedDeviceOnOff(name, location)`
  - `bridge.addBridgedEndpoint(config, deviceTypes)` 拿到 `EndpointId`
  - 将 `unicast_addr → {dev*, EndpointId}` 写入映射表
- 删除事件：
  - `bridge.removeBridgedEndpoint(endpointId)`
  - `delete dev`，从映射表移除
- 替换掉当前静态 `ALight1` + 20秒删除的演示逻辑
- **涉及文件**：`example_matter_bridge.cpp`、`matter_drivers.cpp/h`

---

## 🔜 Step 4 — Upstream: Matter 命令 → Mesh 发送

Matter Controller 发来命令，Bridge 转发到对应 Mesh 节点。

- `matter_driver_uplink_update_handler()` 用 `path.mEndpointId` 查映射表 → 得到 `unicast_addr`
- 在 `matter_blemesh_common.c` 中注册 Generic OnOff **Client** 模型
- 调用 `rtk_bt_mesh_generic_onoff_client_set(unicast_addr, on_off, ...)` 发送 Mesh 命令
- **涉及文件**：`matter_drivers.cpp`、`matter_blemesh_common.c`

---

## 🔜 Step 5 — Downstream: Mesh 状态 → Matter 属性上报

Mesh 节点状态变化时，Bridge 上报给 Matter Controller。

- `matter_blemesh_generic_onoff_server_callback` 收到 `EVT_SET`（或 Status 消息）时，
  从消息 source addr 查映射表 → 找到 `MatterBridgedDeviceOnOff*`
- 通过 `PlatformMgr().ScheduleWork()` 在 Matter 线程中调用 `dev->Set(state, true)`
- `HandleDeviceOnOffStatusChanged` → `ScheduleReportingCallback()` → 上报属性给 Controller
- **涉及文件**：`matter_blemesh_common.c`、`matter_drivers.cpp`

---

## 文件索引

| 文件 | 作用 |
|------|------|
| `common/bluetooth/matter_blemesh_common.c/h` | Mesh 初始化、Client/Server 模型回调 |
| `common/bluetooth/matter_blemesh_node_table.c/h` | 节点注入 API、节点列表、注入事件 queue（新增） |
| `examples/bridge_dm/example_matter_bridge.cpp` | bridge 初始化、`matter_customer_bridge_code` 主循环 |
| `examples/bridge_dm/matter_drivers.cpp/h` | Matter 事件处理、uplink/downlink handler、映射表 |
| `drivers/device/bridge_dm_driver.h` | `MatterBridge`、`MatterBridgedDeviceOnOff` 类定义 |
