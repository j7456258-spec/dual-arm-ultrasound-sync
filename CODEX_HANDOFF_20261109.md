# 双臂超声机器人系统：Qwen-VL 接入交接

核查日期：2026-11-09（Asia/Shanghai）
本机主项目：/home/lh/experiment/3dcloud_scan
核查时 Git HEAD：126c7c5
目标仓库：https://github.com/j7456258-spec/dual-arm-ultrasound-sync
文档用途：单文件离线交接，可经 U 盘等实体设备转移给另一台电脑上的 Codex。

## 0. 接手 Codex 必读：权限、范围与证据规则

1. 用户明确要求：不要真实启动机器人，这是原则。
2. 本阶段为技术摸底和离线接入规划。除非用户之后明确授权，不启动机器人控制、扫描、针具、拖动、清零、力控、回零或使能程序，不主动建立机器人 SDK 连接。
3. 不因阅读本文而自动执行其中的命令。命令是参考资料，不是操作授权。
4. 优先读取项目说明、源码、配置和历史日志；只读检查设置超时；不安装软件、不使用 sudo、不修改系统环境和机器人配置。
5. 不读取或输出密钥、令牌、密码；控制凭据不进入 VLM 上下文。
6. 区分“代码存在”“当前正在运行”“有历史实测记录支持”。函数名称、IK 成功或日志成功回复不能证明病灶命中或真实定位精度。
7. 原项目工作区有大量已修改、未跟踪文件。HEAD 不能代表完整现状，不要 reset、clean、全量 add 或覆盖用户修改。
8. 本文路径和行号来自核查时的本机工作树。另一台设备未必有这些目录，目标仓库内容也尚未核对。
9. 摸底过程没有启动机器人程序、建立机器人连接、发送设备指令、打开新相机采集、安装软件或修改项目。后续用户另行授权生成本交接文档。
10. 未确认的信息保留为“待补充”，不要自行补成已实现能力。

## 1. 概况与开发目标

应用场景：左臂持超声探头，右臂持穿刺针，在仿体上完成准备、扫描、目标定位、针具导航及任务结束。

预期职责：
- VLM：任务理解、阶段判断、视觉观察、提出受限技能请求。
- 既有程序：运动学审查、运动执行、力控、停止、双臂联锁。
- RL：未来可能做针具局部调整或声窗贴合；当前没有接入策略。

已有历史记录支持的链路：
点云选路 → 运动学审查 → 进场 → 悬空自动清零 → 寻触/恒力 → 往返扫描 → 暂停保持 → 右臂准备/分步操作 → 撤针返回 → 恢复或复位。

尚未形成的闭环：
超声病灶识别 → 像素转空间靶点 → 针尖追踪 → 自动导航命中目标。

适合的第一步是离线回放视频与状态、验证阶段理解和技能请求，不需要机器人在线。主链直接使用 RealMan API2，不需要先迁移到 ROS 2。

## 2. 本机环境快照

| 项目 | 已核查结果 |
|---|---|
| 硬件 | NVIDIA Jetson AGX Orin Developer Kit |
| 架构/CPU | aarch64；12 核 Cortex-A78AE；读到最大频率约 2.20 GHz |
| 系统 | Ubuntu 22.04.5 LTS；内核 5.15.148-tegra |
| 内存 | MemTotal 64336272 kB，约 61.4 GiB；检查时可用约 56.4 GiB |
| 磁盘 | 根分区约 915 GiB，检查时可用约 808 GiB |
| GPU/驱动 | Orin (nvgpu)；nvidia-smi 查询驱动 540.4.0 |
| GPU 内存 | 总量和使用量查询为 N/A；共享系统内存，未做模型加载测试 |
| L4T | 36.4.7；nvidia-l4t-core 36.4.7-20250918154033 |
| JetPack | 精确套件版本待补充，不能只凭 L4T 版本填入 |
| CUDA | SDK 元数据 12.6.11；nvcc 12.6 / V12.6.68 |
| TensorRT | 系统包 10.3.0.30-1+cuda12.5 |
| ROS 2 | Humble 已安装；主扫描链不依赖 ROS |
| Qt/GStreamer | Qt 5.15.3；GStreamer 1.20.3 |

