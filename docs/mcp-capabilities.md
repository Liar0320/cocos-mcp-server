# Cocos Creator MCP Server 当前 MCP 能力文档

> 文档性质：基于当前源码注册结果整理  
> 基线时间：2026-04-08  
> 统计口径：`source/tools/*.ts` 当前 `getTools()` 输出，共 157 个工具 / 14 个分类

## 1. 总览

当前项目是一个运行在 Cocos Creator 内部的扩展，负责在本地启动 HTTP 服务，并把编辑器能力封装为 MCP Tools 暴露给外部 AI 客户端。

核心调用链如下：

1. 扩展加载时初始化 `MCPServer` 与 `ToolManager`。
2. 面板控制服务启停和工具启用状态。
3. HTTP 服务对外暴露 `/mcp`、`/health`、`/api/tools` 和简化 `/api/{category}/{tool}`。
4. 每个工具分类类通过 `getTools()` 输出名称、描述和输入 Schema。
5. `tools/call` 根据 `{category}_{tool}` 路由到对应工具执行器。

## 2. 协议与入口

### 2.1 HTTP 服务

- 监听地址：`127.0.0.1`
- 默认端口：`3000`
- 健康检查：`GET /health`
- MCP 入口：`POST /mcp`
- 简化工具列表：`GET /api/tools`
- 简化调用入口：`POST /api/{category}/{toolName}`

### 2.2 MCP 方法

| 方法 | 说明 |
| --- | --- |
| `initialize` | 返回 `protocolVersion: 2024-11-05` 与 `serverInfo` |
| `tools/list` | 返回当前启用工具列表 |
| `tools/call` | 执行某个工具 |

### 2.3 MCP 调用示例

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "scene_get_current_scene",
    "arguments": {}
  }
}
```

### 2.4 简化 HTTP API 示例

```bash
curl -X POST http://127.0.0.1:3000/api/node/create_node \
  -H "Content-Type: application/json" \
  -d '{
    "name": "EnemyRoot",
    "parentUuid": "target-parent-uuid",
    "nodeType": "2DNode"
  }'
```

## 3. 结果格式约定

### 3.1 MCP 返回

`tools/call` 的业务结果会被包装成：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"success\":true,\"data\":{...}}"
      }
    ]
  }
}
```

调用方通常需要再解析一层 `content[0].text`。

### 3.2 简化 API 返回

```json
{
  "success": true,
  "tool": "node_create_node",
  "result": {
    "success": true,
    "data": {}
  }
}
```

## 4. 运行配置

| 配置项 | 默认值 | 说明 |
| --- | --- | --- |
| `port` | `3000` | HTTP 监听端口 |
| `autoStart` | `false` | 扩展加载后是否自动启动 |
| `enableDebugLog` | `false` | 调试日志开关 |
| `allowedOrigins` | `["*"]` | 已定义，但当前服务端未按该字段动态控制 CORS |
| `maxConnections` | `10` | 已定义，但当前 HTTP 服务未实现连接数限制 |

## 5. 工具命名与分类规则

- 工具对外名称格式：`{category}_{toolName}`
- 分类名既用于 MCP 工具名前缀，也用于简化 API 的路径段
- 部分分类使用 camelCase：
  - `sceneAdvanced`
  - `sceneView`
  - `referenceImage`
  - `assetAdvanced`

## 6. 能力分布

| 分类 | 数量 | 能力摘要 |
| --- | ---: | --- |
| `scene` | 8 | 场景打开、保存、创建、关闭、层级读取 |
| `node` | 11 | 节点生命周期、查询、属性与层级变更 |
| `component` | 7 | 组件增删查改与脚本挂载 |
| `prefab` | 10 | 预制体浏览、实例化、创建、同步、还原 |
| `project` | 24 | 项目运行/构建与基础资源操作 |
| `debug` | 10 | 控制台、日志、脚本执行、编辑器信息 |
| `preferences` | 7 | 偏好设置面板、配置查询与导入导出 |
| `server` | 6 | 编辑器服务状态与网络环境查询 |
| `broadcast` | 5 | 广播消息监听与日志 |
| `sceneAdvanced` | 23 | Undo、数组属性、快照、组件/脚本执行 |
| `sceneView` | 20 | Gizmo、视图模式、网格、镜头控制 |
| `referenceImage` | 12 | 参考图导入、切换、缩放、透明度调整 |
| `assetAdvanced` | 11 | 资源依赖、批量处理、清单导出 |
| `validation` | 3 | JSON/MCP 请求安全格式化 |

