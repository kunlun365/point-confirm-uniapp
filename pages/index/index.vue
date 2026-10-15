<template>
  <view style="flex:1; position: relative;">
	<!-- 地图占满全屏 -->
	<map
		id="map"
		style="width:100%; height:100vh;"
		:longitude="centerLng"
		:latitude="centerLat"
		:markers="markers"
		@markertap="tapMarker"
		show-compass
		enable-3D
		enable-overlooking
		show-scale
	/>

    <!-- 顶部统计悬浮 -->
	<view class="top">
	  总数: {{ total }}   |
	  已确认: {{ confirmed }}   |
	  待确认: {{ pending }}   |
	  可疑点: {{ yes }}
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
      centerLng: 114.05   // 默认地图中心经度
    }
  },
  computed: {
    total() { return this.points.length },
    confirmed() { return this.points.filter(p => p.status !== 0).length },
    pending() { return this.points.filter(p => p.status === 0).length },
    yes() { return this.points.filter(p => p.status === 1).length }
  },
  onLoad() {
    this.connectWebSocket()
  },
  methods: {
    connectWebSocket() {
      this.ws = uni.connectSocket({
        url: 'ws://127.0.0.1:8080/ws/point', // 替换为你的局域网IP
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
          if(msg.length > 0){
            this.centerLat = msg[0].lat
            this.centerLng = msg[0].lng
          }
        } else if (msg.type === 'UPDATE') {
          // 单点更新
          const p = this.points.find(p => p.id === msg.pointId)
          if (p) p.status = msg.status
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
        latitude: p.lat,
        longitude: p.lng,
        iconPath:
          p.status === 0 ? '/static/red.png' :
          p.status === 1 ? '/static/yes.png' :
          '/static/no.png',
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
	  if(e.markerId !== undefined){
		console.log("marker点击事件触发", e.markerId)
	    marker = this.markers.find(m => m.id === e.markerId)
	  }

	  // H5/Web fallback: 用坐标匹配最近点兜底
	  const lat = e.detail?.latitude || e.latitude
	  const lng = e.detail?.longitude || e.longitude
	  if(!marker && lat && lng){
	    console.log("marker点击事件触发-最近点经度", lat, "纬度", lng)
	    let minD = Infinity
	    this.points.forEach(p => {
		  const d = Math.abs(p.lat - lat) + Math.abs(p.lng - lng)
		  if(d < minD){
		    minD = d
		    marker = { pointId: p.id }
		  }
	    })
	  }

	  if (!marker) return

      const id = marker.pointId           // 取真实点位 id
      const p = this.points.find(p => p.id === id)
	  if (!p) return

	  console.log("找到的点:", p)

      // 已确认直接提示
      if (p.status !== 0) {
        uni.showToast({
          title: p.status === 1 ? '√ 已确认' : '× 已排除',
          icon: 'none'
        })
        return
      }

      // uni.showModal({
      //   title: '确认',
      //   content: `点位 ${id}\n经纬度: ${p.lat.toFixed(6)}, ${p.lng.toFixed(6)}`,
      //   cancelText: '否',
      //   confirmText: '是',
      //   success: res => {
      //     this.sendConfirm(id, res.confirm ? 1 : 2)
      //   }
      // })
		uni.showActionSheet({
	      title: `${p.id}号点位(${p.lat.toFixed(6)},${p.lng.toFixed(6)})`,
		  itemList: ['可疑', '排除', '暂不处理'],
		  success: res => {
			if(res.tapIndex === 0){
			  this.sendConfirm(id, 1) // 是
			} else if(res.tapIndex === 1){
			  this.sendConfirm(id, 2) // 否
			} else {
			  uni.showToast({ title: '暂不处理', icon: 'none' })
			}
		  }
		})
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
    }
  }
}
</script>

<style>
.top {
  position: absolute;
  top: 10px;            /* 顶部距离 */
  left: 50%;
  transform: translateX(-50%); /* 水平居中 */
  padding: 6px 12px;
  background-color: rgba(255,255,255,0.7); /* 半透明白色 */
  border-radius: 8px;
  font-size: 16px;
  z-index: 999;
  box-shadow: 0 2px 4px rgba(0,0,0,0.2);
}
</style>
