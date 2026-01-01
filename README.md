# ubuntu 系统 X9DR3-F 与 X9Di-F 风扇转速调节

# 相关链接
[官方文档](https://www.supermicro.com/manuals/motherboard/C606_602/MNL-1259.pdf)
实物图 page 10
风扇相关 page 52

[BIOS更新](https://www.supermicro.com/en/support/resources/downloadcenter/firmware/MBD-X9DR3-F/BIOS)

[BMC更新](https://www.supermicro.com/en/support/resources/downloadcenter/firmware/MBD-X9DR3-F/BMC)

[IPMI fan control for Supermicro X9 motherboard--any way to GET the duty cycle?](https://www.reddit.com/r/homelab/comments/vi9mj3/ipmi_fan_control_for_supermicro_x9_motherboardany/)

[Supermicro X9/X10/X11 Fan Speed Control](https://forums.servethehome.com/index.php?threads/supermicro-x9-x10-x11-fan-speed-control.10059/page-10)

[教程视频](https://www.bilibili.com/video/BV1SZvUB6E1Q)

# 没验证成功
### 查看风扇模式
``` bash
sudo ipmitool raw 0x30 0x45 0x00
```

### 调整风扇模式
``` bash
# Full Speed
sudo ipmitool raw 0x30 0x45 0x01 0x00
# Optimal
sudo ipmitool raw 0x30 0x45 0x01 0x01
# Heavy IO
sudo ipmitool raw 0x30 0x45 0x01 0x02
# Standard
sudo ipmitool raw 0x30 0x45 0x01 0x03
```

# 验证成功
``` bash
# 查看风扇转速
sudo ipmitool sensor | grep -i fan
# 查看风扇状态
sudo ipmitool sensor get "FAN1"
sudo ipmitool sensor get "FANB"
# 调整风扇阈值
sudo ipmitool sensor thresh "FAN1" lower 200 300 400
sudo ipmitool sensor thresh "FANB" lower 200 300 400
```

``` bash
# 手动模式
sudo ipmitool raw 0x30 0x45 0x01 0x01
# 设置转速 0x10, 0x11 区域，包含多个风扇接口
# 0x10包含 FAN1-6
# 0x11包含 FANA,B
# 0x32 转速 0x00-0xFF
sudo ipmitool raw 0x30 0x91 0x5A 0x03 0x10 0x32
sudo ipmitool raw 0x30 0x91 0x5A 0x03 0x11 0x32
```

# 烤机软件

CPU stress

GPU gpu_burn


# 风扇配置脚本
# 保存脚本
``` bash
sudo nano /usr/local/bin/fan-control.sh
```
### /usr/local/bin/fan-control.sh
``` bash
#!/bin/bash

while true; do
  cpu0_temp=$(sensors | grep 'Package id 0:' | awk '{print int($4)}')
  cpu1_temp=$(sensors | grep 'Package id 1:' | awk '{print int($4)}')
  cpu_temp=$(echo "$cpu0_temp $cpu1_temp" | awk '{print ($1>$2)?$1:$2}')
  gpu_temp=$(nvidia-smi -i 0 --query-gpu=temperature.gpu --format=csv,noheader,nounits | awk '{print int($1)}')

  # CPU zone
  if [ "$cpu_temp" -lt 35 ]; then
    cpu_pwm=0x2F
  elif [ "$cpu_temp" -lt 45 ]; then
    cpu_pwm=0x3F
  elif [ "$cpu_temp" -lt 55 ]; then
    cpu_pwm=0x4F
  elif [ "$cpu_temp" -lt 60 ]; then
    cpu_pwm=0x7F
  elif [ "$cpu_temp" -lt 65 ]; then
    cpu_pwm=0xAF
  else
    cpu_pwm=0xFF
  fi

  # GPU zone
  if [ "$gpu_temp" -lt 35 ]; then
    gpu_pwm=0x2F
  elif [ "$gpu_temp" -lt 40 ]; then
    cpu_pwm=0x3F
  elif [ "$gpu_temp" -lt 45 ]; then
    cpu_pwm=0x4F
  elif [ "$gpu_temp" -lt 50 ]; then
    gpu_pwm=0x8F
  elif [ "$gpu_temp" -lt 55 ]; then
    gpu_pwm=0xAF
  elif [ "$gpu_temp" -lt 60 ]; then
    gpu_pwm=0xCF
  else
    gpu_pwm=0xFF
  fi

  # 应用设置
  ipmitool raw 0x30 0x91 0x5A 0x03 0x10 $cpu_pwm
  ipmitool raw 0x30 0x91 0x5A 0x03 0x11 $gpu_pwm

  # 日志（可选）
  # echo "$(date) CPU=$cpu_temp GPU=$gpu_temp PWM=C$cpu_pwm/G$gpu_pwm" >> /var/log/fan-control.log
  echo "$(date) CPU=$cpu_temp GPU=$gpu_temp PWM=C$cpu_pwm/G$gpu_pwm" >> /tmp/fan-control.log

  sleep 2
done

```

# 赋予执行权限
``` bash
sudo chmod +x /usr/local/bin/fan-control.sh
```

# 创建 systemd 服务文件
``` bash
sudo nano /etc/systemd/system/fan-control.service
```
### /etc/systemd/system/fan-control.service
``` bash
[Unit]
Description=Fan Control Service
After=network.target

[Service]
ExecStart=/usr/local/bin/fan-control.sh
Restart=always
User=root

[Install]
WantedBy=multi-user.target

```

# 启用并启动服务
``` bash
sudo systemctl daemon-reload
sudo systemctl enable fan-control.service
sudo systemctl start fan-control.service
```
## 查看运行状态：
```bash
systemctl status fan-control.service
```