## 7. 关键调用建议

### 7.1 节点与组件

- 创建节点时建议始终传 `parentUuid`。
- 删除组件时 `componentType` 必须使用 `get_components` 返回的 `type/cid`，不能直接传脚本类名。
- 修改节点基础字段优先用 `node_set_node_property`。
- 修改位置/旋转/缩放优先用 `node_set_node_transform`。

### 7.2 预制体

- 创建预制体至少需要 `nodeUuid`、`savePath`、`prefabName`。
- 还原预制体实例时常用 `prefab_restore_prefab_node` 或 `sceneAdvanced_restore_prefab`。

### 7.3 资源

- 资源基础 CRUD 走 `project_*`。
- 依赖分析、批量导入删除、纹理压缩、清单导出走 `assetAdvanced_*`。

### 7.4 调试与防错

- AI 在拼接复杂 JSON 参数前可先调用 `validation_validate_json_params`。
- 若字符串中有引号、换行等特殊字符，可先走 `validation_safe_string_value`。
- 需要生成完整 MCP 请求体时，可用 `validation_format_mcp_request`。

## 8. 已知实现备注

- 服务端固定监听 `127.0.0.1`，这是当前实现的安全边界。
- `clients` 字段目前总是 `0`，因为当前实现是无状态 HTTP。
- `broadcast` 当前以模拟监听为主，底层 Editor 事件订阅尚未完全启用。
- 内置 `curl` 示例生成函数当前硬编码端口 `8585`，与默认端口 `3000` 不一致，实际使用应以面板配置端口为准。
- 仓库里的旧文档和测试辅助脚本仍有版本漂移，不宜直接作为当前能力来源。

## 9. 完整工具清单

### 9.1 场景工具（scene，8）

- `scene_get_current_scene` | Get current scene information | 必填：-
- `scene_get_scene_list` | Get all scenes in the project | 必填：-
- `scene_open_scene` | Open a scene by path | 必填：`scenePath`
- `scene_save_scene` | Save current scene | 必填：-
- `scene_create_scene` | Create a new scene asset | 必填：`sceneName`, `savePath`
- `scene_save_scene_as` | Save scene as new file | 必填：`path`
- `scene_close_scene` | Close current scene | 必填：-
- `scene_get_scene_hierarchy` | Get the complete hierarchy of current scene | 必填：-

### 9.2 节点工具（node，11）

- `node_create_node` | Create a new node in the scene; supports empty node, component preset or asset instantiation | 必填：`name`
- `node_get_node_info` | Get node information by UUID | 必填：`uuid`
- `node_find_nodes` | Find nodes by name pattern | 必填：`pattern`
- `node_find_node_by_name` | Find first node by exact name | 必填：`name`
- `node_get_all_nodes` | Get all nodes in the scene with UUIDs | 必填：-
- `node_set_node_property` | Set node property value | 必填：`uuid`, `property`, `value`
- `node_set_node_transform` | Set node transform properties with unified interface | 必填：`uuid`
- `node_delete_node` | Delete a node from scene | 必填：`uuid`
- `node_move_node` | Move node to new parent | 必填：`nodeUuid`, `newParentUuid`
- `node_duplicate_node` | Duplicate a node | 必填：`uuid`
- `node_detect_node_type` | Detect whether node is 2D or 3D | 必填：`uuid`

### 9.3 组件工具（component，7）

- `component_add_component` | Add a component to a specific node | 必填：`nodeUuid`, `componentType`
- `component_remove_component` | Remove component by component `cid/type` | 必填：`nodeUuid`, `componentType`
- `component_get_components` | Get all components on a node | 必填：`nodeUuid`
- `component_get_component_info` | Get specific component information | 必填：`nodeUuid`, `componentType`
- `component_set_component_property` | Set built-in or custom component property | 必填：`nodeUuid`, `componentType`, `property`, `propertyType`, `value`
- `component_attach_script` | Attach script component by asset path | 必填：`nodeUuid`, `scriptPath`
- `component_get_available_components` | List available component types | 必填：-