依据：/proc/device-tree/model、/etc/os-release、/etc/nv_tegra_release、/proc/meminfo、lscpu、df、nvidia-smi、dpkg-query、/usr/local/cuda/version.json、pkg-config。

实际控制/扫描解释器：
/home/lh/miniconda3/envs/robot/bin/python

robot 环境：Python 3.10.20；torch 2.7.0；torch built CUDA 12.6；NumPy 2.2.6；OpenCV 包 5.0.0.93；pyrealsense2 2.58.2.10647。
实际只读检查结果：cuda_available=True，device_count=1。

兼容问题：导入 PyTorch 出现按 NumPy 1.x 编译的扩展与 NumPy 2.2.6 不兼容、Failed to initialize NumPy 警告。CUDA 可用不代表图像/tensor/NumPy 转换正常。尚未修改依赖。

其他环境：
- 系统 Python：3.10.12。
- Conda lerobot：Python 3.10.18、torch 2.7.1、lerobot 0.3.4、NumPy 2.2.6。
- Conda teleop_opt：Python 3.10.20、NumPy 1.26.4，未发现 torch。
- /home/lh/pico/TAMEn/.venv：Python 3.10.12。

在检查的上述环境、项目、下载目录和常见模型缓存中，没有发现 Qwen-VL 权重或 transformers、accelerate、qwen-vl-utils、vLLM、SGLang 接入。系统 TensorRT 已安装不等于已有 VLM engine。未做 VLM 加载、吞吐、显存或延迟测试。其他电脑配置待补充。

核查时可见进程中没有发现监控器、扫描、针具桥接、Qwen/vLLM/Ollama、rm_driver 等相关进程。它们记录为“当前未运行”。未读取机器人控制器的当前上电、报警或姿态。ROS 图检查未完成：CLI 尝试写日志被文件系统拒绝，没有继续启动服务。

## 3. 项目地图与阅读顺序

优先阅读：
1. doublearm/REPRODUCE_WORKFLOW.md
2. 运行命令速查.md
3. configs/experiment.yaml
4. configs/contact_scan_continuous.yaml
5. configs/needle_puncture.yaml
6. docs/DEVELOPER_FRAMES_AND_MODULES_20261023.md

以下相对路径均相对于 /home/lh/experiment/3dcloud_scan。

| 模块 | 职责 |
|---|---|
| run.py；src/phantom_scan/cli/main.py | CLI 分发 |
| src/phantom_scan/cli/probe_wrist_plan.py | D435、Freeze、人工选路 |
| interactive_plan.py；local_surface.py | 截面路径、局部二次面、法向平滑 |
| surface_reconstruction.py；surface_backend | PCL/MLS 曲面拟合与投影 |
| motion_planning.py；joint_transfer.py | IK、限位、连续性、进返场审查 |
| motion_execution.py | 反馈读取、MoveJ/MoveL、停止基础逻辑 |
| contact_scan.py；contact_stream.py | 清零、寻触、恒力、往返、暂停和收尾 |
| scan_control.py | 扫描会话动作、暂停/恢复/复位、针具占用 |
| needle_control.py | 右臂准备、调角、分步进退针、撤回 |
| tools/needle_control_bridge.py | JSON 行命令、回复与状态桥接 |
| arm_api_new/realman_arm_api_api2.py | 项目本地 RealMan API2 适配器 |
| arm_api_new/Robotic_Arm | 本地厂商 Python/ctypes/原生 SDK |
| arm_api_new/telemetry.py | 有界队列、后台 UDP 遥测 |
| joint_planning.py；planning_backend | MoveIt/OMPL 离线预览，不是主执行避碰链 |
| tools/moment_level_demo.py | 独立力矩调平实验 |

