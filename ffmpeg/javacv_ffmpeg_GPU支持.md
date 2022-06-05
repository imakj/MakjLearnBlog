# 环境

* 系统信息

  ```
  # uname -a
  Linux 10-149-229-7 3.10.0-957.21.3.el7.x86_64 #1 SMP Tue Jun 18 16:35:19 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
  # cat /etc/redhat-release
  CentOS Linux release 7.6.1810 (Core)
  #  cat /proc/version
  Linux version 3.10.0-957.21.3.el7.x86_64 (mockbuild@kbuilder.bsys.centos.org) (gcc version 4.8.5 20150623 (Red Hat 4.8.5-36) (GCC) ) #1 SMP Tue Jun 18 16:35:19 UTC 2019
  ```

* GPU信息

  ```
  # lspci
  00:0c.0 3D controller: NVIDIA Corporation TU104GL [Tesla T4] (rev a1)
  ```

# 准备

* 查看本机nouveau，如果有则禁用nouveau驱动

  ```
  # lsmod | grep nouveau
  ```

* 驱动文件和cuda文件,注意显卡驱动版本和cuda的对应版本，也可以在安装cuda直接安装显卡驱动

  T4显卡 : NVIDIA-Linux-x86_64-450.80.02.run

  cuda ：cuda_11.0.3_450.51.06_linux.run 

  ffmpege : ffmpeg-4.3.tar
  
  nasm : nasm-2.15.05.tar.gz
  
  yasm : yasm-1.3.0.tar.gz(ffmpege 依赖)
  
  x264 : x264.zip
  
  nvcodeheaders : nv-codec-headers.zip (注意和cuda对应的版本)
  
* 安装epel-release并且升级

  ```
  # sudo yum install epel-release -y
  # sudo yum update -y
  ```

  


# CUDA安装

1. 安装驱动和cuda

   ```
   # sudo chmod +x cuda_11.0.3_450.51.06_linux.run
   # sudo ./cuda_11.0.3_450.51.06_linux.run
   lqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqk
   x  End User License Agreement                                                  x
   x  --------------------------                                                  x
   x                                                                              x
   x  NVIDIA Software License Agreement and CUDA Supplement to                    x
   x  Software License Agreement.                                                 x
   x                                                                              x
   x                                                                              x
   x  Preface                                                                     x
   x  -------                                                                     x
   x                                                                              x
   x  The Software License Agreement in Chapter 1 and the Supplement              x
   x  in Chapter 2 contain license terms and conditions that govern               x
   x  the use of NVIDIA software. By accepting this agreement, you                x
   x  agree to comply with all the terms and conditions applicable                x
   x  to the product(s) included herein.                                          x
   x                                                                              x
   x                                                                              x
   x  NVIDIA Driver                                                               x
   x                                                                              x
   x                                                                              x
   xqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqx
   x Do you accept the above EULA? (accept/decline/quit):                         x
   x accept                                                                       x
   mqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqj
   
   #全选，附带驱动一起安装 然后选择install
   lqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqk
   x CUDA Installer                                                               x
   x - [X] Driver                                                                 x
   x      [X] 450.51.06                                                           x
   x + [X] CUDA Toolkit 11.0                                                      x
   x   [X] CUDA Samples 11.0                                                      x
   x   [X] CUDA Demo Suite 11.0                                                   x
   x   [X] CUDA Documentation 11.0                                                x
   x   Options                                                                    x
   x   Install                                                                    x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x Up/Down: Move | Left/Right: Expand | 'Enter': Select | 'A': Advanced options x
   mqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqj
   
   #3 选择Yes
    A symlink already exists at /usr/local/cuda. Update to this installation?    x
   x Yes                                                                          x
   x No                                                                           x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x                                                                              x
   x Up/Down: Move | 'Enter': Select                                              x
   mqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqj
   
   #最后 安装完成
   ===========
   = Summary =
   ===========
   
   Driver:   Installed
   Toolkit:  Installed in /usr/local/cuda-11.0/
   Samples:  Installed in /root/, but missing recommended libraries
   
   Please make sure that
    -   PATH includes /usr/local/cuda-11.0/bin
    -   LD_LIBRARY_PATH includes /usr/local/cuda-11.0/lib64, or, add /usr/local/cuda-11.0/lib64 to /etc/ld.so.conf and run ldconfig as root
   
   To uninstall the CUDA Toolkit, run cuda-uninstaller in /usr/local/cuda-11.0/bin
   To uninstall the NVIDIA Driver, run nvidia-uninstall
   Logfile is /var/log/cuda-installer.log
   ```

   