### 9.4 预制体工具（prefab，10）

- `prefab_get_prefab_list` | Get all prefabs in project | 必填：-
- `prefab_load_prefab` | Load prefab by path | 必填：`prefabPath`
- `prefab_instantiate_prefab` | Instantiate prefab in scene | 必填：`prefabPath`
- `prefab_create_prefab` | Create prefab from node | 必填：`nodeUuid`, `savePath`, `prefabName`
- `prefab_update_prefab` | Update an existing prefab from node | 必填：`prefabPath`, `nodeUuid`
- `prefab_revert_prefab` | Revert prefab instance to original | 必填：`nodeUuid`
- `prefab_get_prefab_info` | Get detailed prefab information | 必填：`prefabPath`
- `prefab_validate_prefab` | Validate prefab file format | 必填：`prefabPath`
- `prefab_duplicate_prefab` | Duplicate prefab to a new path | 必填：`sourcePrefabPath`, `targetPrefabPath`
- `prefab_restore_prefab_node` | Restore prefab node with asset UUID | 必填：`nodeUuid`, `assetUuid`

### 9.5 项目与资源基础工具（project，24）

- `project_run_project` | Run project in preview mode | 必填：-
- `project_build_project` | Build project | 必填：`platform`
- `project_get_project_info` | Get project information | 必填：-
- `project_get_project_settings` | Get project settings | 必填：-
- `project_refresh_assets` | Refresh asset database | 必填：-
- `project_import_asset` | Import asset file | 必填：`sourcePath`, `targetFolder`
- `project_get_asset_info` | Get asset information | 必填：`assetPath`
- `project_get_assets` | Query assets by type | 必填：-
- `project_get_build_settings` | Get build settings | 必填：-
- `project_open_build_panel` | Open build panel in editor | 必填：-
- `project_check_builder_status` | Check whether builder worker is ready | 必填：-
- `project_start_preview_server` | Start preview server | 必填：-
- `project_stop_preview_server` | Stop preview server | 必填：-
- `project_create_asset` | Create new asset file or folder | 必填：`url`
- `project_copy_asset` | Copy asset | 必填：`source`, `target`
- `project_move_asset` | Move asset | 必填：`source`, `target`
- `project_delete_asset` | Delete asset | 必填：`url`
- `project_save_asset` | Save asset content | 必填：`url`, `content`
- `project_reimport_asset` | Reimport asset | 必填：`url`
- `project_query_asset_path` | Get asset disk path | 必填：`url`
- `project_query_asset_uuid` | Get asset UUID from URL | 必填：`url`
- `project_query_asset_url` | Get asset URL from UUID | 必填：`uuid`
- `project_find_asset_by_name` | Find assets by name | 必填：`name`
- `project_get_asset_details` | Get detailed asset info including sub-assets | 必填：`assetPath`

### 9.6 调试工具（debug，10）

- `debug_get_console_logs` | Get editor console logs | 必填：-
- `debug_clear_console` | Clear editor console | 必填：-
- `debug_execute_script` | Execute JavaScript in scene context | 必填：`script`
- `debug_get_node_tree` | Get detailed node tree for debugging | 必填：-
- `debug_get_performance_stats` | Get performance statistics | 必填：-
- `debug_validate_scene` | Validate current scene for issues | 必填：-
- `debug_get_editor_info` | Get editor and environment information | 必填：-
- `debug_get_project_logs` | Get project logs from temp/logs/project.log | 必填：-
- `debug_get_log_file_info` | Get project log file information | 必填：-
- `debug_search_project_logs` | Search project logs for specific pattern | 必填：`pattern`

### 9.7 偏好设置工具（preferences，7）

