# Bluetooth-applet
This is an example of Bluetooth applet connected to Bluetooth, read and write Bluetooth characteristic value.
# 小程序的实现步骤
### 第一步：搜索周围蓝牙设备并展示在蓝牙列表之中
```
wepy.startBluetoothDevicesDiscovery({services: ['1023']}).then((res) => {
// 为了适配低性能手机，做了一个2.5秒的延时
  setTimeout(() => {
    wepy.getBluetoothDevices().then((res) => {
      _this.flag = true
      let arr = []
      if (res.devices.length > 0) {
        for (let i = 0; i < res.devices.length; i++) {
          res.devices[i].active = false
          if (res.devices[i].RSSI > -60) {
            arr.push(res.devices[i])
          }
        }
        if (arr.length > 0) {
          _this.deviceList = arr
        } else {
          _this.$invoke('toast', 'show', {
            isShowImg: false,
            toastTxt: '未搜索到设备，请重试'
          })
        }
      } else {
        this.$invoke('toast', 'show', {
          isShowImg: false,
          toastTxt: '未搜索到设备，请重试'
        })
      }
      wepy.stopBluetoothDevicesDiscovery()
      _this.$apply()
    }).catch((e) => {
      this.$invoke('toast', 'show', {
        isShowImg: false,
        toastTxt: '搜索失败，请重试'
      })
      _this.flag = true
    })
  }, 2500)
}).catch((e) => {
  if (e.errCode.toString().indexOf('10001') > -1) {
    _this.flag = true
    _this.$apply()
    this.$invoke('toast', 'show', {
      isShowImg: false,
      toastTxt: '请检查是否打开蓝牙'
    })
  }
})
  ```
### 第二步：点击蓝牙列表连接对应的蓝牙模块
```
 // 创建连接
wepy.createBLEConnection({deviceId: _this.deviceId}).then((res) => {
  // 获取连接的设备的UUID
  wepy.getBLEDeviceServices({deviceId: _this.deviceId}).then((res) => {
    _this.serviceId = res.services[0].uuid
    // 获取连接设备的特征值
    wepy.getBLEDeviceCharacteristics({deviceId: _this.deviceId, serviceId: _this.serviceId}).then((res) => {
      for (let i = 0; i < res.characteristics.length; i++) {
        if (res.characteristics[i].properties.read && res.characteristics[i].properties.write) {
          _this.characteristicId.push(res.characteristics[i].uuid)
        }
      }
      _this.$apply()
      _this.getValue(_this.characteristicId, (number) => {
        _this.$invoke('toast', 'hide')
        _this.$invoke('alert', 'show', {
          ifinput: true,
          title: '设备号',
          status: '已连接',
          number: number,
          btntype: '提交'
        })
      })
    }).catch((e) => {
    })
  }).catch((e) => {
  })
}).catch((e) => {
  if (e.errCode.toString().indexOf('10003') > -1) {
    this.getError('连接失败请重试')
  }
})
```
### 第三步：获取连接蓝牙的特征值并转化成10进制的15位设备号展示出来
```
wepy.readBLECharacteristicValue({deviceId: _this.deviceId, serviceId: _this.serviceId, characteristicId: value}).then((res) => {
  wepy.onBLECharacteristicValueChange(function(characteristic) {
    let uuid = _this.ab2hex(characteristic.value)
    _this.dataList.push(uuid)
    _this.$apply()
    cb()
  })
}).catch((e) => {
  this.getError('读取数据失败，请重试')
})
```
### 第四步：输入新的设备号，转化成arraybuffer循环写入
```
wepy.writeBLECharacteristicValue({deviceId: this.deviceId, serviceId: this.serviceId, characteristicId: value1, value: this.toBuffer(value2)}).then((res) => {
  setTimeout(() => {
    cb()
  }, 250)
}).catch((e) => {
  cb()
  // this.getError('写入数据失败，请重试')
})
```
# 注意事项
### 16进制和arraybuffer的互相转化
```
// ArrayBuffer转16进度字符串
ab2hex(buffer) {
  let hexArr = Array.prototype.map.call(
    new Uint8Array(buffer),
    function(bit) {
      return ('00' + bit.toString(16)).slice(-2)
    }
  )
  return hexArr.join('')
}

// 16进度字符串转ArrayBuffer
toBuffer (str) {
  var typedArray = new Uint8Array(str.match(/[\da-f]{2}/gi).map(function (h) {
    return parseInt(h, 16)
  }))
  var buf = typedArray.buffer
  return buf
}
```
### 常见的几个错误码解析
###### 1. 10001
  未打开手机蓝牙功能
###### 2. 10003
  连接蓝牙失败
###### 3. 10007
  当前特征值不支持此操作
###### 4. 10008
  其他错误，如频繁调用读写api，写入的特征值与原来特征值的位数不相等，格式错误等