关联项目：
- /home/lh/experiment/ultrasound_monitor：C++17/Qt5/GStreamer 界面，显示视频、力和状态；含清零、暂停、复位和针具操作通道。
- /home/lh/experiment/handeye_calibration：独立手眼采集、求解。
- /home/lh/experiment/move_control：独立网页示教、钧舵夹爪操作。
- doublearm/server.py：双臂状态、命名关节位姿和滑轨网页；默认模拟，实机需 --connect。
- /home/lh/robot_api：设备接口来源之一；不同项目另有本地适配器，改一份不会自动影响全部入口。
- /home/lh/robot：ROS 2 工作空间，含 rm_driver、rm_bringup、rm_control、rm_ros_interfaces、rm_65_config、force_position_control、get_arm_state、lh_lift_bridge、head_ros2、yolov8_ros2 等。
- /home/lh/pico/TAMEn：其他遥操作/相机工程，含 realman_teleop、orbbec_camera 等。
- /home/lh/pico/TAMEn/dual_arm_ros2：独立双臂 ROS 2/MoveIt 工程。

文档冲突：部分说明仍称监控器“只读”，但源码有操作通道；旧坐标文档记录10月22日变换，当前配置使用10月28日结果；配置注释和 UI 提示有旧值。以当前数值、执行代码和对应会话日志为准。

## 4. 双臂、接口与坐标系

左臂：RM65，6DOF，169.254.128.18:8080，工作系 Base，工具 us_scan。
右臂：RM65，6DOF，169.254.128.19:8080，工作系 Base，工具 tar；Z轴沿针杆。
通信：RealmanArmClient → RealMan API2 Python/ctypes → libapi_c.so → 控制器 IP/端口。
本次没有初始化机械臂 SDK，未重新读控制器软件版本。

控制：MoveJ 关节运动、MoveL 笛卡尔运动、原生工具 Z 力控流。连续扫描沿路径前馈，控制器做法向力补偿。Python 循环不是硬实时力伺服。

坐标/单位：
- T_A_B：把 B 系坐标转换到 A 系，齐次列向量。
- TCP：[x,y,z,rx,ry,rz]，位置米、Euler 弧度；R=Rz·Ry·Rx。
- 关节角：度。
- Fx/Fy/Fz：N；Mx/My/Mz：N·m。
- tool_zero_force_data：工具系表达的清零六维力；力矩参考原点仍需专项确认。
- reaction_n=feedback_sign×Fz，不是三轴合力。
- 左右都叫 Base，但原点和方向不同。
- camera_color_optical_frame：D435彩色光学系，X右/Y下/Z前。
- us_scan YZ：软件名义超声成像平面；tar Z：针轴。名义约定不代表已完成真实声学平面标定。

已有状态：get_state/rm_get_current_arm_state、get_force_data、工具/工作系、报警、电源、模式、trajectory_type、扫描phase/进度/方向/周期/偏差/力稳定窗口、针具prepared/depth_mm/fault/stop_confirmed。

## 5. 标定与精度：不要丢失这些限制

### 5.1 D435

内参、深度尺度、RGB-D 外参/对齐由 RealSense SDK 获取。主点云反投影使用 NumPy 针孔模型。没有在主扫描中重新标定 RGB-D 外参，本次未打开相机读取当前值。

### 5.2 手眼

文件：data/handeye_trials/history_20261013_28p0/best_173444_us_scan.json。
变换：T_us_scan_camera，相机光学系 →探头 TCP。
标记：valid=false，independent_validation=false。
方法：历史 Tsai 候选，使用实测28.0 mm棋盘尺度重算并换工具参考系。
拟合平移 RMS约6.786 mm，旋转 RMS约0.314°。这些不是独立物理定位精度。
复用的物理前提是相机/支架和相关工具定义没有变化，文件名或工具名一致不足以证明有效。

### 5.3 双臂配准

文件：configs/needle_puncture.yaml。
字段：calibration.T_right_base_left_base。
方向：左 Base →右 Base。
来源：docs/validation/DUAL_ARM_COMMON_POINTS_20261028.json。
三共同点固定尺度 SVD 刚体拟合；RMS1.785 mm、最大2.309 mm。
独立验证点误差11.099 mm，状态 demo-only。不能将拟合误差当成实际系统误差。
旧10月22日文档里的变换不是当前版本。

### 5.4 工具