2. 配置环境变量并且使其生效

   ```
   # vim /etc/profile
   
   #最后一行添加
   export LD_LIBRARY_PATH=/usr/local/cuda-11.0/lib64
   export PATH=$PATH:/usr/local/cuda-11.0/bin
   #保存推出 使其生效
   # source /etc/profile
   ```

3. 验证

   ```
   # nvidia-smi
   Tue Feb  9 23:14:39 2021       
   +-----------------------------------------------------------------------------+
   | NVIDIA-SMI 450.51.06    Driver Version: 450.51.06    CUDA Version: 11.0     |
   |-------------------------------+----------------------+----------------------+
   | GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
   | Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
   |                               |                      |               MIG M. |
   |===============================+======================+======================|
   |   0  Tesla T4            Off  | 00000000:00:0B.0 Off |                    0 |
   | N/A   56C    P0    19W /  70W |      0MiB / 15109MiB |      0%      Default |
   |                               |                      |                  N/A |
   +-------------------------------+----------------------+----------------------+
                                                                                  
   +-----------------------------------------------------------------------------+
   | Processes:                                                                  |
   |  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
   |        ID   ID                                                   Usage      |
   |=============================================================================|
   |  No running processes found                                                 |
   +-----------------------------------------------------------------------------+
   # nvcc -V
   nvcc: NVIDIA (R) Cuda compiler driver
   Copyright (c) 2005-2020 NVIDIA Corporation
   Built on Wed_Jul_22_19:09:09_PDT_2020
   Cuda compilation tools, release 11.0, V11.0.221
   Build cuda_11.0_bu.TC445_37.28845127_0
   
   ```

# ffmpeg 安装

1. 安装nasm 

   ```
   # cd /usr/local/src
   # tar xvf nasm-2.15.05.tar.gz
   # cd nasm-2.15.05/
   # ./autogen.sh
   # ./configure --prefix=/usr/local/nasm
   # make -j
   # make install
   ```

3. 配置环境变量并且使其生效

   ```
   # vim /etc/profile
   
   #最后一行添加
   export PATH=$PATH:/usr/local/nasm/bin
   #保存推出 使其生效
   # source /etc/profile
   ```

4. 安装yasm

   ```
   # cd /usr/local/src
   # tar zxvf yasm-1.3.0.tar.gz
   # cd yasm-1.3.0/
   # ./configure --prefix=/usr/local/yasm
   # make -j
   # make install
   ```

5. 配置环境变量并且使其生效

   ```
   # vim /etc/profile
   
   #最后一行添加
   export PATH=$PATH:/usr/local/yasm/bin
   #保存推出 使其生效
   # source /etc/profile
   ```

6. 安装x264

   ```
   # cd /usr/local/src
   # unzip x264.zip
   # cd x264/
   # ./configure --prefix=/usr/local/x264 --enable-shared --enable-static 
   # make -j
   # make install
   ```

7. 配置环境变量并且使其生效

   ```
   # vim /etc/profile
   
   #最后一行添加
   export PATH=$PATH:/usr/local/x264/bin
   #保存推出 使其生效
   # source /etc/profile
   ```

8. 修改文件/etc/ld.so.conf

   ```
   # vim /etc/ld.so.conf
   include ld.so.conf.d/*.conf
   /usr/local/x264/lib
   
   #执行生效
   # ldconfig
   ```

