<template>
  <view class="container">
    <!-- 基础蓝牙操作 -->
    <view class="section">
      <text class="section-header">蓝牙操作</text>
      <view class="button-row">
        <button class="small-btn" @click="startScan" :disabled="isScanning || isConnected">开始扫描</button>
        <button class="small-btn" @click="stopScan" :disabled="!isScanning">停止扫描</button>
        <button class="small-btn" @click="connectDevice" :disabled="!selectedDeviceId || isConnected || isConnecting">连接设备</button>
        <button class="small-btn" @click="disconnectDevice" :disabled="!isConnected">断开连接</button>
      </view>
    </view>
    
    <!-- 显示当前连接的设备 -->
    <view v-if="isConnected" class="connected-device">
      <text>已连接设备: {{ connectedDeviceName || selectedDeviceId }}</text>
      <text>信号强度: {{ connectedDeviceRSSI }}</text>
    </view>
    
    <text class="status">设备状态：{{ deviceStatus }}</text>
    
    <!-- 指令操作区域 -->
    <view class="section" v-if="isConnected">
      <text class="section-header">钥匙指令</text>
      
      <!-- 基础查询指令 -->
      <view class="command-group">
        <text class="group-title">基础查询</text>
        <view class="button-row">
          <button class="cmd-btn" @click="queryKeyInfo">查询钥匙信息</button>
          <button class="cmd-btn" @click="queryKeyRegStatus">查询注册状态</button>
          <button class="cmd-btn" @click="queryKeySNID">查询钥匙SNID</button>
        </view>
      </view>
      
      <!-- 配置管理指令 -->
      <view class="command-group">
        <text class="group-title">配置管理</text>
        <view class="button-row">
          <button class="cmd-btn" @click="resetKeyFactory">恢复出厂配置</button>
          <button class="cmd-btn" @click="setKeyTime">设置校准时间</button>
          <button class="cmd-btn" @click="setKeyPassword">设置通讯密码</button>
        </view>
      </view>
      
      <!-- 钥匙管理指令 -->
      <view class="command-group">
        <text class="group-title">钥匙管理</text>
        <view class="button-row">
          <button class="cmd-btn" @click="registerKey">注册钥匙</button>
          <button class="cmd-btn" @click="verifyKeySNID">验证钥匙SNID</button>
        </view>
      </view>
      
      <!-- 权限设置指令 -->
      <view class="command-group">
        <text class="group-title">权限设置</text>
        <view class="button-row">
          <button class="cmd-btn" @click="setKeyPermissionGeneral">一般授权</button>
          <button class="cmd-btn" @click="setKeyPermissionHighest">最高权限</button>
        </view>
      </view>
    </view>

    <!-- 数据展示区域 -->
    <view class="data-container">
      <text class="section-title">发送数据:</text>
      <text class="hex-data">{{ sentDataHex }}</text>
      
      <text class="section-title">接收数据:</text>
      <text class="hex-data">{{ receivedDataHex }}</text>
    </view>

    <!-- 设备列表 -->
    <view class="section">
      <text class="section-header">设备列表</text>
      <scroll-view scroll-y :scroll-top="0" style="max-height: 200px;">
        <view v-for="(device, index) in devices" :key="index" class="device-item"
          :class="{ 'selected': device.deviceId === selectedDeviceId }"
          @click="selectDevice(device)">
          <text>{{ formatDeviceName(device) }}</text>
          <text class="device-id">ID: {{ device.deviceId }}</text>
          <text class="device-rssi">信号: {{ device.RSSI || '未知' }}</text>
        </view>
      </scroll-view>
      <text v-if="devices.length === 0 && !isScanning">未发现设备</text>
      <text v-if="isScanning">扫描中...</text>
    </view>
  </view>
</template>