us_scan 和 tar 由用户在控制器侧建立，保存快照存在，但本次未读取当前值。tar 已包含针尖工具偏移，不重复叠加针长。不能从现存源码确认用户实际采用几/六点法还是手工输入。

### 5.5 尚缺映射

点云链：
p_left_base=T_left_base_probe×T_probe_camera×p_camera。
p_right_base=T_right_base_left_base×p_left_base。

超声尚缺：像素 →声学毫米 →探头 →左Base →右Base。
D435内参不能代替超声像素尺度，ROI裁剪不是标定，VLM返回像素点不能直接变成可靠进针靶点。

## 6. 可复用技能与完成/失败反馈

| 功能 | 入口 | 输入与反馈 |
|---|---|---|
| 点云选路 | wrist-plan | 相机/手眼/ROI/锚点/间隙 →计划与表面文件；需要人工交互，失败拒绝保存 |
| 运动学审查 | review_motion/check_task/review_contact | 计划、当前状态、限位 →审查结果或异常；不是碰撞认证 |
| 进场 | ContactExecutor.execute/_move | 审查轨迹、速度 →到位或超时/报警/偏离失败，可停止 |
| 自动清零 | _clear_force_at_hover | 悬空进场位 →force_zero_before/after；SDK错误中止 |
| 寻触/恒力 | _native_hold及工具Z力流 | 力、符号、距离/速度 →接触/稳定窗口；超力/超程/超时失败 |
| 往返扫描 | contact_stream.execute_stream_scan | 路径/力/速度 →单程/周期事件；重复到复位或中断 |
| 暂停保持 | _hold_scan_pause；pause | accepted后需等scan_paused；保持接触，不等于硬停止 |
| 恢复/复位 | ScanResetServer.poll | 有效会话、无针具占用 →接受/拒绝和后续阶段 |
| 针具准备 | NeedleController.prepare(context,offset_mm) | 暂停会话、探头局部偏移/保存朝向 →到位回复或失败 |
| 调角 | adjust(degrees,mode) | 仅准备点状态 →到位或失败 |
| 进退针 | step(millimetres) | 带符号步长 →完成/失败、depth_mm |
| 撤针返回 | park() | 退针、撤销调角、回记录起点、释放占用 |
| 内部靶点几何 | needle-plan/build_needle_plan | 人工靶点或深度角度 →离线计划/预览，不直接执行 |
| 力矩调平 | tools/moment_level_demo.py | 独立实验；减小Mx/My不代表声窗完整贴合 |
| 示教/夹爪 | move_control/server.py | 轴/方向/参考系/夹爪参数；独立程序，本次未专项复核实测 |

针具桥接用 stdin/stdout JSON行；有 request_id、reply.ok、message、state。运动完成检查包含位姿误差和轨迹空闲，不只是命令发送成功。stop可清掉待执行操作并请求停止。

depth_mm是机器人针轴累计位移，不是视觉确认的组织内深度。

双臂协调：左臂暂停保持 →右臂申请占用 →操作 →撤针返回释放 →左臂允许恢复/复位；还监测左探头锚点漂移。这是阶段联锁，不是两臂同步轨迹控制。

主审查明确 collision_checked=false。独立MoveIt/OMPL预览不等于主执行链已经具备双臂实时避碰。

## 7. 配置快照与停止机制

以下只报告已有设置，不是建议扩大限制。

configs/contact_scan_continuous.yaml：
- continuous_scan、scan_return_pass、repeat_scan开启。
- 名义扫描速度3 mm/s，加速度1 mm/s²；坡度/姿态/力误差/滞后会降速。
- 默认目标力5 N，文档复现命令覆盖为3 N，实际会话必须读start配置。
- XYZ三轴合力停止阈值20 N。
- 轻触阈值2 N；命令符号默认0，真实执行需明确±1。
- 寻触距离30 mm、速度上限4 mm/s。
- 轻触后附加压入10 mm。
- 当前局部补偿：+X 5 mm，−Y 12 mm，Rx 0°；注释有旧值。
- 接触扫描切向容差3 mm、姿态容差6°、接触丢失1.2 s、轮询10 Hz。