9. 安装nv-codec-headers

   ```
   # yum install alsa-lib-devel #这个需要验证下 需不需要安装
   # cd /usr/local/src
   # unzip nv-codec-headers.zip
   # cd nv-codec-headers/
   # make -j
   sed 's#@@PREFIX@@#/usr/local#' ffnvcodec.pc.in > ffnvcodec.pc
   # make install
   sed 's#@@PREFIX@@#/usr/local#' ffnvcodec.pc.in > ffnvcodec.pc
   install -m 0755 -d '/usr/local/include/ffnvcodec'
   install -m 0644 include/ffnvcodec/*.h '/usr/local/include/ffnvcodec'
   install -m 0755 -d '/usr/local/lib/pkgconfig'
   install -m 0644 ffnvcodec.pc '/usr/local/lib/pkgconfig'
   
   ```

10. 配置环境变量并且使其生效

    ```
    # vim /etc/profile
    
    #最后一行添加
    export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig
    #保存推出 使其生效
    # source /etc/profile
    ```

11. 安装ffmpeg(也可以添加libfdk_aac、libfreetype 、libmp3lame 、libopus 、libvorbis 、libvpx 、libx265 d等其他模块)

    ```
    # cd /usr/local/src
    # tar zxvf ffmpeg-4.3.tar.gz
    # cd ffmpeg-4.3/
    # ./configure \
        --prefix=/usr/local/ffmpeg \
        --extra-cflags='-I/usr/local/cuda-11.0/include -fPIC ' \
        --extra-ldflags='-L/usr/local/cuda-11.0/lib64 -ldl ' \
        --extra-cflags='-I/usr/local/x264/include' \
        --extra-ldflags='-L/usr/local/x264/lib' \
        --pkg-config-flags=--static \
        --enable-shared \
        --enable-gpl \
        --enable-libx264 \
        --extra-libs=-ldl \
        --enable-cuvid \
        --enable-nvenc \
        --enable-nonfree \
        --nvcc=/usr/local/cuda-11.0/bin/nvcc
    # make -j
    # make install
    ```

12. 修改文件/etc/ld.so.conf

    ```
    # vim /etc/ld.so.conf
    include ld.so.conf.d/*.conf
    /usr/local/x264/lib
    /usr/local/ffmpeg/lib
    
    #执行生效
    # ldconfig
    ```

12. 配置环境变量

    ```
    # vim /etc/profile
    
    ……
    export PATH=$PATH:/usr/local/ffmpeg/bin
    #保存推出 使其生效
    # source /etc/profile
    ```

15. 查看版本

    ```
    # ffmpeg -version
    ffmpeg version 4.3 Copyright (c) 2000-2020 the FFmpeg developers
    built with gcc 4.8.5 (GCC) 20150623 (Red Hat 4.8.5-44)
    configuration: --enable-shared --prefix=/usr/local/ffmpeg
    libavutil      56. 51.100 / 56. 51.100
    libavcodec     58. 91.100 / 58. 91.100
    libavformat    58. 45.100 / 58. 45.100
    libavdevice    58. 10.100 / 58. 10.100
    libavfilter     7. 85.100 /  7. 85.100
    libswscale      5.  7.100 /  5.  7.100
    libswresample   3.  7.100 /  3.  7.100
    ```

