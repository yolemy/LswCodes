# Construction Hazard Detection 项目笔记

> 学习参考：[yihong1120/Construction-Hazard-Detection](https://github.com/yihong1120/Construction-Hazard-Detection)  
> Demo：https://visionnaire-cda17.web.app/  
> 许可：AGPL-3.0（本目录仅为学习笔记，不含上游源码）

AI 工地安全监测系统：用 YOLO 做目标检测，用几何/聚类规则做违规判定，再通过直播、告警、违规章据与移动端把结果推给现场。

## 能发现什么

| 规则 | 思路 |
|------|------|
| 未戴安全帽 / 未穿反光背心 | 直接统计 `NO-Hardhat`、`NO-Safety Vest` 检测框 |
| 人员靠近机具/车辆 | 人员与 machinery/vehicle 做空间邻近判定（网格索引加速） |
| 进入路锥管制区 | 安全锥坐标 → HDBSCAN 聚类 → 凸包多边形 → 点在多边形内 |
| 人员进入电杆管制区 / 机具靠近电杆 | 电杆聚类后构造缓冲区并集，再做人/机是否落入 |

数据集共 **11 类**：Hardhat、Mask、NO-Hardhat、NO-Mask、NO-Safety Vest、Person、Safety Cone、Safety Vest、machinery、utility pole、vehicle。

## 运行时架构（推荐路径）

```
摄像头 / RTSP
    ↓
main.py 按库中 stream 配置拉起进程
    ↓
StreamCapture → YOLO Worker（共享内存多路复用 GPU）
    ↓
DangerDetector（规则后处理）
    ↓
┌─────────────┬──────────────┬────────────────┐
│ MediaMTX    │ Redis 协调键 │ PostgreSQL     │
│ 直播/回放   │ 会话/告警元数据 │ 工地/用户/违规 │
└─────────────┴──────────────┴────────────────┘
    ↓
FastAPI 微服务 + BFF + Flutter App + 多渠道通知
```

要点：

1. **视频与状态分离**：直播帧走 MediaMTX（HLS/WebRTC）；Redis 只放 auth、FCM token、overlay demand/ready、紧凑告警元数据，不存视频帧。
2. **多路相机共用推理**：`yolo_worker` 通过共享内存队列批处理，按模型尺寸配置每引擎相机数（如 nano 更密、x 更稀）。
3. **数据库模式优先**：工地、用户、stream、违规章据落 PostgreSQL；本地 JSON 配置仅适合短测。

## 仓库地图（按职责）

| 路径 | 职责 |
|------|------|
| `main.py` / `src/` | 正式运行时：拉流、推理、危险判定、推流与告警 |
| `examples/db_management/` | 用户、群组、工地、stream 配置 API |
| `examples/violation_records/` | 违规章据与截图 API |
| `examples/local_notification_server/` | FCM 等通知 |
| `examples/streaming_web/` | 播放地址、metadata、WebRTC ICE |
| `examples/bff/` | Web Session / CSRF / API 网关 |
| `examples/YOLO_*` | 训练、评估、增广、独立 YOLO API |
| `examples/mcp_server/` | MCP 工具入口（检测、记录、通知） |
| `deploy/` | Keycloak、Nginx、Redis、租户部署样例 |

Flutter 源码在独立仓库 [visionnaire-flutter](https://github.com/yihong1120/visionnaire-flutter)；本仓 `frontend/web/` 是部署产物。

## 危险判定核心（后处理）

YOLO 只给出框与类别；真正的「违规」在 `DangerDetector`：

1. **路锥管制区**：取锥中心点 → HDBSCAN（`min_samples=3`, `min_cluster_size=2`）→ 每簇 ≥3 点做凸包 → 人员中心点是否在多边形内。
2. **PPE**：统计无帽/无背心框数量，写入 `warning_no_hardhat` / `warning_no_safety_vest`。
3. **人机邻近**：人员与 machinery/vehicle 用网格索引缩小候选对，再做距离类判定。
4. **电杆区域**：电杆聚类后构造半径缓冲与外切线并集（Shapely），再做人/机是否落入。

规则可通过 stream 的 `detection_items` 开关按相机启用。

## 技术栈速记

- **检测**：Ultralytics YOLO（当前文档示例含 yolo26 / yolo11）、可选 TensorRT / SAHI
- **几何**：Shapely、SciPy、scikit-learn HDBSCAN
- **服务**：FastAPI、SQLAlchemy async + asyncpg、Redis、MediaMTX
- **身份**：OIDC（Keycloak 部署样例）
- **通知**：FCM、LINE、Messenger、Telegram、WeChat、HTTP broadcast
- **工程**：Python 3.14、`uv` 锁依赖、Docker Compose、按 extra 拆分镜像（streaming / violation / notification / yolo）

## 可借鉴的设计点

1. **检测与业务规则解耦**：模型只负责「看见什么」，规则层负责「算不算违规」，便于按工地开关能力。
2. **直播基础设施与业务状态分层**：视频走专用媒体服务，协调状态走 Redis，避免把帧塞进缓存。
3. **按相机进程 + 共享 YOLO Worker**：扩展路数时推理侧可批处理，配置侧用 DB 动态启停。
4. **可选依赖拆分**：API 运行时、YOLO、训练、GPU/TensorRT、社交通知各自 `extra`，镜像更瘦。
5. **多语言告警**：标签与通知覆盖繁中/简中/英/法/泰/越/印尼/日，适合跨境工地场景。

## 本地想跑通时的最小清单

1. `uv sync`（按需加 `yolo` / `streaming` 等 extra；CUDA 机再加 `yolo-gpu`）
2. 从 Hugging Face 拉 `models/pt/best_*.pt`
3. 起 PostgreSQL、Redis、MediaMTX，按 `.env.example` 填连接与 OIDC
4. 用 database 模式配置 stream，再跑 `main.py`

## 延伸阅读（上游）

- 英文 README / 繁中 README（仓库根目录）
- Roboflow 数据集、Hugging Face 模型仓（README badge）
- `docs/tensorrt-rebuild-guide.md`、硬件规格提案、Flutter OIDC/FCM 相关文档
