<template>
  <view style="flex:1; position: relative;">
    <!-- 地图占满全屏 -->
    <map id="map" style="width:100%; height:100vh;" :longitude="centerLng" :latitude="centerLat" :markers="markers"
      @markertap="tapMarker" show-compass enable-3D enable-overlooking show-scale />

    <!-- 顶部统计悬浮 -->
    <view class="top">
      <view>总数: {{ total }} | 已探索: {{ confirmed }} | 待探索: {{ pending }}</view>
      <view class="top-legend">
        <view class="legend-item">
          <image class="legend-icon" src="/static/yes.png" mode="aspectFit" />
          <text>{{ yes }}</text>
        </view>
        <view class="legend-item">
          <image class="legend-icon" src="/static/no.png" mode="aspectFit" />
          <text>{{ no }}</text>
        </view>
      </view>
    </view>

    <!-- 点位详情弹层 -->
    <view v-if="showPointPanel" class="panel-mask" @click="closePointPanel">
      <view class="panel" @click.stop>
        <view class="panel-title">
          公园详情
        </view>

        <view class="field">
          <text class="field-label">名称</text>
          <text>{{ selectedPoint.name }}</text>
        </view>

        <view class="field">
          <text class="field-label">序号</text>
          <text>{{ selectedPoint.serial_no }}</text>
        </view>

        <view class="field">
          <text class="field-label">编码</text>
          <text>{{ selectedPoint.code }}</text>
        </view>

        <view class="field">
          <text class="field-label">管理模式</text>
          <text>{{ selectedPoint.management_mode }}</text>
        </view>

        <view class="field">
          <text class="field-label">管理单位</text>
          <text>{{ selectedPoint.management_unit }}</text>
        </view>

        <view class="field">
          <text class="field-label">区划</text>
          <text>{{ selectedPoint.district }}</text>
        </view>

        <!-- <view class="field">
          <text class="field-label">省份</text>
          <text>{{ selectedPoint.province }}</text>
        </view>

        <view class="field">
          <text class="field-label">城市</text>
          <text>{{ selectedPoint.city }}</text>
        </view> -->

        <view class="field">
          <text class="field-label">地址</text>
          <text>{{ selectedPoint.formatted_address }}</text>
        </view>

        <view class="field">
          <text class="field-label">经度</text>
          <text>{{ selectedPoint.lng }}</text>
        </view>

        <view class="field">
          <text class="field-label">纬度</text>
          <text>{{ selectedPoint.lat }}</text>
        </view>

        <view class="field">
          <text class="field-label">确认状态</text>
          <text>
            {{ selectedPoint.status === 0 ? '? 待探索' : selectedPoint.status === 1 ? '√ 种草' : '× 避雷' }}
          </text>
        </view>

        <view class="panel-actions">
          <button v-if="selectedPoint.status === 0" type="warn" @click="confirmPoint(1)">
            种草
          </button>
          <button v-if="selectedPoint.status === 0" type="primary" @click="confirmPoint(2)">
            避雷
          </button>
          <button @click="closePointPanel">
            关闭
          </button>
        </view>
      </view>
    </view>
  </view>
</template>

