# gst_avgbrightness
DeepStream GstAvgbrightness Plugin

该插件实现在GPU上对NV12数据的Y通道计算平均亮度，并把计算结果写入`user meta`数据空间以供用户获取

## 一、使用场景

  一些需要计算画面平均亮度的场景

  在`streammux`后面计算NV12数据的亮度，输入到`streammux`的数据必须是`NV12`或`JPEG`，我使用的是`NV12`，`JPEG`没有测试

## 二、环境

  DeepStreamSDK-6.4
  
  Ubuntu
  
  python3.10

## 三、编译 安装

  1、编译
  
  ```bash
  make
  ```

  生成`libgstavgbrightness.so`，`brightness.o`，`gstavgbrightness.o`3个文件

  2、安装 注册
  
  第一个`.so`文件就是我们需要的静态库文件，把它复制到`/opt/nvidia/deepstream/deepstream-6.4/lib/gst-plugins/`目录下，然后刷新`GStreamer `缓存：

  ```bash
  rm -rf ~/.cache/gstreamer-1.0/
  ```
  验证：

  ```bash
  gst-inspect-1.0 gstavgbrightness
  ```

  看到以下输出就表示插件安装注册成功：
  ```bash
  Factory Details:
    Rank  ...
    ...
  Plugin Details:
    Name  xjzhavgbrightness
    ...
  ```

## 四、使用

在python`pyds`中使用：

  1、创建插件、连接插件
  
  ```python
  ... 
  avg_brightness = Gst.ElementFactory.make("gstavgbrightness", "avg-brightness")
  ...
  pipeline.add(avg_brightness)
  ...
  # 把它link到streammux后面
  streammux.link(avg_brightness)
  avg_brightness.link(pgie1)
  ...
  ```

  2、获取输出

  ```python
  def osd_sink_pad_buffer_probe2(pad, info, u_data):
    global perf_data
    global fps_mutex

    gst_buffer = info.get_buffer()
    if not gst_buffer:
        print("Unable to get GstBuffer")
        return Gst.PadProbeReturn.OK

    batch_meta = pyds.gst_buffer_get_nvds_batch_meta(hash(gst_buffer))

    l_frame = batch_meta.frame_meta_list
    while l_frame:
        next_frame_data = l_frame.next
        frame_meta = pyds.NvDsFrameMeta.cast(l_frame.data)

        frame_number = frame_meta.frame_num
        num_rects = frame_meta.num_obj_meta
        source_id = frame_meta.source_id
        batch_id = frame_meta.pad_index

        # Initialize metadata holders
        brightness = -1

        # Process all user metas
        l_user = frame_meta.frame_user_meta_list
        while l_user:
            next_user_meta_data = l_user.next
            user_meta = pyds.NvDsUserMeta.cast(l_user.data)
            meta_type = user_meta.base_meta.meta_type

            if meta_type == 0xABC:  # Custom brightness meta
                brightness_ptr = ctypes.cast(pyds.get_ptr(user_meta.user_meta_data), ctypes.POINTER(ctypes.c_double))
                brightness = brightness_ptr.contents.value
                # print(f"SourceID:{source_id} FrameID:{frame_number} Brightness: {brightness:.1f}"))
            l_user = next_user_meta_data
        l_frame = next_frame_data
    return Gst.PadProbeReturn.OK
  ```
  
