<template>
  <div class="Preview-container">
    <div style="background: black">
      <div class="video-container">
        <video v-show="client.view" autoplay :muted.prop="client.muted || client.isSelf" :srcObject.prop="client.localStream" style="height: 100%;width: 100%" />
        <v-icon v-show="!client.view && !client.muted">
          mdi-account
        </v-icon>
        <v-icon v-show="!client.view && client.muted && client.chat">
          mdi-chat-dot-round clientStatus
        </v-icon>
        <v-icon v-show="!client.view && client.muted && !client.chat">
          mdi-circle-close clientStatus
        </v-icon>
      </div>
    </div>
    <div class="console-container">
      <span style="max-width: 130px">
        <v-icon v-if="client.isRoomAdmin">
          mdi-account
        </v-icon>
        <v-icon v-if="!client.isRoomAdmin">
          mdi-account-multiple
        </v-icon>
        <span>{{ client.nickname }}</span>
        <v-icon v-if="client.isSelf" @click="$emit('changeStreamEvent',client.userId)">
          mdi-refresh
        </v-icon>
      </span>
      <span style="float: right">
        <v-icon v-show="client.view" @click="$emit('fullEvent',client.userId)">
          mdi-fullscreen
        </v-icon>

        <v-icon v-show="client.view" @click="$emit('viewEvent',client.userId)">
          mdi-eye
        </v-icon>
        <v-icon v-show="!client.view" @click="$emit('viewEvent',client.userId)">
          mdi-eye-off
        </v-icon>

        <v-icon v-show="client.muted" @click="$emit('microEvent',client.userId)">
          mdi-microphone-off
        </v-icon>
        <v-icon v-show="!client.muted" @click="$emit('microEvent',client.userId)">
          mdi-microphone
        </v-icon>

        <v-icon v-show="!client.isSelf && isRoomAdmin && client.chat" @click="$emit('banEvent',client.userId)">
          mdi-chat
        </v-icon>
        <v-icon v-show="!client.isSelf && isRoomAdmin && !client.chat" class="el-icon-chat-round" @click="$emit('banEvent',client.userId)">
          mdi-chat-remove
        </v-icon>

        <v-icon v-if="!client.isSelf && isRoomAdmin" style="color: #ff4250;horiz-align: right" @click="$emit('kickEvent',client.userId)">
          mdi-alert-circle
        </v-icon>
      </span>
    </div>
  </div>
</template>

<script>

export default {
  name: 'Preview',
  props: {
    client: {
      type: Object,
      default: null
    },
    isRoomAdmin: {
      type: Boolean,
      default: false
    }
  }
}
</script>

<style lang="scss" scoped>
  .Preview-container{
    display:inline-block;
    width: 240px;
    height: 220px;
    margin-right: 3px;
  }
  .video-container{
    height: 170px;
    background: #939394;
    /*-webkit-mask-image: url("~@/assets/svg/people.svg");*/
    /*mask-image: url("~@/assets/svg/people.svg");*/
    /*mask-repeat: no-repeat;*/
    /*mask-position: center;*/
    text-align: center;
  }
  .console-container{
    height: 26px;
    background: rgb(102, 28, 28);
    line-height:30px;
  }
  .console-container i {
    font-size: 22px;
  }
  .console-container .svg-icon {
    font-size: 22px;
  }
  .clientStatus {
    font-size: 150px;
    line-height: 170px
  }

</style>
