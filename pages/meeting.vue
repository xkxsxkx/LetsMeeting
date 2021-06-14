<template>
  <div class="meeting-container">
    <v-container>
      <v-row justify="space-around">
        <template v-for="(client) in clients">
          <preview
            v-if="client!==undefined"
            :key="client.userId"
            :client="client"
            :is-room-admin="clients[0].isRoomAdmin"
            @fullEvent="fullScreen"
            @kickEvent="kick"
            @viewEvent="changeView"
            @changeStreamEvent="changeStream"
          />
        </template>
      </v-row>
      <v-container>
        <v-main>
          <div style="text-align: center ;height: calc(100vh - 300px);width: 100%">
            <video ref="video_full" style="height:100%" muted autoplay playsinline />
          </div>
        </v-main>
        <v-badge>
          <v-btn value="show">
            <span v-show="isView">
              全体禁视
            </span>
            <span v-show="!isView">
              取消禁视
            </span>
          </v-btn>
          <v-btn value="voice">
            <span v-show="!isMuted">
              全体禁音
            </span>
            <span v-show="isMuted">
              取消禁音
            </span></v-btn>
          <v-btn value="message">
            <span v-show="!isBan">
              全体禁言
            </span>
            <span v-show="isBan">
              取消禁言
            </span>
          </v-btn>
        </v-badge>
      </v-container>
    </v-container>
    <v-dialog
      v-model="dialogFormVisible"
      title="请输入房间号和密码: "
      dark="true"
      max-width="600"
      overlay-opacity="100"
      persistent
      @close="closeView"
    >
      <v-form ref="romeForm">
        <v-card-title>
          <span class="text-h4">Welcome To LetsMeeting</span>
          <span class="text-h6">填写信息, 加入我们的会议吧</span>
        </v-card-title>
        <v-card-text>
          <v-text-field
            v-model="roomFromDate.nickname"
            label="昵称："
            prop="nickname"
            :rules="roomRules"
            :counter="10"
            required
          />
          <v-text-field
            v-model="roomFromDate.roomId"
            label="房间号："
            :rules="roomRules"
            :counter="10"
            required
          />
          <v-text-field
            v-model="roomFromDate.roomPw"
            label="密码:"
          />
        </v-card-text>
        <v-card-actions>
          <v-spacer />
          <v-btn
            color="darken-1"
            text
            @click="createOrEnterRoom('enter')"
          >
            加入
          </v-btn>
          <v-btn
            color="darken-1"
            text
            @click="createOrEnterRoom('create')"
          >
            创建
          </v-btn>
          <v-btn
            color="darken-1"
            text
            @click="dialogFormVisible = false"
          >
            取消
          </v-btn>
        </v-card-actions>
      </v-form>
    </v-dialog>
  </div>
</template>

<script>
import { adapter, io } from 'webrtc-adapter'
import Preview from './meeting-components/Preview.vue'

