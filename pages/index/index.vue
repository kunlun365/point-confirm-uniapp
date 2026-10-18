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
        <view class="legend-item">
          <image class="legend-icon" src="/static/unknown.png" mode="aspectFit" />
          <text>{{ pending }}</text>
        </view>
      </view>
    </view>

    <!-- 点位详情弹层 -->
    <view v-if="showPointPanel" class="panel-mask">
      <view class="panel" @click.stop>
        <view class="panel-title">
          {{ selectedPoint.name }}
        </view>

        <view class="field">
          <text class="field-label">名称</text>
          <text>{{ selectedPoint.name }}</text>
        </view>

        <!-- <view class="field">
          <text class="field-label">编码</text>
          <text>{{ selectedPoint.code }}</text>
        </view> -->

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

        <view class="field">
          <text class="field-label">评分</text>
          <view class="rating">
            <view class="rating-stars">
              <image v-for="i in 5" :key="'e' + i" class="star-img" src="/static/heart-empty.png" mode="aspectFit" />
              <view class="rating-fill" :style="{ width: ratingPercent }">
                <image v-for="i in 5" :key="'f' + i" class="star-img" src="/static/heart-full.png" mode="aspectFit" />
              </view>
            </view>
            <text class="rating-value">{{ selectedPoint.rating }}</text>
          </view>
        </view>

        <view class="field">
          <text class="field-label">地址</text>
          <text>{{ selectedPoint.city }}{{ selectedPoint.district }}{{ selectedPoint.address }}</text>
        </view>

        <!-- <view class="field">
          <text class="field-label">坐标</text>
          <text>{{ selectedPoint.lng }},{{ selectedPoint.lat }}</text>
        </view> -->

        <view class="field">
          <text class="field-label">电话</text>
          <text>{{ selectedPoint.tel }}</text>
        </view>

        <view class="field">
          <text class="field-label">营业时间</text>
          <text>{{ selectedPoint.opentime2 }}</text>
        </view>

        <view class="field">
          <text class="field-label">状态</text>
          <view class="status">
            <image class="status-icon" :src="statusIcon" mode="aspectFit" />
            <text>{{ statusText }}</text>
          </view>
        </view>

        <view class="panel-actions">
          <button v-if="selectedPoint.status === 0" type="primary" @click="confirmPoint(1)">
            种草
          </button>
          <button v-if="selectedPoint.status === 0" type="warn" @click="confirmPoint(2)">
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
      centerLat: 22.518968,   // 默认地图中心纬度
      centerLng: 113.972602,   // 默认地图中心经度
      showPointPanel: false,
      selectedPoint: null,
      heartbeatTimer: null, // 重连/心跳保活
      reconnectTimer: null,
      reconnectDelay: 2000,
      lastMsgTime: Date.now(),
      isPageVisible: true, // 避免僵尸死连接
      connectId: 0,
      sending: false // 防快速确认重复提交
    }
  },
  computed: {
    total() { return this.points.length },
    confirmed() { return this.points.filter(p => p.status !== 0).length },
    pending() { return this.points.filter(p => p.status === 0).length },
    yes() { return this.points.filter(p => p.status === 1).length },
    no() { return this.points.filter(p => p.status === 2).length },
    ratingPercent() {
      const r = Number(this.selectedPoint?.rating) || 0
      return Math.max(0, Math.min(1, r / 5)) * 100 + '%'
    },
    statusIcon() {
      const s = this.selectedPoint?.status
      if (s === 1) return '/static/yes.png'
      if (s === 2) return '/static/no.png'
      return '/static/unknown.png'
    },
    statusText() {
      const s = this.selectedPoint?.status
      if (s === 0) return '待探索'
      if (s === 1) return '种草'
      if (s === 2) return '避雷'
      return '未知'
    }
  },
  onLoad() {
    this.connectWebSocket()
  },
  onHide() {
    this.isPageVisible = false
    if (this.heartbeatTimer) clearInterval(this.heartbeatTimer)
    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
      this.reconnectTimer = null
    }
    if (this.ws && (this.ws.readyState === 0 || this.ws.readyState === 1)) {
      const old = this.ws
      this.ws = null
      this.connectId++ // 让旧连接所有回调失效
      try { old.close({}) } catch (e) { } // 不在前台运行时主动关闭，减少移动设备耗电，服务端同步释放连接
    }
  },
  onShow() {
    this.isPageVisible = true
    const state = this.ws && this.ws.readyState // 0连接中 1已连接 2关闭中 3已关闭
    if (state !== 0 && state !== 1) {
      this.connectWebSocket() // 切回前台重连
    }
  },
  onUnload() {
    this.isPageVisible = false
    if (this.heartbeatTimer) clearInterval(this.heartbeatTimer)
    if (this.reconnectTimer) { clearTimeout(this.reconnectTimer); this.reconnectTimer = null }
    if (this.ws) {
      const old = this.ws
      this.ws = null
      this.connectId++
      try { old.close({}) } catch (e) { }
    }
  },
  methods: {
    connectWebSocket() {
      // 清理旧定时器
      if (this.heartbeatTimer) clearInterval(this.heartbeatTimer)
      if (this.reconnectTimer) {
        clearTimeout(this.reconnectTimer);
        this.reconnectTimer = null
      }

      // 递增连接编号，用于判断"这条回调是否属于当前这次连接"
      const myId = ++this.connectId

      // 关掉旧连接并解除它的回调影响
      if (this.ws) {
        const old = this.ws
        this.ws = null
        try { old.close({}) } catch (e) { }
      }

      let wsUrl
      // #ifdef H5
      wsUrl = `${location.protocol === 'https:' ? 'wss' : 'ws'}://${location.host}/ws/point`
      // #endif
      // #ifndef H5
      wsUrl = 'wss://gis.kunlun365.us.ci/ws/point'
      // #endif
      const ws = uni.connectSocket({
        url: wsUrl,
        success: () => console.log("WebSocket连接中...")
      })
      this.ws = ws // this.wx是Proxy，和ws不等

      ws.onOpen(() => {
        if (myId !== this.connectId) return
        console.log("WebSocket已连接")
        // 启动心跳并重置退避
        this.reconnectDelay = 2000
        this.startHeartbeat()
      })

      ws.onMessage(res => {
        if (myId !== this.connectId) return // 旧连接的迟到消息直接丢弃
        this.lastMsgTime = Date.now() // 收到任何消息(含PONG、广播)都算连接活着
        let msg
        try { msg = JSON.parse(res.data) } catch (e) { return }
        if (Array.isArray(msg)) {
          // 全量初始化点位
          this.points = msg
          this.updateMarkers()
        } else if (msg.type === 'UPDATE') {
          // 单点更新
          const p = this.points.find(p => p.id === msg.pointId)
          if (p) p.status = msg.status

          // 如果详情弹层正显示这个点，同步更新状态。
          // 这样其他用户确认点位后，当前页面的地图、统计和详情状态都会更新。
          if (this.selectedPoint && this.selectedPoint.id === msg.pointId) {
            this.selectedPoint.status = Number(msg.status)
          }

          this.updateMarkers()
        }
      })

      this.ws.onError(e => {
        if (myId !== this.connectId) return // 旧连接异常直接丢弃，防止重连风暴
        console.error("WebSocket错误", e)
        this.scheduleReconnect()
      })

      this.ws.onClose(() => {
        if (myId !== this.connectId) return // 旧连接关闭，不触发重连
        console.log("WebSocket关闭，准备重连")
        if (this.heartbeatTimer) clearInterval(this.heartbeatTimer)
        this.scheduleReconnect()
      })
    },

    startHeartbeat() {
      this.lastMsgTime = Date.now()
      this.heartbeatTimer = setInterval(() => {
        if (this.ws && this.ws.readyState === 1) {
          this.ws.send({ data: JSON.stringify({ type: 'PING' }) })
        }
        // 70s 没收到任何消息，视为半开死链，主动关闭以触发onClose重连
        if (this.ws && Date.now() - this.lastMsgTime > 70000) {
          this.ws.close({})
        }
      }, 25000) // 25s 对移动电信运营商NAT足够保守
    },

    scheduleReconnect() {
      if (!this.isPageVisible) return // 切换后台不重连，等onShow
      if (this.reconnectTimer) return // 已排队，不重复
      if (this.ws && this.ws.readyState === 1) return // 已连上，不必重连

      this.reconnectTimer = setTimeout(() => {
        this.reconnectTimer = null
        this.connectWebSocket()
      }, this.reconnectDelay)

      // 2s→4s→8s→15s 封顶
      this.reconnectDelay = Math.min(this.reconnectDelay * 2, 15000)
    },

    updateMarkers() {
      // H5/Web markerId 不可靠，用 index，保存真实 id
      this.markers = this.points.map((p, index) => ({
        id: index,  // markerId 用前端连续整数
        latitude: Number(p.lat),
        longitude: Number(p.lng),
        iconPath: p.status === 0 ? '/static/red.png' : p.status === 1 ? '/static/yes.png' : '/static/no.png',
        width: 24,   // 可以考虑增大点击范围便于H5模拟器调试
        height: 24,
        pointId: p.id,  // 保存真实后端点位 id
        label: {
          content: p.id.toString(),
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
        let hit = null
        this.points.forEach(p => {
          const d = Math.abs(Number(p.lat) - lat) + Math.abs(Number(p.lng) - lng)
          if (d < minD) { minD = d; hit = p }
        })
        if (hit && minD < 0.01) marker = { pointId: hit.id } // 约 1km，按需调
      }

      if (!marker) return

      const id = marker.pointId           // 取真实点位 id
      const p = this.points.find(p => p.id === id)
      if (!p) return

      console.log("找到的点:", p)

      this.selectedPoint = { ...p }
      this.showPointPanel = true
    },

    confirmPoint(status) {
      if (!this.selectedPoint || this.sending) return
      this.sending = true

      const id = this.selectedPoint.id

      console.log("种草点位:", id)
      this.sendConfirm(id, status)

      // 暂时不直接修改本地点位状态，等待服务端广播 UPDATE 后再更新。
      this.closePointPanel()

      this.$nextTick(() => { this.sending = false })
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
  gap: 6px;
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

.rating {
  display: flex;
  align-items: center;
  gap: 6px;
  min-width: 0;
}

.rating-stars {
  position: relative;
  display: inline-flex;
  gap: 2px;
  line-height: 0;
}

.star-img {
  width: 16px;
  height: 16px;
  flex-shrink: 0;
  display: block;
}

.rating-fill {
  position: absolute;
  top: 0;
  left: 0;
  height: 100%;
  overflow: hidden;
  display: inline-flex;
  gap: 2px;
  white-space: nowrap;
}

.rating-value {
  flex-shrink: 0;
  color: #333333;
  font-size: 13px;
}

.status {
  display: flex;
  align-items: center;
  gap: 4px;
  min-width: 0;
}

.status-icon {
  width: 16px;
  height: 16px;
  flex-shrink: 0;
  display: block;
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
    max-width: none;
    width: calc(100vw - 160px);
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
