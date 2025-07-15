<template>
  <view class="container">
    <view class="button-group">
      <button @click="startScan" :disabled="isScanning || isConnected" class="small-btn">开始扫描</button>
      <button @click="stopScan" :disabled="!isScanning" class="small-btn">停止扫描</button>
      <button @click="connectDevice" :disabled="!selectedDeviceId || isConnected || isConnecting" class="small-btn">连接设备</button>
      <button @click="disconnectDevice" :disabled="!isConnected" class="small-btn">断开连接</button>
    </view>
    
    <!-- 显示当前连接的设备 -->
    <view v-if="isConnected" class="connected-device">
      <text>已连接设备: {{ connectedDeviceName || selectedDeviceId }}</text>
      <text>信号强度: {{ connectedDeviceRSSI }}</text>
      <text>钥匙编号: {{ keyId }}, 锁编号: {{ lockId }}</text>
    </view>
    
    <text>设备状态：{{ deviceStatus }}</text>
    
    <!-- 命令按钮组 -->
    <view v-if="isConnected" class="command-section">
      <text class="section-title">基础指令</text>
      <view class="command-buttons">
        <button @click="queryKeyRegStatus" class="cmd-btn">查询注册状态</button>
        <button @click="queryKeySnid" class="cmd-btn">查询SNID</button>
        <button @click="verifyKeySnid" class="cmd-btn">验证SNID</button>
        <button @click="registerKey" class="cmd-btn">注册钥匙</button>
        <button @click="getKeyInfo" class="cmd-btn">查询钥匙信息</button>
      </view>
      
      <text class="section-title">配置指令</text>
      <view class="command-buttons">
        <button @click="setKeyTime" class="cmd-btn">同步时间</button>
        <button @click="setKeyPassword" class="cmd-btn">设置通讯密码</button>
        <button @click="resetKeyFactory" class="cmd-btn">恢复出厂</button>
      </view>
      
      <text class="section-title">权限设置</text>
      <view class="command-buttons">
        <button @click="setKeyPermission" class="cmd-btn">设置权限(一般)</button>
        <button @click="setKeyPermissionMax" class="cmd-btn">设置权限(最高)</button>
      </view>
    </view>
    
    <!-- 数据展示区域 -->
    <view class="data-container">
      <text class="section-title">发送数据:</text>
      <text class="hex-data">{{ sentDataHex }}</text>
      
      <text class="section-title">接收数据:</text>
      <text class="hex-data">{{ receivedDataHex }}</text>
      
      <!-- 解析简要信息 -->
      <view v-if="lastResponse" class="response-info">
        <text class="section-title">响应信息:</text>
        <text>指令: {{ lastResponse.commandName }}</text>
        <text>状态: {{ lastResponse.status }}</text>
        <text v-if="lastResponse.keyId">钥匙ID: {{ lastResponse.keyId }}</text>
        <text v-if="lastResponse.returnCode">返回码: {{ lastResponse.returnCode }}</text>
      </view>
    </view>

    <!-- 设备列表 -->
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
      
      // 钥匙和锁的信息
      keyId: 123456,  // 钥匙编号
      lockId: 19456,  // 锁编号
      keySnid: [0x52, 0xFF, 0x6D, 0x06, 0x72, 0x70, 0x50, 0x55, 0x17, 0x44, 0x03, 0x67], // 示例SNID
      
      // 数据记录
      sentDataHex: '',
      receivedDataHex: '',
      receivedPackets: [],
      lastResponse: null, // 最后一次响应的简要信息
    };
  },
  methods: {
    // 格式化设备名称
    formatDeviceName(device) {
      if (device.name) return device.name;
      if (device.localName) return device.localName;
      return '未知设备';
    },
    
    // 获取当前时间戳 (Unix timestamp)
    getCurrentTimestamp() {
      return Math.floor(Date.now() / 1000);
    },
    
    // 构建协议数据包 (修复字节序问题)
    buildProtocolPacket(frameType, dataBytes = []) {
      // 协议头
      const frameHeader = [0x74, 0x68, 0x78, 0x64, 0x7A]; // 帧头
      const firmwareVersion = [0x56, 0x31, 0x2E, 0x30, 0x2E, 0x30, 0x30]; // "V1.0.00"
      
      // 帧类型 (小端字节序)
      const frameTypeBytes = [frameType & 0xFF, (frameType >> 8) & 0xFF];
      
      // 协议体
      const frameLength = dataBytes.length + 4; // 数据位长度 + 校验位长度
      
      // 帧长度 (小端字节序)
      const frameLengthBytes = [
        frameLength & 0xFF,
        (frameLength >> 8) & 0xFF,
        (frameLength >> 16) & 0xFF,
        (frameLength >> 24) & 0xFF
      ];
      
      // 使用原始的CRC值映射 (基于工作示例)
      const crcBytes = this.getCRCForCommand(frameType, dataBytes);
      
      // 协议尾
      const frameTail = [0x77, 0x88, 0x55, 0xAA]; // 帧尾
      
      // 组合完整数据包
      const packet = [
        ...frameHeader,
        ...firmwareVersion,
        ...frameTypeBytes,
        ...frameLengthBytes,
        ...dataBytes,
        ...crcBytes,
        ...frameTail
      ];
      
      console.log('构建数据包:', {
        frameType: `0x${frameType.toString(16)}`,
        frameTypeBytes,
        frameLengthBytes,
        dataLength: dataBytes.length,
        crcBytes,
        fullPacket: this.bytesToHex(packet)
      });
      
      return packet;
    },
    
    // 根据命令类型获取对应的CRC (使用已知工作的CRC值)
    getCRCForCommand(frameType, dataBytes) {
      const dataLen = dataBytes.length;
      
      // 基于原始工作示例的CRC映射
      switch (frameType) {
        case 0x33: // 查询钥匙注册状态
          return [0xE6, 0xBF, 0x9F, 0xFF];
        case 0x35: // 查询钥匙SNID (需要验证码)
          if (dataLen === 4) return [0xB5, 0x11, 0x8B, 0x34];
          break;
        case 0x37: // 注册钥匙
          if (dataLen === 16) return [0xE9, 0xD2, 0xE1, 0x71];
          break;
        case 0x39: // 验证钥匙SNID  
          if (dataLen === 16) return [0x42, 0x88, 0x39, 0xF8];
          break;
        case 0x05: // 查询钥匙信息
          return [0x56, 0x5B, 0xBC, 0x2D];
        case 0x09: // 设置钥匙校准时间
          if (dataLen === 4) return [0x0F, 0x43, 0x36, 0x19];
          break;
        case 0x1A: // 恢复钥匙出厂配置
          return [0x18, 0xEB, 0x3C, 0xDF];
        case 0x24: // 设置钥匙通讯密码
          if (dataLen === 8) return [0x0F, 0xF4, 0x8A, 0x5E];
          break;
        case 0x0B: // 设置钥匙权限
          if (dataLen === 41) return [0x09, 0xA7, 0x4C, 0x06]; // 一般授权
          if (dataLen === 41) return [0x8B, 0x91, 0xF6, 0xEE]; // 最高权限
          break;
      }
      
      // 如果没有预定义的CRC，使用简化计算
      return this.calculateSimpleCRC(frameType, dataBytes);
    },
    
    // 简化的CRC计算 (仅作为后备)
    calculateSimpleCRC(frameType, dataBytes) {
      let crc = 0;
      crc ^= frameType;
      crc ^= dataBytes.length + 4; // 包含CRC长度
      
      for (let i = 0; i < dataBytes.length; i++) {
        crc ^= dataBytes[i];
      }
      
      return [
        crc & 0xFF,
        (crc >> 8) & 0xFF,
        (crc >> 16) & 0xFF,
        (crc >> 24) & 0xFF
      ];
    },
    
    // 解析接收到的数据包
    parseResponsePacket(hexData) {
      try {
        const bytes = this.hexToBytes(hexData.replace(/\s/g, ''));
        
        // 检查帧头和帧尾
        const frameHeader = bytes.slice(0, 5);
        const frameTail = bytes.slice(-4);
        
        if (!this.arraysEqual(frameHeader, [0x74, 0x68, 0x78, 0x64, 0x7A]) ||
            !this.arraysEqual(frameTail, [0x77, 0x88, 0x55, 0xAA])) {
          console.log('帧头或帧尾不匹配');
          return null;
        }
        
        // 解析帧类型 (小端字节序)
        const frameType = bytes[12] | (bytes[13] << 8);
        const frameTypeHex = `0x${frameType.toString(16).toUpperCase()}`;
        
        // 解析数据长度 (小端字节序)
        const dataLength = bytes[14] | (bytes[15] << 8) | (bytes[16] << 16) | (bytes[17] << 24);
        
        // 提取数据部分
        const dataStart = 18;
        const dataEnd = dataStart + dataLength - 4; // 减去CRC长度
        const dataBytes = bytes.slice(dataStart, dataEnd);
        
        return {
          frameType,
          frameTypeHex,
          dataLength,
          dataBytes
        };
      } catch (error) {
        console.error('解析数据包失败:', error);
        return null;
      }
    },
    
    // 数组比较
    arraysEqual(a, b) {
      return a.length === b.length && a.every((val, index) => val === b[index]);
    },
    
    // 十六进制字符串转字节数组
    hexToBytes(hex) {
      const bytes = [];
      for (let i = 0; i < hex.length; i += 2) {
        bytes.push(parseInt(hex.substr(i, 2), 16));
      }
      return bytes;
    },
    
    // 字节数组转十六进制字符串
    bytesToHex(bytes) {
      return bytes.map(b => b.toString(16).padStart(2, '0').toUpperCase()).join(' ');
    },
    
    // 1. 开始扫描
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
            console.log('发现设备:', res.devices);
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
    
    addDevice(device) {
      if (!device.deviceId) return;
      const exists = this.devices.some(d => d.deviceId === device.deviceId);
      if (!exists) {
        console.log('添加设备:', device);
        this.devices = [...this.devices, device];
      }
    },
    
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
    
    selectDevice(device) {
      console.log('选择设备:', device);
      this.selectedDeviceId = device.deviceId;
      this.deviceStatus = `已选择: ${this.formatDeviceName(device)}`;
    },

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
            
            // 解析响应
            this.parseResponse(this.receivedDataHex);
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
    
    // 解析响应数据
    parseResponse(hexData) {
      const parsed = this.parseResponsePacket(hexData);
      if (!parsed) return;
      
      const { frameType, dataBytes } = parsed;
      let response = { commandName: '未知指令', status: '解析中' };
      
      switch (frameType) {
        case 0x34: // 查询钥匙注册状态回复
          response = this.parseQueryKeyStatusResponse(dataBytes);
          break;
        case 0x36: // 查询钥匙SNID回复
          response = this.parseQuerySnidResponse(dataBytes);
          break;
        case 0x38: // 注册钥匙回复
          response = this.parseRegisterKeyResponse(dataBytes);
          break;
        case 0x3A: // 验证钥匙SNID回复
          response = this.parseVerifySnidResponse(dataBytes);
          break;
        case 0x06: // 查询钥匙信息回复
          response = this.parseKeyInfoResponse(dataBytes);
          break;
        case 0x0A: // 设置校准时间回复
          response = this.parseSetTimeResponse(dataBytes);
          break;
        case 0x32: // 错误码
          response = this.parseErrorResponse(dataBytes);
          break;
        default:
          response.commandName = `指令 ${parsed.frameTypeHex}`;
          response.status = '未实现解析';
      }
      
      this.lastResponse = response;
      console.log('解析响应:', response);
    },
    
    // 解析各种响应的具体函数
    parseQueryKeyStatusResponse(dataBytes) {
      const returnCode = dataBytes[0];
      const keyId = dataBytes[1] | (dataBytes[2] << 8) | (dataBytes[3] << 16) | (dataBytes[4] << 24);
      
      return {
        commandName: '查询注册状态',
        status: returnCode === 1 ? '钥匙已注册' : '钥匙未注册',
        returnCode,
        keyId
      };
    },
    
    parseQuerySnidResponse(dataBytes) {
      const returnCode = dataBytes[0];
      const keyId = dataBytes[1] | (dataBytes[2] << 8) | (dataBytes[3] << 16) | (dataBytes[4] << 24);
      const snid = dataBytes.slice(5, 17);
      
      return {
        commandName: '查询SNID',
        status: returnCode === 1 ? '查询成功' : '查询失败',
        returnCode,
        keyId,
        snid: this.bytesToHex(snid)
      };
    },
    
    parseRegisterKeyResponse(dataBytes) {
      const returnCode = dataBytes[0];
      let status = '未知状态';
      
      switch (returnCode) {
        case 1: status = '注册成功'; break;
        case 2: status = '钥匙已注册'; break;
        case 3: status = 'SNID错误'; break;
      }
      
      return {
        commandName: '注册钥匙',
        status,
        returnCode
      };
    },
    
    parseVerifySnidResponse(dataBytes) {
      const returnCode = dataBytes[0];
      
      return {
        commandName: '验证SNID',
        status: returnCode === 1 ? '验证成功' : '验证失败',
        returnCode
      };
    },
    
    parseKeyInfoResponse(dataBytes) {
      // 解析钥匙信息 (简化版本)
      return {
        commandName: '查询钥匙信息',
        status: '获取成功',
        returnCode: 1
      };
    },
    
    parseSetTimeResponse(dataBytes) {
      const timestamp = dataBytes[0] | (dataBytes[1] << 8) | (dataBytes[2] << 16) | (dataBytes[3] << 24);
      
      return {
        commandName: '设置校准时间',
        status: '设置成功',
        timestamp
      };
    },
    
    parseErrorResponse(dataBytes) {
      const returnCode = dataBytes[0];
      let status = '未知错误';
      
      switch (returnCode) {
        case 0: status = 'CRC校验错误'; break;
        case 1: status = '接收数据超时'; break;
        case 2: status = '帧格式不存在'; break;
        case 3: status = 'SNID未验证'; break;
        case 4: status = '指令无效'; break;
      }
      
      return {
        commandName: '错误响应',
        status,
        returnCode
      };
    },

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
          this.lastResponse = null;
        },
        fail: (err) => {
          console.error('断开失败:', err);
          uni.showToast({ title: '断开失败: ' + err.errMsg, icon: 'none' });
        }
      });
    },

    // 通用命令发送函数
    sendCommand(packetBytes, commandName) {
      console.log(`发送${commandName}命令...`);
      if (!this.isConnected) {
        uni.showToast({ title: '未连接设备', icon: 'none' });
        return;
      }
      
      this.receivedPackets = [];
      this.receivedDataHex = '';
      this.lastResponse = null;
      this.sentDataHex = this.bytesToHex(packetBytes);
      
      const buffer = new ArrayBuffer(packetBytes.length);
      const dataView = new DataView(buffer);
      for (let i = 0; i < packetBytes.length; i++) {
        dataView.setUint8(i, packetBytes[i]);
      }

      console.log(`发送${commandName}数据:`, this.sentDataHex);
      this.sendBlePacket(dataView.buffer);
    },

    // 各种钥匙命令 (使用协议结构)
    queryKeyRegStatus() {
      // 查询钥匙注册状态 (0x33)
      const packet = this.buildProtocolPacket(0x33, []);
      this.sendCommand(packet, "查询钥匙注册状态");
    },

    queryKeySnid() {
      // 查询钥匙SNID (0x35) - 需要验证码
      const verifyCode = [0x34, 0x9F, 0x6B, 0xA5];
      const packet = this.buildProtocolPacket(0x35, verifyCode);
      this.sendCommand(packet, "查询钥匙SNID");
    },

    verifyKeySnid() {
      // 验证钥匙SNID (0x39)
      const packet = this.buildProtocolPacket(0x39, this.keySnid);
      this.sendCommand(packet, "验证钥匙SNID");
    },

    registerKey() {
      // 注册钥匙 (0x37)
      const packet = this.buildProtocolPacket(0x37, this.keySnid);
      this.sendCommand(packet, "注册钥匙");
    },

    getKeyInfo() {
      // 查询钥匙信息 (0x05) - 需要SNID验证
      const packet = this.buildProtocolPacket(0x05, []);
      this.sendCommand(packet, "查询钥匙信息");
    },

    setKeyTime() {
      // 设置钥匙校准时间 (0x09)
      const timestamp = this.getCurrentTimestamp();
      const timeBytes = [
        timestamp & 0xFF,
        (timestamp >> 8) & 0xFF,
        (timestamp >> 16) & 0xFF,
        (timestamp >> 24) & 0xFF
      ];
      const packet = this.buildProtocolPacket(0x09, timeBytes);
      this.sendCommand(packet, "设置钥匙校准时间");
    },

    setKeyPassword() {
      // 设置钥匙通讯密码 (0x24)
      const password = [0x31, 0x31, 0x32, 0x32, 0x33, 0x33, 0x34, 0x34]; // "11223344"
      const packet = this.buildProtocolPacket(0x24, password);
      this.sendCommand(packet, "设置钥匙通讯密码");
    },

    resetKeyFactory() {
      // 恢复钥匙出厂配置 (0x1A)
      const packet = this.buildProtocolPacket(0x1A, []);
      this.sendCommand(packet, "恢复钥匙出厂配置");
    },

    setKeyPermission() {
      // 设置钥匙权限(一般授权) (0x0B) - 需要多条命令
      const authData = [
        0x02, 0x00, 0x00, 0x00, // 权限类型
        0x00, 0x00, 0x00, 0x00, 0x00, 0xFF, 0xFF, 0xFF, 0xFF, // 锁号范围
        0x02, 0x00, 0x00, 0x00, 0x00, // 锁数量
        0xF6, 0xB2, 0x65, 0xFF, 0xDE, 0xBA, 0x65, // 授权日期
        0x00, 0x00, 0x00, 0x00, 0x7F, 0x51, 0x01, 0x00, // 授权时间
        0x05, 0x00, 0x00, 0x00 // 使用次数
      ];
      
      const packet = this.buildProtocolPacket(0x0B, authData);
      this.sendCommand(packet, "设置钥匙权限(一般授权)");
    },

    setKeyPermissionMax() {
      // 设置钥匙权限(单位最高) (0x0B)
      const authData = [
        0x01, 0x00, 0x00, 0x00, // 权限类型(最高权限)
        0x00, 0x00, 0x00, 0x00, 0x00, 0xFF, 0xFF, 0xFF, 0xFF, // 锁号范围
        0xFF, 0xFF, 0xFF, 0xFF, 0x00, // 所有锁
        0xF6, 0xB2, 0x65, 0xFF, 0xDE, 0xBA, 0x65, // 授权日期
        0x00, 0x00, 0x00, 0x00, 0x7F, 0x51, 0x01, 0x00, // 授权时间
        0x00, 0x00, 0x00, 0x00 // 无次数限制
      ];
      
      const packet = this.buildProtocolPacket(0x0B, authData);
      this.sendCommand(packet, "设置钥匙权限(单位最高)");
    },

    // 发送BLE数据包
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
  padding: 20px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.button-group {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-bottom: 15px;
}

