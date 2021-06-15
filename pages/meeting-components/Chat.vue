<template>
  <div class="Chat-container">
    <v-textarea
      v-model="recText"
      outlined
      :rows="10"
      type="textarea"
      readonly
      placeholder="还没有人聊天..."
    />
    <v-text-field
      :append-icon="'mdi-message'"
      :append-outer-icon="'mdi-chat'"
      filled
      label="请输入内容"
      type="text"
      @click:append="$emit('noticeEvent',sendText)"
      @click:append-outer="$emit('chatEvent',sendText)"
    >
    </v-text-field>
  </div>
</template>

<script>

export default {
  name: 'Chat',
  props: {
    client: {
      type: Object,
      default: null
    },
    receiveMsg: {
      type: String,
      require: false,
      default: ''
    }
  },
  data () {
    return {
      recText: '',
      sendText: ''
    }
  },
  watch: {
    receiveMsg () {
      this.receiveMsgHandle()
    }
  },
  methods: {
    receiveMsgHandle () {
      this.recText += this.receiveMsg
    }
  }
}
</script>

<style lang="scss" scoped>

</style>