右臂：单次调角≤5°，单次进退针≤10 mm，累计推进上限配置150 mm；UI旧提示写20 mm。左探头漂移容差2 mm/1°。右臂速度代码值为10（SDK百分比）。

motion_execution.py仍有审查后起点漂移检查：0.5 mm、0.2°、关节0.1°。

已有软件慢停、暂停、清轨迹和失败直接停止分支；Ctrl+C/异常有中断路径。软件停止不等于硬件急停认证，硬件急停链本次未核查。不要通过修改限制消除未知接触或精度问题。

## 8. 视频、感知、同步与样例

设备记录：迈瑞DC-8pro；探头完整型号待补充。
链路：HDMI → Lattice/Word Vision USB采集卡 →V4L2 →GStreamer。
请求：YUY2、640×360、60fps、MMAP。
预览ROI：160,50,256,260；转RGB后appsink/QImage显示。
录像在ROI前分流，保存完整画面，H.264/MP4；默认x264enc，有nvv4l2h264enc分支，但不能认定该分支已实测。

本次读取代表性历史视频元数据：640×360、约60.0305fps、17385帧。它不是当前实时值。检查时主机可见三组RealSense435视频节点，未发现Word Vision采集卡节点；没有新启动采集。

D435：配置序列号420122071063，640×480/30fps，采样60帧；pyrealsense2、Z16深度/BGR8彩色、深度对齐彩色；保留timestamp_ms。本次未核对节点与指定序列号对应关系。

现有几何感知：点云反投影、体素/离群过滤、PCA/MLS、局部二次曲面、截面一维求根、弧长法向平滑、区域质量和网格IK初筛。没有找到超声病灶分割、针尖检测/追踪、经验证可见性或贴合质量模型。

通用YOLO位于另一ROS工作空间，默认权重：
/home/lh/robot/src/yolov8_ros2/yolov8_ros2/best.pt。
输入彩色图/对齐深度/CameraInfo，输出/yolov8/detections、/yolov8/annotated_image。启动配置含饮料类别别名，不能当超声模型。

同步：扫描JSONL有wall/monotonic纳秒时间；视频帧表有record_input_index、pts_ns、receive_monotonic_ns、receive_wall_ns；run_id/同名文件关联；遥测UDP127.0.0.1:9876。
没有硬件同步证据，HDMI/超声内部延迟未测，session中hdmi_delay_ms=null。主机接收时间不是超声曝光时间。

实时VLM图像出口尚缺：当前appsink主要供C++进程内显示，未发现现成对外帧服务。离线可直接用MP4和帧表。

代表数据：
/home/lh/experiment_data/ultrasound_monitor/contact_20261029_102050_672583.mp4
/home/lh/experiment_data/ultrasound_monitor/contact_20261029_102050_672583.frames.csv
/home/lh/experiment_data/ultrasound_monitor/contact_20261029_102050_672583.session.json
/home/lh/experiment_data/ultrasound_monitor/contact_20261029_095538_547839.mp4

docs/validation/ULTRASOUND_DATASET_REVIEW_20261021.md
docs/validation/ultrasound_review_20261021/152928_52s_roi.png

已有复核明确没有病灶真值；低回声、暗区不能直接认定病灶或脱耦。

## 9. 历史验证范围

扫描日志：logs/contact_20261029_102050_672583.jsonl。
包含1994条feedback、清零、接触、2个完成扫描周期、暂停、复位、最终complete。
关键行：161清零后、2252暂停锚点、8696最终完成。

针具日志：logs/needle_manual_20261029_091200_301990.jsonl。
54条命令、54条成功回复；prepare7、angle3、step34、park7、remember_heading3。
最大记录轴向位移约60.09mm。证明该次控制流程有完成记录，不证明针具命中病灶或精确组织深度。

已有计划：data/plans/interactive_scan_plan_20261028_210123.json，108个waypoints。

本次未重跑测试或实机复现。

## 10. 建议的VLM接入边界（尚未实现）

独立任务编排层：聚合图像/状态 →调用VLM →检查结构化请求 →既有技能 →收集结果。