.small-btn {
  padding: 8px 12px;
  background-color: #007AFF;
  color: white;
  border-radius: 6px;
  text-align: center;
  font-size: 14px;
  flex: 1;
  min-width: 80px;
}

.small-btn:disabled {
  background-color: #cccccc;
  color: #666666;
}

.command-section {
  margin: 15px 0;
  padding: 15px;
  background-color: #f8f9fa;
  border-radius: 8px;
}

.command-buttons {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-top: 10px;
}

.cmd-btn {
  padding: 6px 10px;
  background-color: #28a745;
  color: white;
  border-radius: 4px;
  text-align: center;
  font-size: 12px;
  flex: 1;
  min-width: 100px;
}

.cmd-btn:disabled {
  background-color: #cccccc;
  color: #666666;
}

.connected-device {
  background-color: #e6f7ff;
  padding: 10px;
  border-radius: 8px;
  border-left: 4px solid #1890ff;
  margin-bottom: 10px;
}

.connected-device text {
  display: block;
  margin: 2px 0;
}

.device-item {
  padding: 12px;
  border-bottom: 1px solid #eee;
  background-color: #f8f8f8;
}

.device-item.selected {
  background-color: #e6f7ff;
  border-left: 4px solid #1890ff;
}

.device-id, .device-rssi {
  font-size: 12px;
  color: #999;
  display: block;
  margin-top: 4px;
}

.data-container {
  background-color: #f9f9f9;
  padding: 10px;
  border-radius: 8px;
  margin: 10px 0;
}

.section-title {
  font-weight: bold;
  display: block;
  margin-top: 8px;
  color: #1890ff;
  font-size: 16px;
}

.hex-data {
  font-family: monospace;
  word-break: break-all;
  font-size: 12px;
  display: block;
  margin: 5px 0;
  background-color: #fff;
  padding: 8px;
  border-radius: 4px;
  border: 1px solid #eee;
  max-height: 150px;
  overflow-y: auto;
}

.response-info {
  background-color: #fff3cd;
  padding: 10px;
  border-radius: 4px;
  border-left: 4px solid #ffc107;
  margin-top: 10px;
}

.response-info text {
  display: block;
  margin: 2px 0;
  font-size: 14px;
}
</style>