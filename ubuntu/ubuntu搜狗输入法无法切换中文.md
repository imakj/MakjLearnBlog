# ubuntu20.04 搜狗输入法无法切换中文

1. 下载搜狗输入法安装

   ```
   $ sudo apt install fcitx-bin
   $ sudo apt-get install fcitx-table
   $ sudo dpkg -i sougou*.deb
   ```

   如果报错，安装依赖

   ```
   $ sudo apt-get install -f
   ```

2. 设置语言为fcitx并且重启

   **settings–>Region&language–>Manage Installed Languages**

3. 如果这个时候还无法输入中文，查看系统日志缺少哪个装哪个

   ```
   $ tail -f /var/log/syslog | grep sogou
   ```

   我的缺少**libqt5qml5**和**libgsettings-qt1**

4. 安装

   ```
   $ sudo apt install libqt5qml5
   sudo apt install libgsettings-qt1
   ```

5. 安装完重启，输入法正常