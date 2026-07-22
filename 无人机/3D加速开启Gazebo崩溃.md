
## 开启 3D 加速时

正常启动：

```
cd ~/PX4-Autopilot
make px4_sitl gz_x500
```

这时 Gazebo 会尽量使用 VMware 提供的虚拟 GPU。

适合：

- 主机显卡性能较好
- VMware Tools 已正确安装
- Gazebo 不闪退、不黑屏
- 希望画面更流畅

## 关闭 3D 加速时

建议配合软件渲染：

```
cd ~/PX4-Autopilot
LIBGL_ALWAYS_SOFTWARE=1 make px4_sitl gz_x500
```

这里的：

```
LIBGL_ALWAYS_SOFTWARE=1
```

表示强制 OpenGL 使用 CPU 软件渲染，绕过 VMware 虚拟显卡。

适合：

- Gazebo 图形窗口崩溃
- 出现 `ruby3.0 意外停止`- 

