<template>
  <div class="meeting-container">
    <v-main>
      <v-container>
        <v-row>
          <v-cols cols="12">
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
            <v-divider />
            <v-row>
              <div style="text-align: center ; height: calc(100vh - 300px); width: 100%">
                <video ref="video_full" style="height:100%" muted autoplay playsinline />
              </div>
            </v-row>
          </v-cols>
          <v-col cols="1" />
          <v-cols cols="4">
            <v-row>
              <Chat :receive-msg="receiveMsg" @chatEvent="sendChat" @noticeEvent="notice"/>
            </v-row>
            <v-row>
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
              <v-btn value="leave" @click="leave">
                离开
              </v-btn>
            </v-row>
          </v-cols>
        </v-row>
      </v-container>
    </v-main>
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
          <v-radio-group
            v-model="roomFromDate.radio"
            row
          >
            <v-radio label="摄像头" value="1" />
            <v-radio label="电脑屏幕" value="2" />
          </v-radio-group>
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
import io from 'socket.io-client'
import Preview from './meeting-components/Preview.vue'
import Chat from './meeting-components/Chat.vue'
// import socket from '~/plugins/socket.io.js'

export default {
  name: 'Meeting',
  components: { Preview, Chat },
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
      state: 'init',
      pc: null,
      dialogFormVisible: true,
      localWebsocket: undefined,
      wsUrl: undefined,
      receiveMsg: '',
      isInRoom: false,
      isBan: false,
      isView: true,
      isMuted: false,
      fullScreenId: '',
      roomId: '',
      offerdesc: null,
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
        isRoomAdmin: false,
        nowStream: 'screen'
      },
      {
        userId: '1',
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
      },
      pcConfig: {
        iceServers: [
          {
            urls: 'stun:stun.l.google.com:19302'
          },
          {
            urls: 'turn:207.246.74.80:3478',
            credential: 'maixiquan123',
            username: 'maixiquan'
          }]
      }
    }
  },
  computed: {
  },
  beforeMount () {
  },
  async mounted () {
    this.dialogFormVisible = true
    this.roomFromDate.nickname = this.name
    try {
      await this.startv()
      await this.initLocalWebsocket()
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
    isPc () {
      const userAgentInfo = navigator.userAgent
      const Agents = ['Android', 'iPhone', 'SymbianOS', 'Windows Phone', 'iPad', 'iPod']
      let flag = true

      for (let v = 0; v < Agents.length; v++) {
        if (userAgentInfo.indexOf(Agents[v]) > 0) {
          flag = false
          break
        }
      }
      return flag
    },
    is_android () {
      const u = navigator.userAgent
      // const app = navigator.appVersion
      const isAndroid = u.includes('Android') > -1 || u.includes('Linux') > -1 // g
      const isIOS = !!u.match(/\(i[^;]+;( U;)? CPU.+Mac OS X/) // ios终端
      if (isAndroid) {
        // 这个是安卓操作系统
        return true
      }
      if (isIOS) {
        // 这个是ios操作系统
        return false
      }
    },
    initLocalWebsocket () {
      console.log('初始化weosocket')
      // const response = await getUrl()
      // this.wsUrl = response.data
      // console.log('获取到wsurl:' + this.wsUrl)
      // this.localWebsocket = new WebSocket(this.wsUrl)
      this.localWebsocket = io.connect()
      this.localWebsocket.on('joined', (roomid, id) => {
        this.state = 'joined'
        this.createPeerConnection()
        this.bindTracks()
      })
      this.localWebsocket.on('otherjoin', (roomid) => {
        if (this.state === 'joined_unbind') {
          this.createPeerConnection()
          this.bindTracks()
        }
        this.state = 'joined_conn'
        this.call()
        console.log('recerive other_join message, state=', this.state)
      })
      this.localWebsocket.on('full', (roomid, id) => {
        console.log('receive full message', roomid, id)
        this.hangup()
        this.stopV()
        this.state = 'leaved'
        console.log('receive full message, state=', this.state)
        alert('the room is full!')
      })
      this.localWebsocket.on('leave', (roomid, id) => {
        console.log('receive leaved message', roomid, id)
        this.state = 'leaved'
        this.localWebsocket.disconnnect()
        console.log('receive leaved message, start=', this.state)
      })
      this.localWebsocket.on('bye', (roomid, id) => {
        this.state = 'joined_unbind'
        this.hangup()
        this.receiveMsg = ''
        console.log('receive bye message,, start=', this.state)
      })
      this.localWebsocket.on('disconnect', (socket) => {
        console.log('receive disconnect message!', this.roomid)
        if (!(this.state === 'leaved')) {
          this.hangup()
          this.stopV()
        }
        this.state = 'leaved'
      })
      this.localWebsocket.on('message', (roomid, data) => {
        console.log('receive message!', roomid, data)
        if (data === null || data === undefined) {
          console.error('the message is invalid!')
          return
        }
        if (Object.prototype.hasOwnProperty.call(data, 'type') && data.type === 'offer') {
          this.receiveMsg = data.sdp
          this.pc.setRemoteDescription(new RTCSessionDescription(data))

          // create answer
          this.pc.createAnswer()
            .then(this.getAnswer)
            .catch(this.handleAnswerError)
        } else if (Object.prototype.hasOwnProperty.call(data, 'type') && data.type === 'answer') {
          // answer.value = data.sdp
          this.pc.setRemoteDescription(new RTCSessionDescription(data))
        } else if (Object.prototype.hasOwnProperty.call(data, 'type') && data.type === 'candidate') {
          const candidate = new RTCIceCandidate({
            sdpMLineIndex: data.label,
            candidate: data.candidate
          })
          this.pc.addIceCandidate(candidate)
        } else {
          console.log('the message is invalid!', data)
        }
      })
    },
    getAnswer (desc) {
      this.pc.setLocalDescription(desc)
      this.sendMessage(this.roomid, desc)
    },
    handleAnswerError (err) {
      console.error('Fail to create answer:', err)
    },
    call () {
      if (this.state === 'joined_conn') {
        const offerOptions = {
          offerToRecieveAudio: 1,
          offerToRecieveVideo: 1
        }
        this.pc.createOffer(offerOptions).then(this.getOffer)
      }
    },
    getOffer (desc) {
      this.pc.setLocalDescription(desc)
      this.receiveMsg += desc.sdp
      this.offerdesc = desc
      this.sendMessage(this.roomid, this.offerdesc)
    },
    bindTracks () {
      if (this.pc === null || this.pc === undefined) {
        console.error('pc is null or undefine')
        return
      }
      if (this.clients[0].localStream === null || this.clients[0].localStream === undefined) {
        console.error('localstream is null or nudefined!')
        return
      }
      this.clients[0].localStream.getTracks().forEach(
        (track) => {
          this.pc.addTrack(track, this.clients[0].localStream)
        }
      )
    },
    createPeerConnection () {
      if (!this.pc) {
        this.pc = new RTCPeerConnection(this.pcConfig)
        this.pc.onicecandidate = (e) => {
          if (e.candidate) {
            this.sendMessage(this.roomid, {
              type: 'candidate',
              label: event.candidate.sdpMLineIndex,
              id: event.candidate.sdpMid,
              candidate: event.candidate.candidate
            })
          } else {
            console.log('this is the end candidate')
          }
        }
        this.pc.ontrack = this.getRemoteStream
      } else {
        console.warning('the pc have be created!')
      }
    },
    sendMessage (roomid, data) {
      console.log('send message to other end', roomid, data)
      if (!this.localWebsocket) {
        console.log('socket is null')
      } else {
        this.localWebsocket.emit('message', roomid, data)
      }
    },
    hangup () {
      if (this.pc) {
        this.offerdesc = null
        this.pc.close()
        this.pc = null
      }
    },
    async startv () {
      if (!navigator.mediaDevices ||
        !navigator.mediaDevices.getUserMedia) {
        console.error('the getUserMedia is not supported!')
      } else {
        let constraints
        if (this.radio === '1' && this.shareDesk()) {
          constraints = {
            video: false,
            audio: {
              echoCancellation: true,
              noiseSuppression: true,
              autoGainControl: true
            }
          }
        } else {
          constraints = {
            video: true,
            audio: {
              echoCancellation: true,
              noiseSuppression: true,
              autoGainControl: true
            }
          }
        }
        // camera
        const audioStream1 = await navigator.mediaDevices.getUserMedia(constraints)
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
      }
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
      this.connSignalServer()
    },
    connSignalServer () {
      this.startv()
      this.dialogFormVisible = false
    },
    wsSend (data) { // 数据发送
      this.localWebsocket.send(JSON.stringify(data))
    },
    leave () {
      if (this.localWebsocket) {
        this.localWebsocket.emit('leave', this.roomId)
      }
      this.hangup()
      this.stopV()
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
const TYPE_COMMAND_ROOM_ENTER = 'enterRoom'
const TYPE_COMMAND_ROOM_CREATE = 'createRoom'

// const TYPE_COMMAND_SIGN = 'SIGN'

/*
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