export default {
  name: 'Meeting',
  components: { Preview },
  data () {
    const valiRoomId = (rule, value, callback) => {
      if (value === '') {
        callback(new Error('请输入5-10位纯数字'))
      } else {
        const reg = /^\d{5,10}$/
        if (!reg.test(value)) {
          return callback(new Error('请输入5-10位纯数字'))
        }
        callback()
      }
    }
    return {
      meetingId: '0',
      dialogFormVisible: true,
      localWebsocket: undefined,
      wsUrl: undefined,
      receiveMsg: '',
      isInRoom: false,
      isBan: false,
      isView: true,
      isMuted: false,
      fullScreenId: '',
      clients: [{
        userId: '0',
        nickname: '未连接',
        roomId: '0',
        localStream: undefined,
        peerConnection: undefined,
        muted: false,
        view: true,
        chat: true,
        isSelf: true,
        isRoomAdmin: true,
        nowStream: 'screen'
      },
      {
        userId: '0',
        nickname: '未连接',
        roomId: '0',
        localStream: undefined,
        peerConnection: undefined,
        muted: false,
        view: true,
        chat: true,
        isSelf: false,
        isRoomAdmin: false,
        nowStream: 'screen'
      }
      ],
      roomFromDate: {
        nickname: '',
        roomId: '',
        roomPw: '',
        radio: '2'
      },
      roomFromRules: {
        roomId: [
          { validator: valiRoomId, trigger: 'blur' }
        ],
        roomPw: [
          { validator: valiRoomId, trigger: 'blur' }
        ]
      }
    }
  },
  computed: {
  },
  mounted () {
    this.dialogFormVisible = true
    this.roomFromDate.nickname = this.name
    try {
      // await this.initLocalWebsocket()
    } catch (e) {
      console.log('websoccket error: ' + e.message)
      this.$message.error('net connect error!')
      this.closeView()
    }
  },
  beforeDestroy () {
    console.log('will destory')
  },
  destroyed () {
    if (this.isInRoom) {
      const msg = new MessageModel(TYPE_COMMAND_KICK, this.roomFromDate.roomId,
        '', this.clients[0].userId)
      console.log(msg)
    }
  },
  methods: {
    async startv () {
      // camera
      console.log(adapter.browserDetails.browser)
      const audioStream1 = await navigator.mediaDevices.getUserMedia({ audio: true, video: true })
      console.log('摄像头设置')
      const c01 = {
        userId: '0',
        roomId: '0',
        nickname: '未连接',
        localStream: audioStream1,
        peerConnection: undefined,
        muted: false,
        view: true,
        chat: true,
        isSelf: true,
        isRoomAdmin: false,
        nowStream: 'screen'
      }
      this.$set(this.clients, 0, c01)
      console.log('本地流')
      console.log(this.clients[0].localStream)
      console.log('本地摄像头设置成功')
    },
    stopV () {
      this.clients[0].localStream.getTracks().forEach(function (track) {
        track.stop()
      })
    },
    addV () {
      navigator.mediaDevices.getDisplayMedia({ constraints }).then((stream) => {
        this.clients[0].localStream = stream
      }).catch((error) => {
        console.log(error)
      })
    },
    async changeStream (userId) {
      if (this.clients[0].nowStream === 'screen') {
        navigator.mediaDevices.getUserMedia({ video: true, audio: true })
          .then((mediaStream) => {
            this.stopV()
            console.log('切换为摄像头')
            this.clients[0].localStream = mediaStream
            console.log('本地摄像头设置成功')
          }).catch((e) => {
            console.log('本地摄像头设置失败 ' + e.message)
          })
        this.clients[0].nowStream = 'camera'
      } else {
        this.stopV()
        const mediaStream = await navigator.mediaDevices.getDisplayMedia(constraints)
        const audioStream = await navigator.mediaDevices.getUserMedia({ audio: true, video: false })
        console.log('切换为屏幕')
        this.clients[0].localStream = mediaStream
        this.clients[0].localStream.addTrack(audioStream.getAudioTracks()[0])
        console.log('本地播放器设置成功')
        this.clients[0].nowStream = 'screen'
      }
    },
    ban (userId) {
      console.log('ban:' + userId)
      if (userId === '') { // 全体禁言
        if (this.isBan) { // 恢复
          const msg = new MessageModel(TYPE_COMMAND_BAN, this.roomFromDate.roomId, 'false', '')
          this.wsSend(msg)
        } else {
          // 全体禁言
          const msg = new MessageModel(TYPE_COMMAND_BAN, this.roomFromDate.roomId, 'true', '')
          this.wsSend(msg)
        }
      } else if (this.clients[userId].chat) {
        const msg = new MessageModel(TYPE_COMMAND_BAN, this.roomFromDate.roomId, 'true', userId)
        this.wsSend(msg)
      } else { // 全员发送chat开启
        const msg = new MessageModel(TYPE_COMMAND_BAN, this.roomFromDate.roomId, 'false', userId)
        this.wsSend(msg)
      }
    },
    changeMicro (userId) {
      console.log('changeMicro:' + userId)
      if (userId === '') {
        if (this.isMuted) {
          const msg = new MessageModel(TYPE_COMMAND_MUTED, this.roomFromDate.roomId, 'false', '')
          this.wsSend(msg)
        } else {
          const msg = new MessageModel(TYPE_COMMAND_MUTED, this.roomFromDate.roomId, 'true', '')
          this.wsSend(msg)
        }
      } else if (userId === this.clients[0].userId) {
        if (this.clients[0].muted) {
          // 打开麦克风
          const msg = new MessageModel(TYPE_COMMAND_MUTED, this.roomFromDate.roomId, 'false', this.clients[0].userId)
          this.wsSend(msg)
        } else {
          // 关闭麦克风
          const msg = new MessageModel(TYPE_COMMAND_MUTED, this.roomFromDate.roomId, 'true', this.clients[0].userId)
          this.wsSend(msg)
        }
      } else if (this.clients[0].isRoomAdmin) { // 别人
        // 自己是管理员，就要彻底开关他的麦克风
        if (this.clients[Number(userId)].muted) {
          // 通知所有人打开此人麦克风
          const msg = new MessageModel(TYPE_COMMAND_MUTED, this.roomFromDate.roomId, 'false', userId)
          this.wsSend(msg)
        } else {
          // 通知所有人关闭此人麦克风
          const msg = new MessageModel(TYPE_COMMAND_MUTED, this.roomFromDate.roomId, 'true', userId)
          this.wsSend(msg)
        }
      } else if (this.clients[Number(userId)].muted) {
        this.clients[Number(userId)].muted = false
      } else {
        this.clients[Number(userId)].muted = true
      }
    },
    fullScreen (userId) {
      console.log('fullScreen:' + userId)
      if (userId === this.clients[0].userId) {
        this.$refs.video_full.srcObject = this.clients[0].localStream
        this.fullScreenId = '0'
      } else {
        this.$refs.video_full.srcObject = this.clients[userId].localStream
        this.fullScreenId = userId
      }
    },
    kick (userId) {
      console.log('kick:' + userId)
      const msg = new MessageModel(TYPE_COMMAND_KICK, this.roomFromDate.roomId, '', userId)
      this.wsSend(msg)
    },
    initLocalWebsocket () {
      console.log('初始化weosocket')
      // const response = await getUrl()
      // this.wsUrl = response.data
      // console.log('获取到wsurl:' + this.wsUrl)
      // this.localWebsocket = new WebSocket(this.wsUrl)
      this.localWebsocket = io()
      this.localWebsocket.onmessage = this.wseReceiveMessage
      this.localWebsocket.onopen = () => {
        console.log('localWebsocket打开')
      }
      this.localWebsocket.onerror = () => {
        console.log('localWebsocket错误')
        // 重连？
      }
      this.localWebsocket.onclose = (e) => {
        console.log('localWebsocket关闭' + e)
      }
    },
    createOrEnterRoom (method) { // 进入房间
      this.$refs.romeForm.validate((valid) => {
        if (valid) {
          let msg
          if (method === 'create') {
            msg = new MessageModel(TYPE_COMMAND_ROOM_CREATE, this.roomFromDate.roomId, this.roomFromDate.nickname, '', this.roomFromDate.roomPw, this.token)
            console.log('创建房间:' + JSON.stringify(msg))
            this.wsSend(msg)
          } else {
            console.log('加入房间:' + this.roomFromDate.roomId)
            msg = new MessageModel(TYPE_COMMAND_ROOM_ENTER, this.roomFromDate.roomId, this.roomFromDate.nickname, '', this.roomFromDate.roomPw, this.token)
            this.wsSend(msg)
          }
        } else {
          console.log('表单验证错误')
          return false
        }
      })
    },
    wsSend (data) { // 数据发送
      this.localWebsocket.send(JSON.stringify(data))
    }
  }
}