优先复用两个边界：
1. scan_control.py：pause/resume/reset/snapshot及占用联锁。
2. tools/needle_control_bridge.py：JSON命令/回复和右臂状态。

针具接口是QProcess标准输入输出桥接，不是现成HTTP或ROS Action。点云规划和扫描启动仍是交互CLI；需要统一任务生命周期，不能靠自动输入确认字符串完成接入。

输入可含：超声帧/ROI、帧龄、阶段、接触力/目标力/稳定窗口、路径进度、暂停状态、针具占用/准备/累计位移/故障、计划ID、标定版本和有效性。控制凭据保留在执行层。

需要补反馈语义：
- accepted不等于动作完成。
- 暂停需确认scan_paused及保持状态。
- 为技能统一started/running/succeeded/failed/cancelled和超时。
- 针具成功只表示机器人动作完成，不表示针尖可见/命中。
- locate_target、verify_probe_coupling、navigate_to_lesion尚无可靠闭环。

离线输入示例（不是当前实机状态）：

~~~json
{
  "schema_version": 1,
  "mode": "offline",
  "task": "扫描仿体并提示疑似目标",
  "observation": {
    "frame": "replay/frame_001200.png",
    "roi_xywh": [160, 50, 256, 260],
    "stage": "scan",
    "left": {"reaction_n": 3.0, "scan_paused": false},
    "right": {"prepared": false, "needle_locked": false},
    "calibration": {"handeye_valid": false, "ultrasound_pixel_mapping_valid": false}
  },
  "allowed_skills": ["observe", "pause_scan"]
}
~~~

离线输出示例（不是现有控制通道可执行报文）：

~~~json
{
  "schema_version": 1,
  "stage_estimate": "scan",
  "finding": "存在需要人工复核的低回声区域",
  "lesion_confirmed": null,
  "requested_skill": "pause_scan",
  "parameters": {},
  "execute": false
}
~~~

## 11. 第一轮离线工作与用户待补信息

可离线推进：
- 选定准确Qwen-VL权重、量化/推理框架；验证本机延迟/内存，不能凭参数量猜测性能。
- 验证图像/tensor兼容问题，不先更改控制环境。
- 建立MP4+帧表+JSONL回放包。
- 建立空执行技能适配器，验证阶段判断、无依据目标判断率、请求合法性和时延。
- 准备人工确认的正常扫描、疑似目标、目标/针尖不可见、贴合不良样例。

必须由用户补充或决定，按优先级：
P0：仿体厂家结构图、病灶位置/尺寸、人工确认目标帧。
P0：超声探头完整型号、成像深度/缩放/显示模式及采集中是否变化。
P0：相机支架、探头、针具、TCP定义自最后记录后是否改变。
P1：VLM自主权限和每项技能人工确认边界。
P1：任务精度/成功标准；现有约11.1mm独立配准误差是否满足演示。
P1：本机或远端部署取向、指定模型；其他推理电脑配置未知。
P2：RL要优化贴合、可见性还是局部路径，以及评价真值。

## 12. 启动命令索引与参考（只展示，不执行）

现行参考：doublearm/REPRODUCE_WORKFLOW.md。
第23行监控录像；第36行点云选路；第52行审查；第65行扫描；第80行暂停/针具。

以下命令会打开设备或连接控制器，有些会运动。本文不授权执行。设备路径和旧计划只用于复现记录，当前采集卡节点未发现，手眼文件未独立验证。

监控：
~~~bash
/home/lh/experiment/ultrasound_monitor/build/ultrasound_monitor \
  --device /dev/v4l/by-id/usb-Lattice_Word_Vision_USB_Video-video-index0 \
  --width 640 --height 360 --fps 60 \
  --roi 160,50,256,260 --record --encoder x264enc
~~~

点云规划：
~~~bash
cd /home/lh/experiment/3dcloud_scan
/home/lh/miniconda3/envs/robot/bin/python run.py wrist-plan \
  --handeye-result data/handeye_trials/history_20261013_28p0/best_173444_us_scan.json \
  --allow-unvalidated-handeye --probe-angle-deg 0 --path-mode section \
  --normal-offset-m 0 --clearance-m 0.03