- `preferences_open_preferences_settings` | Open preferences panel | 必填：-
- `preferences_query_preferences_config` | Query preferences configuration | 必填：`name`
- `preferences_set_preferences_config` | Set preferences configuration | 必填：`name`, `path`, `value`
- `preferences_get_all_preferences` | Get all preference categories | 必填：-
- `preferences_reset_preferences` | Reset preferences to default values | 必填：-
- `preferences_export_preferences` | Export preferences configuration | 必填：-
- `preferences_import_preferences` | Import preferences configuration from file | 必填：`importPath`

### 9.8 服务器工具（server，6）

- `server_query_server_ip_list` | Query server IP list | 必填：-
- `server_query_sorted_server_ip_list` | Get sorted server IP list | 必填：-
- `server_query_server_port` | Query editor server port | 必填：-
- `server_get_server_status` | Get comprehensive server status | 必填：-
- `server_check_server_connectivity` | Check network connectivity | 必填：-
- `server_get_network_interfaces` | Get available network interfaces | 必填：-

### 9.9 广播工具（broadcast，5）

- `broadcast_get_broadcast_log` | Get recent broadcast message log | 必填：-
- `broadcast_listen_broadcast` | Start listening for specific broadcast message | 必填：`messageType`
- `broadcast_stop_listening` | Stop listening for specific broadcast message | 必填：`messageType`
- `broadcast_clear_broadcast_log` | Clear broadcast log | 必填：-
- `broadcast_get_active_listeners` | Get active listener list | 必填：-

### 9.10 高级场景工具（sceneAdvanced，23）

- `sceneAdvanced_reset_node_property` | Reset node property to default value | 必填：`uuid`, `path`
- `sceneAdvanced_move_array_element` | Move array element position | 必填：`uuid`, `path`, `target`, `offset`
- `sceneAdvanced_remove_array_element` | Remove array element at index | 必填：`uuid`, `path`, `index`
- `sceneAdvanced_copy_node` | Copy node for later paste | 必填：`uuids`
- `sceneAdvanced_paste_node` | Paste copied nodes | 必填：`target`, `uuids`
- `sceneAdvanced_cut_node` | Cut node | 必填：`uuids`
- `sceneAdvanced_reset_node_transform` | Reset node transform | 必填：`uuid`
- `sceneAdvanced_reset_component` | Reset component to default values | 必填：`uuid`
- `sceneAdvanced_restore_prefab` | Restore prefab instance from asset | 必填：`nodeUuid`, `assetUuid`
- `sceneAdvanced_execute_component_method` | Execute method on component | 必填：`uuid`, `name`
- `sceneAdvanced_execute_scene_script` | Execute scene script method | 必填：`name`, `method`
- `sceneAdvanced_scene_snapshot` | Create scene snapshot | 必填：-
- `sceneAdvanced_scene_snapshot_abort` | Abort snapshot creation | 必填：-
- `sceneAdvanced_begin_undo_recording` | Begin undo recording | 必填：`nodeUuid`
- `sceneAdvanced_end_undo_recording` | End undo recording | 必填：`undoId`
- `sceneAdvanced_cancel_undo_recording` | Cancel undo recording | 必填：`undoId`
- `sceneAdvanced_soft_reload_scene` | Soft reload current scene | 必填：-
- `sceneAdvanced_query_scene_ready` | Check whether scene is ready | 必填：-
- `sceneAdvanced_query_scene_dirty` | Check whether scene has unsaved changes | 必填：-
- `sceneAdvanced_query_scene_classes` | Query registered classes | 必填：-
- `sceneAdvanced_query_scene_components` | Query available scene components | 必填：-
- `sceneAdvanced_query_component_has_script` | Check whether component has script | 必填：`className`
- `sceneAdvanced_query_nodes_by_asset_uuid` | Find nodes using asset UUID | 必填：`assetUuid`

### 9.11 场景视图工具（sceneView，20）