16. 验证，是否支持H264硬解码

    ```
    # ffmpeg -h encoder=h264_nvenc
    ffmpeg version 4.3 Copyright (c) 2000-2020 the FFmpeg developers
      built with gcc 4.8.5 (GCC) 20150623 (Red Hat 4.8.5-44)
      configuration: --prefix=/usr/local/ffmpeg --extra-cflags='-I/usr/local/cuda-11.0/include -fPIC ' --extra-ldflags='-L/usr/local/cuda-11.0/lib64 -ldl ' --extra-cflags=-I/usr/local/x264/include --extra-ldflags=-L/usr/local/x264/lib --pkg-config-flags=--static --enable-shared --enable-gpl --enable-libx264 --extra-libs=-ldl --enable-cuvid --enable-nvenc --enable-nonfree --nvcc=/usr/local/cuda-11.0/bin/nvcc
      libavutil      56. 51.100 / 56. 51.100
      libavcodec     58. 91.100 / 58. 91.100
      libavformat    58. 45.100 / 58. 45.100
      libavdevice    58. 10.100 / 58. 10.100
      libavfilter     7. 85.100 /  7. 85.100
      libswscale      5.  7.100 /  5.  7.100
      libswresample   3.  7.100 /  3.  7.100
      libpostproc    55.  7.100 / 55.  7.100
    Encoder h264_nvenc [NVIDIA NVENC H.264 encoder]:
        General capabilities: delay hardware 
        Threading capabilities: none
        Supported hardware devices: cuda cuda 
        Supported pixel formats: yuv420p nv12 p010le yuv444p p016le yuv444p16le bgr0 rgb0 cuda
    h264_nvenc AVOptions:
      -preset            <int>        E..V...... Set the encoding preset (from 0 to 11) (default medium)
         default         0            E..V...... 
         slow            1            E..V...... hq 2 passes
         medium          2            E..V...... hq 1 pass
         fast            3            E..V...... hp 1 pass
         hp              4            E..V...... 
         hq              5            E..V...... 
         bd              6            E..V...... 
         ll              7            E..V...... low latency
         llhq            8            E..V...... low latency hq
         llhp            9            E..V...... low latency hp
         lossless        10           E..V...... 
         losslesshp      11           E..V...... 
      -profile           <int>        E..V...... Set the encoding profile (from 0 to 3) (default main)
         baseline        0            E..V...... 
         main            1            E..V...... 
         high            2            E..V...... 
         high444p        3            E..V...... 
      -level             <int>        E..V...... Set the encoding level restriction (from 0 to 51) (default auto)
         auto            0            E..V...... 
         1               10           E..V...... 
         1.0             10           E..V...... 
         1b              9            E..V...... 
         1.0b            9            E..V...... 
         1.1             11           E..V...... 
         1.2             12           E..V...... 
         1.3             13           E..V...... 
         2               20           E..V...... 
         2.0             20           E..V...... 
         2.1             21           E..V...... 
         2.2             22           E..V...... 
         3               30           E..V...... 
         3.0             30           E..V...... 
         3.1             31           E..V...... 
         3.2             32           E..V...... 
         4               40           E..V...... 
         4.0             40           E..V...... 
         4.1             41           E..V...... 
         4.2             42           E..V...... 
         5               50           E..V...... 
         5.0             50           E..V...... 
         5.1             51           E..V...... 
      -rc                <int>        E..V...... Override the preset rate-control (from -1 to INT_MAX) (default -1)
         constqp         0            E..V...... Constant QP mode
         vbr             1            E..V...... Variable bitrate mode
         cbr             2            E..V...... Constant bitrate mode
         vbr_minqp       8388612      E..V...... Variable bitrate mode with MinQP (deprecated)
         ll_2pass_quality 8388616      E..V...... Multi-pass optimized for image quality (deprecated)
         ll_2pass_size   8388624      E..V...... Multi-pass optimized for constant frame size (deprecated)
         vbr_2pass       8388640      E..V...... Multi-pass variable bitrate mode (deprecated)
         cbr_ld_hq       8            E..V...... Constant bitrate low delay high quality mode
         cbr_hq          16           E..V...... Constant bitrate high quality mode
         vbr_hq          32           E..V...... Variable bitrate high quality mode
      -rc-lookahead      <int>        E..V...... Number of frames to look ahead for rate-control (from 0 to INT_MAX) (default 0)
      -surfaces          <int>        E..V...... Number of concurrent surfaces (from 0 to 64) (default 0)
      -cbr               <boolean>    E..V...... Use cbr encoding mode (default false)
      -2pass             <boolean>    E..V...... Use 2pass encoding mode (default auto)
      -gpu               <int>        E..V...... Selects which NVENC capable GPU to use. First GPU is 0, second is 1, and so on. (from -2 to INT_MAX) (default any)
         any             -1           E..V...... Pick the first device available
         list            -2           E..V...... List the available devices
      -delay             <int>        E..V...... Delay frame output by the given amount of frames (from 0 to INT_MAX) (default INT_MAX)
      -no-scenecut       <boolean>    E..V...... When lookahead is enabled, set this to 1 to disable adaptive I-frame insertion at scene cuts (default false)
      -forced-idr        <boolean>    E..V...... If forcing keyframes, force them as IDR frames. (default false)
      -b_adapt           <boolean>    E..V...... When lookahead is enabled, set this to 0 to disable adaptive B-frame decision (default true)
      -spatial-aq        <boolean>    E..V...... set to 1 to enable Spatial AQ (default false)
      -spatial_aq        <boolean>    E..V...... set to 1 to enable Spatial AQ (default false)
      -temporal-aq       <boolean>    E..V...... set to 1 to enable Temporal AQ (default false)
      -temporal_aq       <boolean>    E..V...... set to 1 to enable Temporal AQ (default false)
      -zerolatency       <boolean>    E..V...... Set 1 to indicate zero latency operation (no reordering delay) (default false)
      -nonref_p          <boolean>    E..V...... Set this to 1 to enable automatic insertion of non-reference P-frames (default false)
      -strict_gop        <boolean>    E..V...... Set 1 to minimize GOP-to-GOP rate fluctuations (default false)
      -aq-strength       <int>        E..V...... When Spatial AQ is enabled, this field is used to specify AQ strength. AQ strength scale is from 1 (low) - 15 (aggressive) (from 1 to 15) (default 8)
      -cq                <float>      E..V...... Set target quality level (0 to 51, 0 means automatic) for constant quality mode in VBR rate control (from 0 to 51) (default 0)
      -aud               <boolean>    E..V...... Use access unit delimiters (default false)
      -bluray-compat     <boolean>    E..V...... Bluray compatibility workarounds (default false)
      -init_qpP          <int>        E..V...... Initial QP value for P frame (from -1 to 51) (default -1)
      -init_qpB          <int>        E..V...... Initial QP value for B frame (from -1 to 51) (default -1)
      -init_qpI          <int>        E..V...... Initial QP value for I frame (from -1 to 51) (default -1)
      -qp                <int>        E..V...... Constant quantization parameter rate control method (from -1 to 51) (default -1)
      -weighted_pred     <int>        E..V...... Set 1 to enable weighted prediction (from 0 to 1) (default 0)
      -coder             <int>        E..V...... Coder type (from -1 to 2) (default default)
         default         -1           E..V...... 
         auto            0            E..V...... 
         cabac           1            E..V...... 
         cavlc           2            E..V...... 
         ac              1            E..V...... 
         vlc             2            E..V...... 
      -b_ref_mode        <int>        E..V...... Use B frames as references (from 0 to 2) (default disabled)
         disabled        0            E..V...... B frames will not be used for reference
         each            1            E..V...... Each B frame will be used for reference
         middle          2            E..V...... Only (number of B frames)/2 will be used for reference
      -a53cc             <boolean>    E..V...... Use A53 Closed Captions (if available) (default true)
      -dpb_size          <int>        E..V...... Specifies the DPB size used for encoding (0 means automatic) (from 0 to INT_MAX) (default 0)
    
    
    ```

    







http://fsemouse.com/wordpress/2019/06/04/centos7%E5%AE%89%E8%A3%85ffmpeg%E4%BB%A5%E5%8F%8A%E7%AC%AC%E4%B8%89%E6%96%B9%E4%BE%9D%E8%B5%96/