class MessageModel {
  constructor (command, roomId, message, userId, roomPw, token) {
    this.command = command
    this.userId = userId
    this.roomId = roomId
    this.message = message
    this.roomPw = roomPw
    this.token = token
  }
}

const TYPE_COMMAND_KICK = 'KICK'
const TYPE_COMMAND_MUTED = 'MUTED'
const TYPE_COMMAND_BAN = 'BAN'
const TYPE_COMMAND_ROOM_ENTER = 'enterRoom'
const TYPE_COMMAND_ROOM_CREATE = 'createRoom'
/*
const TYPE_COMMAND_VIEW = 'VIEW'
const TYPE_COMMAND_READY = 'ready'
const TYPE_COMMAND_OFFER = 'offer'
const TYPE_COMMAND_ANSWER = 'answer'
const TYPE_COMMAND_CANDIDATE = 'candidate'

const TYPE_COMMAND_ERROR = 'error'
const TYPE_COMMAND_SUCCESS = 'success'
const TYPE_COMMAND_CHAT = 'chat'

// const TYPE_COMMAND_SIGN = 'SIGN'

const iceServers = {
  'iceServers': [
    { url: 'stun:stun.ekiga.net' },
    { url: 'stun:stun.ideasip.com' }
  ]
}
const offerOptions = {
  iceRestart: true,
  offerToReceiveAudio: true,
  offerToReceiveVideo: true
}

*/
const constraints = {
  audio: true,
  video: true
}

</script>