~~~

连接审查（不运动，但会连设备并生成日志，本次未执行）：
~~~bash
/home/lh/miniconda3/envs/robot/bin/python run.py hover-check \
  /home/lh/experiment/3dcloud_scan/data/plans/interactive_scan_plan_20261028_210123.json \
  --points all --run-through --return-to-start --clearance-m 0.03 --connect
~~~

真实扫描参考（本次未执行）：
~~~bash
/home/lh/miniconda3/envs/robot/bin/python run.py contact-scan \
  /home/lh/experiment/3dcloud_scan/data/plans/interactive_scan_plan_20261028_210123.json \
  --contact-config configs/contact_scan_continuous.yaml \
  --stage scan --force-n 3 --command-sign 1 --clearance-m 0.03 \
  --execute --telemetry
~~~

## 13. 证据索引

相对主项目：
- src/phantom_scan/cli/main.py:7，CLI列表。
- configs/experiment.yaml:8，相机；:54，机器人。
- configs/contact_scan_continuous.yaml:3，接触参数。
- configs/needle_puncture.yaml:19，当前配准；:6，针具规划。
- src/phantom_scan/contact_scan.py:623，反馈、保护和会话；:749，自动清零。
- src/phantom_scan/contact_stream.py:92，连续扫描。
- src/phantom_scan/scan_control.py:81，会话动作、占用联锁。
- src/phantom_scan/needle_control.py:129，左锚点检查；:177，到位/超时；:246，准备；:368，调角/步进/返回；:416，状态。
- tools/needle_control_bridge.py:97，命令分发与回复。
- src/phantom_scan/motion_execution.py:10，起点漂移；:15，反馈；:40，停止。
- src/phantom_scan/motion_planning.py:1，非碰撞检查声明。
- src/phantom_scan/joint_planning.py:1，离线预览声明。
- arm_api_new/realman_arm_api_api2.py:511，连接；:1429，Z力控。
- arm_api_new/telemetry.py:74，时间戳。
- docs/DEVELOPER_FRAMES_AND_MODULES_20261023.md:7，坐标；:64，手眼；:140，缺失像素映射；:202，超声。
- docs/validation/ULTRASOUND_DATASET_REVIEW_20261021.md:30，图像真值限制；:49，延迟。

其他项目：
- /home/lh/experiment/ultrasound_monitor/src/main.cpp:23，工作进程默认配置。
- /home/lh/experiment/ultrasound_monitor/src/video_capture.cpp:33，采集管线；:60，录像分流；:112，帧/实际参数。
- /home/lh/experiment/ultrasound_monitor/src/needle_control_process.cpp:24，QProcess接口。
- /home/lh/robot/src/yolov8_ros2/launch/yolov8_launch.py:11，通用模型及输入。
- /home/lh/experiment_data/ultrasound_monitor/contact_20261029_102050_672583.session.json:3，时间关联和未测HDMI延迟。

行号对应摸底时工作树，后续修改需重新核对。

## 14. 能力分级与交付状态

A. 有证据支持、可作为接入基础：CUDA/PyTorch基础、历史视频/帧表/状态、扫描清零寻触恒力往返暂停复位、针具准备调角步进撤回、会话联锁和反馈协议。这里的“可用基础”不授权实机启动。

B. 代码已有仍需专项验证：不同构型进返场、真实坐标精度、针轴入面效果、跨会话复用、力矩调平对贴合的效果、硬件编码和异常恢复、网页夹爪与多进程使用边界。

病灶识别、针尖追踪、超声像素映射、RL局部调整属于尚缺实现/证据，不属于“已跑通”。

GitHub状态：曾用gh api尝试访问目标仓库，退出码4，未配置可用GitHub登录。未上传或创建远端文件，仓库内容未核对。

后续若上传，只提交获授权的交接文档。不要附带整个脏工作区、日志、视频、权重或凭据。

转移说明：本文件是自包含UTF-8文本，可直接复制到实体存储设备。它不包含源码、视频、权重或标定文件本体；另一台Codex可据此规划，但执行或精确代码核对仍需要获授权的实际文件。