<script>
export default {
  data() {
    return {
      ws: null,
      points: [], // 全部点位
      markers: [], // 地图 marker
      centerLat: 22.54,   // 默认地图中心纬度
      centerLng: 114.05,   // 默认地图中心经度
      showPointPanel: false,
      selectedPoint: null
    }
  },
  computed: {
    total() { return this.points.length },
    confirmed() { return this.points.filter(p => p.status !== 0).length },
    pending() { return this.points.filter(p => p.status === 0).length },
    yes() { return this.points.filter(p => p.status === 1).length },
    no() { return this.points.filter(p => p.status === 2).length }
  },
  onLoad() {
    this.connectWebSocket()
  },
  methods: {
    connectWebSocket() {
      this.ws = uni.connectSocket({
        url: 'ws://192.168.100.7:8080/ws/point',
        success: () => console.log("WebSocket连接中...")
      })

      this.ws.onOpen(() => {
        console.log("WebSocket已连接")
      })

      this.ws.onMessage(res => {
        const msg = JSON.parse(res.data)
        if (Array.isArray(msg)) {
          // 全量初始化点位
          this.points = msg
          this.updateMarkers()
          // 自动把地图中心设为第一个点
          if (msg.length > 0) {
            this.centerLat = 22.523896
            this.centerLng = 113.967667
          }
        } else if (msg.type === 'UPDATE') {
          // 单点更新
          const p = this.points.find(p => p.serial_no === msg.pointId)
          if (p) p.status = msg.status

          // 如果详情弹层正显示这个点，同步更新状态。
          // 这样其他用户确认点位后，当前页面的地图、统计和详情状态都会更新。
          if (this.selectedPoint && String(this.selectedPoint.serial_no) === String(msg.pointId)) {
            this.selectedPoint.status = Number(msg.status)
          }

          this.updateMarkers()
        }
      })

      this.ws.onError(e => {
        console.error("WebSocket错误", e)
      })

      this.ws.onClose(() => {
        console.log("WebSocket关闭")
      })
    },

    updateMarkers() {
      // H5/Web markerId 不可靠，用 index，保存真实 id
      this.markers = this.points.map((p, index) => ({
        id: index,  // markerId 用前端连续整数
        latitude: Number(p.lat),
        longitude: Number(p.lng),
        iconPath:
          p.status === 0 ? '/static/red.png' :
            p.status === 1 ? '/static/yes.png' :
              '/static/no.png',
        width: 24,   // 可以考虑增大点击范围便于H5模拟器调试
        height: 24,
        pointId: p.serial_no,  // 保存真实后端点位 id
        label: {
          content: p.serial_no.toString(),
          color: '#000000',
          fontSize: 14,
          anchorX: -0.2,
          anchorY: -20 // label在图标上方
        }
      }))
    },

    tapMarker(e) {
      let marker
      // H5/uni-app markerId 有可能 undefined
      if (e.markerId !== undefined) {
        console.log("marker点击事件触发", e.markerId)
        marker = this.markers.find(m => m.id === e.markerId)
      }

      // H5/Web fallback: 用坐标匹配最近点兜底
      const lat = e.detail?.latitude || e.latitude
      const lng = e.detail?.longitude || e.longitude
      if (!marker && lat && lng) {
        console.log("marker点击事件触发-最近点经度", lat, "纬度", lng)
        let minD = Infinity
        this.points.forEach(p => {
          const d = Math.abs(p.lat - lat) + Math.abs(p.lng - lng)
          if (d < minD) {
            minD = d
            marker = { pointId: p.serial_no }
          }
        })
      }

      if (!marker) return

      const id = marker.pointId           // 取真实点位 id
      const p = this.points.find(p => p.serial_no === id)
      if (!p) return

      console.log("找到的点:", p)

      this.selectedPoint = { ...p }
      this.showPointPanel = true
    },

    confirmPoint(status) {
      if (!this.selectedPoint) return

      const id = this.selectedPoint.serial_no

      console.log("种草点位:", id)
      this.sendConfirm(id, status)

      // 暂时不直接修改本地点位状态，等待服务端广播 UPDATE 后再更新。
      this.closePointPanel()
    },

    sendConfirm(id, status) {
      if (!this.ws || this.ws.readyState !== 1) {
        uni.showToast({ title: 'WebSocket未连接', icon: 'none' })
        return
      }

      const msg = {
        type: 'CONFIRM',
        pointId: id,
        status: status
      }
      this.ws.send({ data: JSON.stringify(msg) })
    },

    closePointPanel() {
      this.showPointPanel = false
      this.selectedPoint = null
    }
  }
}
</script>

<style>
.top {
  position: absolute;
  top: 10px;
  left: 50%;
  transform: translateX(-50%);
  padding: 5px 10px;
  background-color: rgba(255, 255, 255, 0.82);
  border-radius: 7px;
  font-size: 14px;
  line-height: 1.5;
  white-space: normal;
  text-align: center;
  z-index: 999;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.15);
}

.top-legend {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 18px;
  margin-top: 2px;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 3px;
}

.legend-icon {
  width: 1.3em;
  height: 1.2em;
  vertical-align: middle;
}

.panel-mask {
  position: fixed;
  inset: 0;
  z-index: 9999;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 12px;
  box-sizing: border-box;
  background: rgba(0, 0, 0, 0.35);
}

.panel {
  width: 560px;
  max-width: 100%;
  max-height: 75vh;
  overflow-y: auto;
  box-sizing: border-box;
  padding: 14px;
  border-radius: 10px;
  background: #ffffff;
  color: #333333;
}

.panel-title {
  margin-bottom: 8px;
  font-size: 18px;
  line-height: 1.5;
  font-weight: bold;
}

.field {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  padding: 5px 0;
  border-bottom: 1px solid #eeeeee;
  font-size: 14px;
  line-height: 1.5;
  overflow-wrap: anywhere;
}

.field-label {
  flex-shrink: 0;
  width: 76px;
  color: #777777;
}

.panel-actions {
  display: flex;
  gap: 8px;
  margin-top: 12px;
}

.panel-actions button {
  flex: 1;
  min-width: 0;
  height: 36px;
  line-height: 36px;
  margin: 0;
  padding: 0 6px;
  font-size: 14px;
  white-space: nowrap;
}

/* 手机窄屏适配 */
@media screen and (max-width: 480px) {
  .top {
    top: 12px;
    max-width: calc(100vw - 16px);
    padding: 4px 7px;
    font-size: 12px;
  }

  .panel {
    padding: 12px;
    border-radius: 9px;
  }

  .field {
    gap: 6px;
    padding: 4px 0;
    font-size: 13px;
  }

  .field-label {
    width: 70px;
  }

  .panel-actions {
    gap: 6px;
    margin-top: 10px;
  }

  .panel-actions button {
    padding: 0 3px;
    font-size: 13px;
  }
}
</style>