- `sceneView_change_gizmo_tool` | Change Gizmo tool | 必填：`name`
- `sceneView_query_gizmo_tool_name` | Query current Gizmo tool name | 必填：-
- `sceneView_change_gizmo_pivot` | Change transform pivot point | 必填：`name`
- `sceneView_query_gizmo_pivot` | Query current Gizmo pivot point | 必填：-
- `sceneView_query_gizmo_view_mode` | Query view/select mode | 必填：-
- `sceneView_change_gizmo_coordinate` | Change coordinate system | 必填：`type`
- `sceneView_query_gizmo_coordinate` | Query coordinate system | 必填：-
- `sceneView_change_view_mode_2d_3d` | Change 2D/3D mode | 必填：`is2D`
- `sceneView_query_view_mode_2d_3d` | Query current 2D/3D mode | 必填：-
- `sceneView_set_grid_visible` | Show or hide grid | 必填：`visible`
- `sceneView_query_grid_visible` | Query grid visibility | 必填：-
- `sceneView_set_icon_gizmo_3d` | Set IconGizmo 3D/2D mode | 必填：`is3D`
- `sceneView_query_icon_gizmo_3d` | Query IconGizmo mode | 必填：-
- `sceneView_set_icon_gizmo_size` | Set IconGizmo size | 必填：`size`
- `sceneView_query_icon_gizmo_size` | Query IconGizmo size | 必填：-
- `sceneView_focus_camera_on_nodes` | Focus scene camera on nodes | 必填：`uuids`
- `sceneView_align_camera_with_view` | Apply view to selected node | 必填：-
- `sceneView_align_view_with_node` | Apply selected node to view | 必填：-
- `sceneView_get_scene_view_status` | Get comprehensive scene view status | 必填：-
- `sceneView_reset_scene_view` | Reset scene view defaults | 必填：-

### 9.12 参考图工具（referenceImage，12）

- `referenceImage_add_reference_image` | Add reference image(s) to scene | 必填：`paths`
- `referenceImage_remove_reference_image` | Remove reference image(s) | 必填：-
- `referenceImage_switch_reference_image` | Switch to specific reference image | 必填：`path`
- `referenceImage_set_reference_image_data` | Set reference image transform/display property | 必填：`key`, `value`
- `referenceImage_query_reference_image_config` | Query reference image configuration | 必填：-
- `referenceImage_query_current_reference_image` | Query current reference image data | 必填：-
- `referenceImage_refresh_reference_image` | Refresh reference image display | 必填：-
- `referenceImage_set_reference_image_position` | Set reference image position | 必填：`x`, `y`
- `referenceImage_set_reference_image_scale` | Set reference image scale | 必填：`sx`, `sy`
- `referenceImage_set_reference_image_opacity` | Set reference image opacity | 必填：`opacity`
- `referenceImage_list_reference_images` | List available reference images | 必填：-
- `referenceImage_clear_all_reference_images` | Clear all reference images | 必填：-

### 9.13 高级资源工具（assetAdvanced，11）

- `assetAdvanced_save_asset_meta` | Save asset meta information | 必填：`urlOrUUID`, `content`
- `assetAdvanced_generate_available_url` | Generate an available URL from input URL | 必填：`url`
- `assetAdvanced_query_asset_db_ready` | Check whether asset DB is ready | 必填：-
- `assetAdvanced_open_asset_external` | Open asset with external program | 必填：`urlOrUUID`
- `assetAdvanced_batch_import_assets` | Batch import assets | 必填：`sourceDirectory`, `targetDirectory`
- `assetAdvanced_batch_delete_assets` | Batch delete assets | 必填：`urls`
- `assetAdvanced_validate_asset_references` | Validate asset references and broken links | 必填：-
- `assetAdvanced_get_asset_dependencies` | Get asset dependency tree | 必填：`urlOrUUID`
- `assetAdvanced_get_unused_assets` | Find unused assets | 必填：-
- `assetAdvanced_compress_textures` | Batch compress texture assets | 必填：-
- `assetAdvanced_export_asset_manifest` | Export asset manifest/inventory | 必填：-

### 9.14 验证辅助工具（validation，3）

- `validation_validate_json_params` | Validate and fix JSON parameters before sending to other tools | 必填：`jsonString`
- `validation_safe_string_value` | Create safe string value for JSON | 必填：`value`
- `validation_format_mcp_request` | Format complete MCP request with correct escaping | 必填：`toolName`, `arguments`
