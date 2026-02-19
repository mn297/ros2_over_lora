Our goal is IP over SIK 915 mhz radio using ppp and have ROS 2 working over that

Here are proof of concept using socat emulated serial

## CASE 1: emulate 2 serials on 1 machine

<!-- 1 -->
```bash
sudo pkill pppd || true
sudo pkill socat || true
sudo rm -f /tmp/ttyA /tmp/ttyB
```

<!-- 2 -->
```bash
sudo socat -d -d \
  pty,raw,echo=0,link=/tmp/ttyA \
  pty,raw,echo=0,link=/tmp/ttyB
```

<!-- 3 -->
```bash
sudo pppd /tmp/ttyA 57600 \
  10.10.10.1:10.10.10.2 \
  noauth local debug nodetach nocrtscts nodefaultroute \
  persist maxfail 0
```

<!-- 4 -->
```bash
sudo pppd /tmp/ttyB 57600 \
  10.10.10.2:10.10.10.1 \
  noauth local debug nodetach nocrtscts nodefaultroute \
  persist maxfail 0
```

<!-- 5 -->
```bash
ip -br addr show ppp0 ppp1
ip addr show ppp0
ip addr show ppp1
```

## CASE 2: both radios plugged in 1 machine

```bash
sudo pppd /dev/ttyUSB0 57600 \
  10.10.10.1:10.10.10.2 \
  noauth local debug nodetach nocrtscts nodefaultroute \
  persist maxfail 0 mtu 296 mru 296

sudo pppd /dev/ttyUSB1 57600 \
  10.10.10.2:10.10.10.1 \
  noauth local debug nodetach nocrtscts nodefaultroute \
  persist maxfail 0 mtu 296 mru 296
```

## CASE 3: real hardware

Radio is usually `/dev/ttyUSB0`.

**Setup PPP (run on each machine with its own IP):**
```bash
# Machine A
sudo pppd /dev/ttyUSB0 57600 \
  10.10.10.1:10.10.10.2 \
  noauth local debug nodetach nocrtscts nodefaultroute \
  persist maxfail 0 mtu 128 mru 128

# Machine B
sudo pppd /dev/ttyUSB0 57600 \
  10.10.10.2:10.10.10.1 \
  noauth local debug nodetach nocrtscts nodefaultroute \
  persist maxfail 0 mtu 128 mru 128
```

**Test PPP:**
```bash
ping 10.10.10.2   # from A
ping 10.10.10.1   # from B
```

**Setup ROS network config for both:**
```bash
export ROS_DOMAIN_ID=0          # or any same number on both
export ROS_LOCALHOST_ONLY=0
export ROS_DISCOVERY_SERVER=10.10.10.1:11811
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
```

**Machine A:**
```bash
fastdds discovery --server-id 0 --udp-address 10.10.10.1 --udp-port 11811
ros2 topic pub -r 10 /chatter std_msgs/msg/String "{data: 'hello'}"
```

Now B should be able to:
```bash
ros2 topic list
ros2 topic echo /chatter
ros2 topic hz /chatter
ros2 topic bw /chatter
```

If not:
```bash
export ROS_SUPER_CLIENT=TRUE
ros2 topic list
```

**Teardown:**
```bash
sudo pkill pppd || true
sudo pkill socat || true
sudo rm -f /tmp/ttyA /tmp/ttyB
sudo kill "$(cat /tmp/pppA.pid)"
```

# ros2_over_lora