<script>
export default {
  data() {
    return {
      deviceId: null,
      serviceId: '0000FFF0-0000-1000-8000-00805F9B34FB',
      characteristicId: {
        write: '0000FFF6-0000-1000-8000-00805F9B34FB',
        notify: '0000FFF7-0000-1000-8000-00805F9B34FB'
      },
      isConnected: false,
      isConnecting: false,
      deviceStatus: '未连接',
      devices: [],
      selectedDeviceId: null,
      isScanning: false,
      scanTimeout: null,
      connectedDeviceName: '',
      connectedDeviceRSSI: 0,
      
      // 数据记录
      sentDataHex: '',
      receivedDataHex: '',
      receivedPackets: []
    };
  },
  methods: {
    // 格式化设备名称
    formatDeviceName(device) {
      if (device.name) return device.name;
      if (device.localName) return device.localName;
      return '未知设备';
    },
    
    // 1. 初始化蓝牙并开始扫描
    startScan() {
      console.log('开始扫描...');
      uni.openBluetoothAdapter({
        success: () => {
          console.log('蓝牙适配器初始化成功');
          uni.getBluetoothAdapterState({
            success: (res) => {
              console.log('蓝牙状态:', res);
              if (!res.available) {
                uni.showToast({ title: '蓝牙不可用', icon: 'none' });
                return;
              }
              this.startDeviceDiscovery();
            },
            fail: (err) => {
              console.error('获取蓝牙状态失败:', err);
              uni.showToast({ title: '获取蓝牙状态失败: ' + err.errMsg, icon: 'none' });
            }
          });
        },
        fail: (err) => {
          console.error('蓝牙初始化失败:', err);
          if (err.errCode === 10001) {
            this.showBluetoothAuthDialog();
          } else {
            uni.showToast({ title: '蓝牙初始化失败: ' + err.errMsg, icon: 'none' });
          }
        }
      });
    },
    
    // 请求蓝牙权限
    showBluetoothAuthDialog() {
      console.log('请求蓝牙权限...');
      uni.showModal({
        title: '需要蓝牙权限',
        content: '请打开系统设置并授予蓝牙权限',
        confirmText: '去设置',
        success: (res) => {
          if (res.confirm) {
            uni.openSetting();
          }
        }
      });
    },
    
    // 开始设备发现
    startDeviceDiscovery() {
      console.log('开始设备发现...');
      this.stopScan();
      
      this.isScanning = true;
      this.devices = [];
      this.selectedDeviceId = null;
      this.deviceStatus = '扫描中...';
      
      const discoveryOptions = {};
      if (this.serviceId && uni.canIUse('startBluetoothDevicesDiscovery.services')) {
        discoveryOptions.services = [this.serviceId];
      }
      
      uni.startBluetoothDevicesDiscovery({
        ...discoveryOptions,
        allowDuplicatesKey: false,
        success: () => {
          console.log('开始扫描成功');
          
          this.deviceFoundHandler = (res) => {
            console.log('发现设备:', res.devices.name);
            res.devices.forEach(device => {
              const deviceName = this.formatDeviceName(device);
              if (deviceName.toUpperCase().startsWith('KEY')) {
                this.addDevice(device);
              }
            });
          };
          uni.onBluetoothDeviceFound(this.deviceFoundHandler);
          
          this.scanTimeout = setTimeout(() => {
            if (this.isScanning) {
              this.stopScan();
              uni.showToast({ title: '扫描超时，未发现设备', icon: 'none' });
            }
          }, 10000);
          uni.showToast({ title: '正在扫描设备...', icon: 'none' });
        },
        fail: (err) => {
          console.error('启动扫描失败:', err);
          let msg = '启动扫描失败';
          if (err.errCode === 10000) msg = '未初始化蓝牙适配器';
          else if (err.errCode === 10001) msg = '未授权使用蓝牙';
          uni.showToast({ title: `${msg}: ${err.errMsg}`, icon: 'none' });
          this.isScanning = false;
        }
      });
    },
    
    // 添加设备（去重）
    addDevice(device) {
      if (!device.deviceId) return;
      const exists = this.devices.some(d => d.deviceId === device.deviceId);
      if (!exists) {
        console.log('添加设备:', device);
        this.devices = [...this.devices, device];
      }
    },
    
    // 2. 停止扫描
    stopScan() {
      console.log('停止扫描...');
      if (this.scanTimeout) {
        clearTimeout(this.scanTimeout);
        this.scanTimeout = null;
      }
      
      if (typeof uni.offBluetoothDeviceFound === 'function') {
        uni.offBluetoothDeviceFound(this.deviceFoundHandler);
      } else {
        console.warn('当前平台不支持 offBluetoothDeviceFound');
      }
      
      this.deviceFoundHandler = null;
      
      uni.stopBluetoothDevicesDiscovery({
        success: () => {
          this.isScanning = false;
          this.deviceStatus = this.devices.length > 0 ? '选择设备' : '未发现设备';
          console.log('扫描已停止');
        },
        fail: (err) => {
          console.error('停止扫描失败:', err);
          uni.showToast({ title: '停止扫描失败: ' + err.errMsg, icon: 'none' });
          this.isScanning = false;
        }
      });
    },
    
    // 3. 选择设备
    selectDevice(device) {
      console.log('选择设备:', device);
      this.selectedDeviceId = device.deviceId;
      this.deviceStatus = `已选择: ${this.formatDeviceName(device)}`;
    },

    // 4. 连接设备
    connectDevice() {
      console.log('连接设备:', this.selectedDeviceId);
      if (!this.selectedDeviceId) {
        uni.showToast({ title: '请先选择设备', icon: 'none' });
        return;
      }

      const selectedDevice = this.devices.find(d => d.deviceId === this.selectedDeviceId);
      if (selectedDevice) {
        this.connectedDeviceName = this.formatDeviceName(selectedDevice);
        this.connectedDeviceRSSI = selectedDevice.RSSI || 0;
      }

      this.deviceId = this.selectedDeviceId;
      this.isConnecting = true;
      this.deviceStatus = '连接中...';

      this.stopScan();

      uni.createBLEConnection({
        deviceId: this.deviceId,
        success: () => {
          console.log('连接成功');
          this.isConnected = true;
          this.isConnecting = false;
          this.deviceStatus = '已连接';
          uni.showToast({ title: '连接成功', icon: 'success' });
          
          setTimeout(() => {
            this.getServices();
          }, 1500);
        },
        fail: (err) => {
          console.error('连接失败:', err);
          this.isConnecting = false;
          this.deviceStatus = '连接失败';
          uni.showToast({ title: '连接失败: ' + err.errMsg, icon: 'none' });
        }
      });

      uni.onBLEConnectionStateChange((res) => {
        console.log('连接状态变化:', res);
        if (res.deviceId === this.deviceId && !res.connected) {
          this.isConnected = false;
          this.deviceStatus = '已断开';
          uni.showToast({ title: '设备已断开', icon: 'none' });
        }
      });
    },

    // 5. 获取服务
    getServices() {
      console.log('获取服务...');
      uni.getBLEDeviceServices({
        deviceId: this.deviceId,
        success: (res) => {
          console.log('设备服务列表:', res.services);
          const targetService = res.services.find(
            s => s.uuid.toLowerCase() === this.serviceId.toLowerCase()
          );
          
          if (targetService) {
            this.getCharacteristics(targetService.uuid);
          } else {
            console.error('目标服务未找到');
            uni.showToast({ title: '未找到目标服务', icon: 'none' });
          }
        },
        fail: (err) => {
          console.error('获取服务失败:', err);
          uni.showToast({ title: '获取服务失败: ' + err.errMsg, icon: 'none' });
        }
      });
    },

    // 6. 获取特征值
    getCharacteristics(serviceId) {
      console.log('获取特征值...', serviceId);
      uni.getBLEDeviceCharacteristics({
        deviceId: this.deviceId,
        serviceId: serviceId,
        success: (res) => {
          console.log('特征值列表:', res.characteristics);
          
          let writeChar = null;
          let notifyChar = null;
          
          res.characteristics.forEach(char => {
            const charUUID = char.uuid.toLowerCase();
            if (char.properties.write && charUUID === this.characteristicId.write.toLowerCase()) {
              writeChar = char;
            }
            if ((char.properties.notify || char.properties.indicate) && charUUID === this.characteristicId.notify.toLowerCase()) {
              notifyChar = char;
            }
          });
          
          if (writeChar && notifyChar) {
            console.log('找到写入和通知特征');
            this.characteristicId.write = writeChar.uuid;
            this.characteristicId.notify = notifyChar.uuid;
            this.enableNotifications(serviceId, notifyChar.uuid);
          } else {
            console.warn('特征值不匹配');
            let missing = [];
            if (!writeChar) missing.push('写入特征');
            if (!notifyChar) missing.push('通知特征');
            uni.showToast({ title: `缺少特征值: ${missing.join(', ')}`, icon: 'none' });
          }
        },
        fail: (err) => {
          console.error('获取特征值失败:', err);
          uni.showToast({ title: '获取特征值失败: ' + err.errMsg, icon: 'none' });
        }
      });
    },

    // 7. 启用通知
    enableNotifications(serviceId, characteristicId) {
      console.log('启用通知...', serviceId, characteristicId);
      uni.notifyBLECharacteristicValueChange({
        deviceId: this.deviceId,
        serviceId: serviceId,
        characteristicId: characteristicId,
        state: true,
        success: () => {
          console.log('通知启用成功');
          uni.onBLECharacteristicValueChange((res) => {
            const bytes = new Uint8Array(res.value);
            const hexStr = Array.from(bytes).map(b => 
              b.toString(16).padStart(2, '0').toUpperCase()
            ).join(' ');
            
            console.log('收到数据包:', hexStr);
            this.receivedPackets.push(hexStr);
            this.receivedDataHex = this.receivedPackets.join(' ');
            this.deviceStatus = `已接收 ${this.receivedPackets.length} 个数据包`;
          });
          this.deviceStatus = '已启用通知';
          uni.showToast({ title: '通知已启用', icon: 'success' });
        },
        fail: (err) => {
          console.error('启用通知失败:', err);
          uni.showToast({ title: '启用通知失败: ' + err.errMsg, icon: 'none' });
        }
      });
    },
    
    // 8. 断开连接
    disconnectDevice() {
      console.log('断开连接...');
      uni.closeBLEConnection({
        deviceId: this.deviceId,
        success: () => {
          console.log('断开连接成功');
          this.isConnected = false;
          this.deviceStatus = '已断开';
          this.selectedDeviceId = null;
          uni.showToast({ title: '已断开连接', icon: 'success' });
          
          this.sentDataHex = '';
          this.receivedDataHex = '';
          this.receivedPackets = [];
        },
        fail: (err) => {
          console.error('断开失败:', err);
          uni.showToast({ title: '断开失败: ' + err.errMsg, icon: 'none' });
        }
      });
    },

    // 指令发送的通用方法
    sendCommand(hexData, commandName) {
      if (!this.isConnected) {
        uni.showToast({ title: '未连接设备', icon: 'none' });
        return;
      }
      
      console.log(`发送${commandName}指令...`);
      this.receivedPackets = [];
      this.receivedDataHex = '';
      this.sentDataHex = hexData;
      
      const bytes = hexData.split(" ").map(b => parseInt(b, 16));
      const buffer = new ArrayBuffer(bytes.length);
      const dataView = new DataView(buffer);
      for (let i = 0; i < bytes.length; i++) {
        dataView.setUint8(i, bytes[i]);
      }

      console.log(`发送${commandName}完整数据:`, hexData);
      this.sendBlePacket(dataView.buffer);
    },

    // 各个指令的具体方法
    // 查询钥匙信息 (0x05 0x06)
    queryKeyInfo() {
      const hexData = "74 68 78 64 7A 56 31 2E 30 2E 30 30 05 00 04 00 00 00 56 5B BC 2D 77 88 55 AA";
      this.sendCommand(hexData, "查询钥匙信息");
    },

    // 查询钥匙注册状态 (0x33 0x34)
    queryKeyRegStatus() {
      const hexData = "74 68 78 64 7A 56 31 2E 30 2E 30 30 33 00 04 00 00 00 E6 BF 9F FF 77 88 55 AA";
      this.sendCommand(hexData, "查询钥匙注册状态");
    },

    // 恢复钥匙出厂配置 (0x1A 0x1B)
    resetKeyFactory() {
      const hexData = "74 68 78 64 7A 56 31 2E 30 2E 30 30 1A 00 04 00 00 00 18 EB 3C DF 77 88 55 AA";
      this.sendCommand(hexData, "恢复钥匙出厂配置");
    },

    // 验证钥匙SNID (0x39 0x3A)
    verifyKeySNID() {
      const hexData = "74 68 78 64 7A 56 31 2E 30 2E 30 30 39 00 10 00 00 00 12 82 0A 8B 6F F7 E3 19 D0 96 B3 32 42 88 39 F8 77 88 55 AA";
      this.sendCommand(hexData, "验证钥匙SNID");
    },

    // 查询钥匙SNID (0x35 0x36)
    queryKeySNID() {
      const hexData = "74 68 78 64 7A 56 31 2E 30 2E 30 30 35 00 08 00 00 00 34 9F 6B A5 B5 11 8B 34 77 88 55 AA";
      this.sendCommand(hexData, "查询钥匙SNID");
    },

    // 设置钥匙校准时间 (0x09 0x0A)
    setKeyTime() {
      const hexData = "74 68 78 64 7A 56 31 2E 30 2E 30 30 09 00 08 00 00 00 B8 A7 B3 65 0F 43 36 19 77 88 55 AA";
      this.sendCommand(hexData, "设置钥匙校准时间");
    },

    // 注册钥匙 (0x37 0x38)
    registerKey() {
      const hexData = "74 68 78 64 7A 56 31 2E 30 2E 30 30 37 00 10 00 00 00 52 FF 6D 06 72 70 50 55 17 44 03 67 E9 D2 E1 71 77 88 55 AA";
      this.sendCommand(hexData, "注册钥匙");
    },

    // 设置钥匙通讯密码 (0x24 0x25)
    setKeyPassword() {
      const hexData = "74 68 78 64 7A 56 31 2E 30 2E 30 30 24 00 0C 00 00 00 31 31 32 32 33 33 34 34 0F F4 8A 5E 77 88 55 AA";
      this.sendCommand(hexData, "设置钥匙通讯密码");
    },

    // 设置钥匙权限（离线）一般授权 (0x0B 0x0C 0x0D 0x0E)
    setKeyPermissionGeneral() {
      uni.showLoading({ title: '发送一般授权...' });
      
      // 第一步：发送0x0B指令
      const hexData1 = "74 68 78 64 7A 56 31 2E 30 2E 30 30 0B 00 29 00 00 00 02 00 00 00 00 00 00 00 00 FF FF FF FF 02 00 00 00 00 F6 B2 65 FF DE BA 65 00 00 00 00 7F 51 01 00 05 00 00 00 09 A7 4C 06 77 88 55 AA";
      this.sendCommand(hexData1, "设置钥匙权限(步骤1)");
      
      // 延迟发送后续指令
      setTimeout(() => {
        const hexData2 = "74 68 78 64 7A 56 31 2E 30 2E 30 30 0D 00 10 00 00 00 01 00 00 00 01 00 00 00 2B 00 00 00 7B D5 03 11 77 88 55 AA";
        this.sendCommand(hexData2, "设置钥匙权限(步骤2)");
      }, 1000);
      
      setTimeout(() => {
        const hexData3 = "74 68 78 64 7A 56 31 2E 30 2E 30 30 0D 00 10 00 00 00 02 00 00 00 64 00 00 00 01 00 00 00 0E E6 19 31 77 88 55 AA";
        this.sendCommand(hexData3, "设置钥匙权限(步骤3)");
        uni.hideLoading();
      }, 2000);
    },

    // 设置钥匙权限（离线）单位最高 (0x0B 0x0C)
    setKeyPermissionHighest() {
      const hexData = "74 68 78 64 7A 56 31 2E 30 2E 30 30 0B 00 29 00 00 00 01 00 00 00 00 00 00 00 00 FF FF FF FF FF FF FF FF 00 F6 B2 65 FF DE BA 65 00 00 00 00 7F 51 01 00 00 00 00 00 8B 91 F6 EE 77 88 55 AA";
      this.sendCommand(hexData, "设置钥匙权限(单位最高)");
    },

    // 10. 发送BLE数据包
    sendBlePacket(data) {
      console.log('开始发送数据包...');
      const packetSize = 20;
      const totalLength = data.byteLength;
      let offset = 0;
      let packetCount = 0;
      const totalPackets = Math.ceil(totalLength / packetSize);
      const sentPackets = [];

      const sendNext = () => {
        if (offset >= totalLength) {
          console.log('发送完成');
          uni.showToast({ title: '发送完成', icon: 'success' });
          sentPackets.forEach((packet, index) => {
            console.log(`包 ${index + 1}/${sentPackets.length}: ${packet}`);
          });
          return;
        }

        const end = Math.min(offset + packetSize, totalLength);
        const packet = data.slice(offset, end);
        offset = end;
        packetCount++;

        const bytes = new Uint8Array(packet);
        const hexStr = Array.from(bytes).map(b => 
          b.toString(16).padStart(2, '0').toUpperCase()
        ).join(' ');
        
        sentPackets.push(hexStr);
        console.log(`发送包 ${packetCount}/${totalPackets}: ${hexStr}`);

        uni.writeBLECharacteristicValue({
          deviceId: this.deviceId,
          serviceId: this.serviceId,
          characteristicId: this.characteristicId.write,
          value: packet,
          success: () => {
            this.deviceStatus = `发送中 (${packetCount}/${totalPackets})`;
            setTimeout(sendNext, 100);
          },
          fail: (err) => {
            console.error(`发送包 ${packetCount} 失败:`, err);
            uni.showToast({ title: `第${packetCount}包发送失败: ${err.errMsg}`, icon: 'none' });
          }
        });
      };

      sendNext();
    }
  },
  onUnload() {
    console.log('页面卸载，清理资源...');
    this.stopScan();
    if (this.isConnected) {
      this.disconnectDevice();
    }
    uni.closeBluetoothAdapter();
  }
};
</script>

