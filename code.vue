<template>
  <view class="container">
    <view class="button-group">
      <button @click="startScan" :disabled="isScanning || isConnected" class="small-btn">开始扫描</button>
      <button @click="connectDevice" :disabled="!selectedDeviceId || isConnected || isConnecting" class="small-btn">连接设备</button>
      <button @click="disconnectDevice" :disabled="!isConnected" class="small-btn">断开连接</button>
    </view>
    
    <!-- 显示当前连接的设备 -->
    <view v-if="isConnected" class="connected-device">
      <text>已连接设备:</text>
      <text>{{ connectedDeviceName || selectedDeviceId }}</text>
      <text>信号强度: {{ connectedDeviceRSSI }}</text>
    </view>
    
    <text>设备状态：{{ deviceStatus }}</text>
    
    <!-- 命令按钮组 -->
    <view v-if="isConnected" class="command-section">
      <text class="section-title">钥匙指令</text>
      <view class="command-buttons">
        <button @click="queryKeyRegStatus" class="cmd-btn">1.查询注册钥匙状态</button>
        <button @click="queryKeySnid" class="cmd-btn">2.查询钥匙SNID</button>
		<button @click="registerKey" class="cmd-btn">3注册钥匙</button>
        <button @click="verifyKeySnid" class="cmd-btn">验证钥匙SNID</button>
		<button @click="getKeyInfo" class="cmd-btn">查询钥匙信息</button>
        <button @click="resetKeyFactory" class="cmd-btn">恢复出厂配置</button>
        <button @click="setKeyTime" class="cmd-btn">设置校准时间</button>
        
        <button @click="setKeyPassword" class="cmd-btn">设置通讯密码</button>
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
    </view>
	<!-- 数据解析结果展示区域 -->
	<view v-if="parsedResult" class="result-container">
	  <text class="section-title">数据解析结果:</text>
	  <view class="result-content">
	    <text class="result-item">命令类型: {{ parsedResult.commandName }}</text>
	    <text v-for="(item, index) in parsedResult.data" :key="index" class="result-item">
	      {{ item.label }}: {{ item.value }}
	    </text>
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
        write: '0000FFF6-0000-1000-8000-00805F9B34FB', // 写入特征值
        notify: '0000FFF7-0000-1000-8000-00805F9B34FB' // 通知特征值
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
      sentDataHex: '',      // 发送的完整数据(16进制)
      receivedDataHex: '',  // 接收的完整数据(16进制)
      receivedPackets: [],   // 存储接收到的所有数据包
      
      // 数据解析结果
      parsedResult: null,
      
      // 命令类型映射
      commandTypeMap: {
        '06': '查询钥匙信息',      // 0x06 回复
        '34': '查询钥匙注册状态',   // 0x34 回复  
        '1B': '恢复出厂配置',      // 0x1B 回复
        '3A': '验证钥匙SNID',     // 0x3A 回复
        '36': '查询钥匙SNID',     // 0x36 回复
        '0A': '设置校准时间',      // 0x0A 回复
        '38': '注册钥匙',         // 0x38 回复
        '25': '设置通讯密码',      // 0x25 回复
        '0C': '设置权限(离线)',    // 0x0C 回复
        '0E': '一般授权'          // 0x0E 回复
      },
	  snid: '68f9357e082b'
    };
  },
  mounted() {

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
            
            // 解析接收到的数据
            this.parseReceivedData(this.receivedDataHex);
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
 
    // 解析接收到的数据
    parseReceivedData(hexData) {
      if (!hexData) return;
      
      // 移除空格，转换为大写
      const cleanHex = hexData.replace(/\s/g, '').toUpperCase();
      console.log('接收到的数据:',cleanHex)  
    // 获取命令类型 
	 
	  const commandType = cleanHex.substr(24, 2);
	  
	  console.log('识别到命令类型:'+commandType);
	  
	  // 根据命令类型解析数据
	  this.parseCommandData(commandType, cleanHex);
        
       
    },
 
    // 根据命令类型解析数据
    parseCommandData(commandType, hexData) {
	  const commandName = this.commandTypeMap[commandType] || `未知命令(${commandType})`;
     
      switch (commandType) {
        case '34':
          this.parseKeyRegistrationStatus(hexData, commandName);
          break;
        case '06':
          this.parseKeyInfo(hexData, commandName);
          break;
        case '36':
          this.parseKeySnid(hexData, commandName);
          break;
        case '1B':
          this.parseResetFactory(hexData, commandName);
          break;
        case '3A':
          this.parseVerifySnid(hexData, commandName);
          break;
        case '0A':
          this.parseSetKeyTime(hexData, commandName);
          break;
	    case '38':
		  this.parseRegisterKey(hexData, commandName);
		  break;
        case '25':
          this.parseSetPassword(hexData, commandName);
          break;
        case '0C':
          this.parseSetPermission(hexData, commandName);
          break;
        case '0E':
          this.parseGeneralAuth(hexData, commandName);
          break;
        // 其他命令类型的解析可以在这里添加
        default:
          this.parsedResult = {
            commandName: commandName,
            data: [
              { label: '命令类型', value: commandType },
              { label: '状态', value: '暂未实现解析' }
            ]
          };
          break;
      }
    },

    // 解析查询钥匙SNID (36命令)
    parseKeySnid(hexData, commandName) {
      try {
        const cleanHex = hexData.replace(/[^a-fA-F0-9]/gi, '');
        
        // 判断偏移位置 18 的字节是否为 01
        const flagIndex = 18 * 2;
        if (cleanHex.length > flagIndex + 2) {
          const flagByte = cleanHex.substr(flagIndex, 2);
          console.log('flagByte:' + flagByte);
          
          if (flagByte === '01') {
            // 提取钥匙ID（从偏移位置 19 开始，4字节）
            const keyIdStartIndex = 19 * 2;
            const keyIdHex = cleanHex.substr(keyIdStartIndex, 8);
            
            // 提取 snid（从偏移位置 23 开始，共6字节）
            const snidStartIndex = 23 * 2;
            const snidHex = cleanHex.substr(snidStartIndex, 12);
            const formattedSnid = snidHex.match(/.{2}/g)?.join(' ') || '';
            
            // 更新实例的snid，去掉空格用于后续注册
            this.snid = snidHex;
            
            this.parsedResult = {
              commandName: commandName,
              data: [
                { label: '查询状态', value: '查询成功' },
                { label: '钥匙ID', value: keyIdHex },
                { label: 'SNID', value: formattedSnid }
              ]
            };
          } else {
            this.parsedResult = {
              commandName: commandName,
              data: [
                { label: '查询状态', value: '查询失败' },
                { label: '状态码', value: flagByte }
              ]
            };
          }
        }
      } catch (error) {
        console.error('解析查询钥匙SNID失败:', error);
        this.parsedResult = {
          commandName: commandName,
          data: [
            { label: '解析状态', value: '解析失败' },
            { label: '错误信息', value: error.message }
          ]
        };
      }
    },

    // 解析恢复出厂配置 (1B命令)
    parseResetFactory(hexData, commandName) {
      try {
        const cleanHex = hexData.replace(/[^a-fA-F0-9]/gi, '');
        
        // 从回复数据来看，恢复出厂配置的回复比较简单，主要确认操作成功
        this.parsedResult = {
          commandName: commandName,
          data: [
            { label: '操作状态', value: '恢复出厂配置成功' }
          ]
        };
      } catch (error) {
        console.error('解析恢复出厂配置失败:', error);
        this.parsedResult = {
          commandName: commandName,
          data: [
            { label: '解析状态', value: '解析失败' },
            { label: '错误信息', value: error.message }
          ]
        };
      }
    },

    // 解析验证钥匙SNID (3A命令)
    parseVerifySnid(hexData, commandName) {
      try {
        const cleanHex = hexData.replace(/[^a-fA-F0-9]/gi, '');
        
        // 获取验证结果（偏移位置18）
        const flagIndex = 18 * 2;
        if (cleanHex.length > flagIndex + 2) {
          const flagByte = cleanHex.substr(flagIndex, 2);
          let verifyStatus = '';
          
          switch (flagByte) {
            case '01':
              verifyStatus = '验证成功';
              break;
            case '00':
              verifyStatus = '验证失败';
              break;
            default:
              verifyStatus = '未知状态码: ' + flagByte;
          }
          
          this.parsedResult = {
            commandName: commandName,
            data: [
              { label: '验证状态', value: verifyStatus }
            ]
          };
        }
      } catch (error) {
        console.error('解析验证钥匙SNID失败:', error);
        this.parsedResult = {
          commandName: commandName,
          data: [
            { label: '解析状态', value: '解析失败' },
            { label: '错误信息', value: error.message }
          ]
        };
      }
    },

    // 解析设置校准时间 (0A命令)
    parseSetKeyTime(hexData, commandName) {
      try {
        const cleanHex = hexData.replace(/[^a-fA-F0-9]/gi, '');
        
        // 设置校准时间的回复包含时间信息
        // 时间从偏移18开始，4字节
        const timeStartIndex = 18 * 2;
        if (cleanHex.length > timeStartIndex + 8) {
          const timeHex = cleanHex.substr(timeStartIndex, 8);
          const timestamp = parseInt(timeHex, 16);
          const date = new Date(timestamp * 1000);
          
          this.parsedResult = {
            commandName: commandName,
            data: [
              { label: '操作状态', value: '时间设置成功' },
              { label: '设置时间戳', value: timeHex },
              { label: '时间', value: date.toLocaleString() }
            ]
          };
        } else {
          this.parsedResult = {
            commandName: commandName,
            data: [
              { label: '操作状态', value: '时间设置成功' }
            ]
          };
        }
      } catch (error) {
        console.error('解析设置校准时间失败:', error);
        this.parsedResult = {
          commandName: commandName,
          data: [
            { label: '解析状态', value: '解析失败' },
            { label: '错误信息', value: error.message }
          ]
        };
      }
    },

	// 注册钥匙(38命令)
	parseRegisterKey(hexData, commandName) {
	  // 清理 hex 字符串
	  const cleanHex = hexData.replace(/[^a-fA-F0-9]/gi, '');
	
	  // 获取 flagByte（偏移位置18）
	  const flagIndex = 18 * 2;
	  let registerStatus = '';
	
	  if (cleanHex.length > flagIndex + 2) {
	    const flagByte = cleanHex.substr(flagIndex, 2);
	    console.log('flagByte: ' + flagByte);
	
	    // 判断注册状态
	    switch (flagByte) {
	      case '01':
	        registerStatus = '注册成功';
	        break;
	      case '02':
	        registerStatus = '注册失败，钥匙已注册';
	        break;
	      case '03':
	        registerStatus = '注册失败，钥匙SNID错误';
	        break;
	      default:
	        registerStatus = '未知状态码: ' + flagByte;
	    }
	
	    // 构建解析结果
	    this.parsedResult = {
	      commandName: commandName,
	      data: [
	        { label: '注册状态', value: registerStatus }
	      ]
	    };
	  } else {
	    console.error('hexData 长度不足，无法读取 flagByte');
	  }
	},

    // 解析设置通讯密码 (25命令)
    parseSetPassword(hexData, commandName) {
      try {
        const cleanHex = hexData.replace(/[^a-fA-F0-9]/gi, '');
        
        // 设置密码的回复包含密码确认
        // 密码从偏移18开始，8字节
        const passwordStartIndex = 18 * 2;
        if (cleanHex.length > passwordStartIndex + 16) {
          const passwordHex = cleanHex.substr(passwordStartIndex, 16);
          
          this.parsedResult = {
            commandName: commandName,
            data: [
              { label: '操作状态', value: '密码设置成功' },
              { label: '设置密码', value: passwordHex }
            ]
          };
        } else {
          this.parsedResult = {
            commandName: commandName,
            data: [
              { label: '操作状态', value: '密码设置成功' }
            ]
          };
        }
      } catch (error) {
        console.error('解析设置通讯密码失败:', error);
        this.parsedResult = {
          commandName: commandName,
          data: [
            { label: '解析状态', value: '解析失败' },
            { label: '错误信息', value: error.message }
          ]
        };
      }
    },

    // 解析设置权限(离线) (0C命令)
    parseSetPermission(hexData, commandName) {
      try {
        const cleanHex = hexData.replace(/[^a-fA-F0-9]/gi, '');
        
        // 权限设置的回复比较简单，主要确认操作状态
        const flagIndex = 18 * 2;
        if (cleanHex.length > flagIndex + 2) {
          const flagByte = cleanHex.substr(flagIndex, 2);
          let status = '';
          
          switch (flagByte) {
            case '00':
              status = '权限设置成功';
              break;
            default:
              status = '权限设置状态: ' + flagByte;
          }
          
          this.parsedResult = {
            commandName: commandName,
            data: [
              { label: '操作状态', value: status }
            ]
          };
        } else {
          this.parsedResult = {
            commandName: commandName,
            data: [
              { label: '操作状态', value: '权限设置完成' }
            ]
          };
        }
      } catch (error) {
        console.error('解析设置权限失败:', error);
        this.parsedResult = {
          commandName: commandName,
          data: [
            { label: '解析状态', value: '解析失败' },
            { label: '错误信息', value: error.message }
          ]
        };
      }
    },

    // 解析一般授权 (0E命令)
    parseGeneralAuth(hexData, commandName) {
      try {
        const cleanHex = hexData.replace(/[^a-fA-F0-9]/gi, '');
        
        // 一般授权的回复包含授权结果
        const flagIndex = 18 * 2;
        if (cleanHex.length > flagIndex + 8) {
          const authId = cleanHex.substr(flagIndex, 8);
          
          this.parsedResult = {
            commandName: commandName,
            data: [
              { label: '操作状态', value: '授权设置成功' },
              { label: '授权ID', value: authId }
            ]
          };
        } else {
          this.parsedResult = {
            commandName: commandName,
            data: [
              { label: '操作状态', value: '授权设置完成' }
            ]
          };
        }
      } catch (error) {
        console.error('解析一般授权失败:', error);
        this.parsedResult = {
          commandName: commandName,
          data: [
            { label: '解析状态', value: '解析失败' },
            { label: '错误信息', value: error.message }
          ]
        };
      }
    },

    // 解析钥匙注册状态 (34命令)
    parseKeyRegistrationStatus(hexData, commandName) {
      try {
        // 1. 清理 hex 字符串（去掉空格、换行等非十六进制字符）
        const cleanHex = hexData.replace(/[^a-fA-F0-9]/gi, '');
        console.log(cleanHex); // 输出: 746878647A56312E302E30303400090000000140E20100C72FB18C778855AA
    
        // 2. 定义数据起始偏移量（单位：字节）
        const dataStart = 18; // 注册状态和钥匙ID位于偏移量 18 字节（字符索引 36）
    
        // 3. 检查长度是否足够（至少包含注册状态 + 钥匙ID）
        if (cleanHex.length < dataStart * 2 + 8) { // 8字节 = 1字节状态 + 4字节钥匙ID + 3字节填充
          throw new Error('数据长度不足，无法解析注册状态和钥匙ID');
        }
    
        // 4. 提取注册状态（1字节）
        const regStatusIndex = dataStart * 2; // 字符索引 36
        const regStatusHex = cleanHex.substr(regStatusIndex, 2); // 提取 "01"
        console.log('regStatusHex:', regStatusHex); // 输出: 01
    
        let regStatusText = '';
        if (regStatusHex === '01') {
          regStatusText = '已注册';
        } else if (regStatusHex === '02') {
          regStatusText = '未注册';
        } else {
          regStatusText = '未知状态';
        }
    
        // 5. 提取钥匙ID（4字节，小端序）
        const keyIDStartIndex = (dataStart + 1) * 2; // 字符索引 38
        const keyIDHexRaw = cleanHex.substr(keyIDStartIndex, 8); // 提取 "40E20100"
        console.log('keyIDHexRaw:', keyIDHexRaw); // 输出: 40E20100
    
        // 6. 小端序处理：反转每两个字符一组
        const keyIDBytes = [
          keyIDHexRaw.substr(6, 2), // 第4字节: 00
          keyIDHexRaw.substr(4, 2), // 第3字节: 01
          keyIDHexRaw.substr(2, 2), // 第2字节: E2
          keyIDHexRaw.substr(0, 2), // 第1字节: 40
        ];
        const keyIDDecimal = parseInt(keyIDBytes.join(''), 16); // 0001E240 → 123456
    
        // 7. 构建解析结果
        this.parsedResult = {
          commandName: commandName,
          data: [
            { label: '注册状态', value: `${regStatusText} (${regStatusHex})` },
            { label: '钥匙ID(16进制)', value: keyIDHexRaw },
            { label: '钥匙ID(10进制)', value: keyIDDecimal }
          ]
        };
    
        console.log('解析结果:', this.parsedResult);
    
      } catch (error) {
        console.error('解析钥匙注册状态失败:', error);
        this.parsedResult = {
          commandName: commandName,
          data: [
            { label: '解析状态', value: '解析失败' },
            { label: '错误信息', value: error.message }
          ]
        };
      }
    },

    // 解析查询钥匙SNID钥匙信息 (06命令) 
    parseKeyInfo(hexData, commandName) {
      // 清理 hex 字符串
      const cleanHex = hexData.replace(/[^a-fA-F0-9]/gi, '');
      
      // 判断偏移位置 18 的字节是否为 01
      const flagIndex = 18 * 2;
      if (cleanHex.length > flagIndex + 2) {
        const flagByte = cleanHex.substr(flagIndex, 2);
        console.log('flagByte:'+flagByte)
        if (flagByte === '01') {
          // 提取 snid（从偏移位置 20 开始，共6字节）
          const snidStartIndex = 20 * 2;
          this.snid = cleanHex.substr(snidStartIndex, 12);
    
          // 格式化成带空格的字符串（如：52 FF 6D 06 72 70）
          this.snid = this.snid.match(/.{2}/g)?.join(' ') || '';
        } else if (flagByte === '02') {
          // 如果是 02，则不设置 snid
          // this.snid = null;
        }
      }
    
      // 构建返回结果
      if (this.snid) {
        this.parsedResult = {
          commandName: commandName,
          data: [
            { label: 'snid', value: this.snid }
          ]
        };
      }
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
          this.parsedResult = null;
        },
        fail: (err) => {
          console.error('断开失败:', err);
          uni.showToast({ title: '断开失败: ' + err.errMsg, icon: 'none' });
        }
      });
    },
 
    // 通用命令发送函数
    sendCommand(hexData, commandName) {
      console.log(`发送${commandName}命令...`);
      if (!this.isConnected) {
        uni.showToast({ title: '未连接设备', icon: 'none' });
        return;
      }
      
      // 清空之前的解析结果
      this.parsedResult = null;
      this.receivedPackets = [];
      this.receivedDataHex = '';
      this.sentDataHex = hexData;
      
      const bytes = hexData.split(" ").map(b => parseInt(b, 16));
      const buffer = new ArrayBuffer(bytes.length);
      const dataView = new DataView(buffer);
      for (let i = 0; i < bytes.length; i++) {
        dataView.setUint8(i, bytes[i]);
      }
 
      console.log(`发送${commandName}数据:`, hexData);
      this.sendBlePacket(dataView.buffer);
    },
 
    // 各种钥匙命令
    getKeyInfo() {
      const hexData = "74 68 78 64 7A 56 31 2E 30 2E 30 30 05 00 04 00 00 00 56 5B BC 2D 77 88 55 AA";
      this.sendCommand(hexData, "查询钥匙信息");
    },
 
    queryKeyRegStatus() {
      const hexData = "74 68 78 64 7A 56 31 2E 30 2E 30 30 33 00 04 00 00 00 E6 BF 9F FF 77 88 55 AA";
      this.sendCommand(hexData, "查询钥匙注册状态");
    },
 
    resetKeyFactory() {
      const hexData = "74 68 78 64 7A 56 31 2E 30 2E 30 30 1A 00 04 00 00 00 18 EB 3C DF 77 88 55 AA";
      this.sendCommand(hexData, "恢复钥匙出厂配置");
    },
 
    verifyKeySnid() {
      const hexData = "74 68 78 64 7A 56 31 2E 30 2E 30 30 39 00 10 00 00 00 12 82 0A 8B 6F F7 E3 19 D0 96 B3 32 42 88 39 F8 77 88 55 AA";
      this.sendCommand(hexData, "验证钥匙SNID");
    },
 
    queryKeySnid() {
      const hexData = "74 68 78 64 7A 56 31 2E 30 2E 30 30 35 00 08 00 00 00 34 9F 6B A5 B5 11 8B 34 77 88 55 AA";
      this.sendCommand(hexData, "查询钥匙SNID");
    },
 
    setKeyTime() {
      const hexData = "74 68 78 64 7A 56 31 2E 30 2E 30 30 09 00 08 00 00 00 B8 A7 B3 65 0F 43 36 19 77 88 55 AA";
      this.sendCommand(hexData, "设置钥匙校准时间");
    },
 
    registerKey() {
      const hexData = "74 68 78 64 7A 56 31 2E 30 2E 30 30 37 00 10 00 00 00 "+ this.snid+" 50 55 17 44 03 67 E9 D2 E1 71 77 88 55 AA";
      this.sendCommand(hexData, "注册钥匙");
    },
 
    setKeyPassword() {
      const hexData = "74 68 78 64 7A 56 31 2E 30 2E 30 30 24 00 0C 00 00 00 31 31 32 32 33 33 34 34 0F F4 8A 5E 77 88 55 AA";
      this.sendCommand(hexData, "设置钥匙通讯密码");
    },
 
    setKeyPermission() {
      // 一般授权权限设置 - 发送多条命令
      const commands = [
        "74 68 78 64 7A 56 31 2E 30 2E 30 30 0B 00 29 00 00 00 02 00 00 00 00 00 00 00 00 FF FF FF FF 02 00 00 00 00 F6 B2 65 FF DE BA 65 00 00 00 00 7F 51 01 00 05 00 00 00 09 A7 4C 06 77 88 55 AA",
        "74 68 78 64 7A 56 31 2E 30 2E 30 30 0D 00 10 00 00 00 01 00 00 00 01 00 00 00 2B 00 00 00 7B D5 03 11 77 88 55 AA",
        "74 68 78 64 7A 56 31 2E 30 2E 30 30 0D 00 10 00 00 00 02 00 00 00 64 00 00 00 01 00 00 00 0E E6 19 31 77 88 55 AA"
      ];
      
      this.sendMultipleCommands(commands, "设置钥匙权限(一般授权)");
    },
 
    setKeyPermissionMax() {
      const hexData = "74 68 78 64 7A 56 31 2E 30 2E 30 30 0B 00 29 00 00 00 01 00 00 00 00 00 00 00 00 FF FF FF FF FF FF FF FF 00 F6 B2 65 FF DE BA 65 00 00 00 00 7F 51 01 00 00 00 00 00 8B 91 F6 EE 77 88 55 AA";
      this.sendCommand(hexData, "设置钥匙权限(单位最高)");
    },
 
    // 发送多条命令
    sendMultipleCommands(commands, commandName) {
      console.log(`发送${commandName}命令...`);
      if (!this.isConnected) {
        uni.showToast({ title: '未连接设备', icon: 'none' });
        return;
      }
      
      this.parsedResult = null;
      this.receivedPackets = [];
      this.receivedDataHex = '';
      this.sentDataHex = commands.join('\n');
      
      let currentIndex = 0;
      const sendNext = () => {
        if (currentIndex >= commands.length) {
          console.log(`${commandName}全部命令发送完成`);
          return;
        }
        
        const hexData = commands[currentIndex];
        const bytes = hexData.split(" ").map(b => parseInt(b, 16));
        const buffer = new ArrayBuffer(bytes.length);
        const dataView = new DataView(buffer);
        for (let i = 0; i < bytes.length; i++) {
          dataView.setUint8(i, bytes[i]);
        }
        
        console.log(`发送${commandName}第${currentIndex + 1}条命令:`, hexData);
        this.sendBlePacket(dataView.buffer);
        
        currentIndex++;
        // 延迟发送下一条命令
        setTimeout(sendNext, 2000);
      };
      
      sendNext();
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
 
 .result-container {
  background-color: #f0f8ff;
  padding: 15px;
  border-radius: 8px;
  border-left: 4px solid #28a745;
  margin: 10px 0;
 }
 
 .result-content {
  margin-top: 8px;
 }
 
 .result-item {
  display: block;
  margin: 4px 0;
  font-size: 14px;
  color: #333;
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
 </style>