<style>
.container {
  padding: 15px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.section {
  background-color: #f8f9fa;
  padding: 12px;
  border-radius: 8px;
  margin-bottom: 8px;
}

.section-header {
  font-size: 16px;
  font-weight: bold;
  color: #333;
  display: block;
  margin-bottom: 10px;
}

.button-row {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-bottom: 8px;
}

.small-btn {
  flex: 1;
  min-width: 70px;
  padding: 8px 10px;
  background-color: #007AFF;
  color: white;
  border-radius: 6px;
  text-align: center;
  font-size: 12px;
  border: none;
}

.small-btn:disabled {
  background-color: #cccccc;
  color: #666666;
}

.cmd-btn {
  flex: 1;
  min-width: 80px;
  padding: 6px 8px;
  background-color: #52c41a;
  color: white;
  border-radius: 4px;
  text-align: center;
  font-size: 11px;
  border: none;
  margin: 2px;
}

.cmd-btn:disabled {
  background-color: #d9d9d9;
  color: #8c8c8c;
}

.command-group {
  margin-bottom: 12px;
}

.group-title {
  font-size: 13px;
  font-weight: bold;
  color: #666;
  display: block;
  margin-bottom: 6px;
}

.connected-device {
  background-color: #e6f7ff;
  padding: 8px;
  border-radius: 6px;
  border-left: 3px solid #1890ff;
  margin-bottom: 8px;
}

.connected-device text {
  display: block;
  font-size: 12px;
  margin: 2px 0;
}

.status {
  font-size: 14px;
  color: #666;
  text-align: center;
  padding: 5px;
}

.device-item {
  padding: 10px;
  border-bottom: 1px solid #eee;
  background-color: #fff;
  margin: 2px 0;
  border-radius: 4px;
}

.device-item.selected {
  background-color: #e6f7ff;
  border-left: 3px solid #1890ff;
}

.device-id, .device-rssi {
  font-size: 11px;
  color: #999;
  display: block;
  margin-top: 3px;
}

.data-container {
  background-color: #f9f9f9;
  padding: 10px;
  border-radius: 6px;
  margin: 8px 0;
}

.section-title {
  font-weight: bold;
  display: block;
  margin-top: 6px;
  color: #1890ff;
  font-size: 13px;
}

.hex-data {
  font-family: monospace;
  word-break: break-all;
  font-size: 11px;
  display: block;
  margin: 4px 0;
  background-color: #fff;
  padding: 6px;
  border-radius: 3px;
  border: 1px solid #eee;
  max-height: 60px;
  overflow-y: auto;
}
</style>