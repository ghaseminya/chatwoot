<template>
  <div class="reply-box" :class="replyBoxClass">
    <reply-top-panel
      :mode="replyType"
      :set-reply-mode="setReplyMode"
      :is-message-length-reaching-threshold="isMessageLengthReachingThreshold"
      :characters-remaining="charactersRemaining"
    />
    <div class="reply-box__top">
      <canned-response
        v-if="showCannedResponsesList"
        v-on-clickaway="hideCannedResponse"
        data-dropdown-menu
        :on-keyenter="replaceText"
        :on-click="replaceText"
      />
      <emoji-input
        v-if="showEmojiPicker"
        v-on-clickaway="hideEmojiPicker"
        :on-click="emojiOnClick"
      />
      <resizable-text-area
        ref="messageInput"
        v-model="message"
        class="input"
        :placeholder="messagePlaceHolder"
        :min-height="4"
        @typing-off="onTypingOff"
        @typing-on="onTypingOn"
        @focus="onFocus"
        @blur="onBlur"
      />
    </div>
    <div v-if="hasAttachments" class="attachment-preview-box">
      <attachment-preview
        :attachments="attachedFiles"
        :remove-attachment="removeAttachment"
      />
    </div>
    <reply-bottom-panel
      :mode="replyType"
      :send-button-text="replyButtonLabel"
      :on-file-upload="onFileUpload"
      :show-file-upload="showFileUpload"
      :toggle-emoji-picker="toggleEmojiPicker"
      :show-emoji-picker="showEmojiPicker"
      :on-send="sendMessage"
      :is-send-disabled="isReplyButtonDisabled"
    />
  </div>
</template>

<script>
import { mapGetters } from 'vuex';
import { mixin as clickaway } from 'vue-clickaway';

import EmojiInput from 'shared/components/emoji/EmojiInput';
import CannedResponse from './CannedResponse';
import ResizableTextArea from 'shared/components/ResizableTextArea';
import AttachmentPreview from 'dashboard/components/widgets/AttachmentsPreview';
import ReplyTopPanel from 'dashboard/components/widgets/WootWriter/ReplyTopPanel';
import ReplyBottomPanel from 'dashboard/components/widgets/WootWriter/ReplyBottomPanel';
import { REPLY_EDITOR_MODES } from 'dashboard/components/widgets/WootWriter/constants';
import {
  isEscape,
  isEnter,
  hasPressedShift,
} from 'shared/helpers/KeyboardHelpers';
import { MESSAGE_MAX_LENGTH } from 'shared/helpers/MessageTypeHelper';
import inboxMixin from 'shared/mixins/inboxMixin';

export default {
  components: {
    EmojiInput,
    CannedResponse,
    ResizableTextArea,
    AttachmentPreview,
    ReplyTopPanel,
    ReplyBottomPanel,
  },
  mixins: [clickaway, inboxMixin],
  props: {
    inReplyTo: {
      type: [String, Number],
      default: '',
    },
  },
  data() {
    return {
      message: '',
      isFocused: false,
      showEmojiPicker: false,
      showCannedResponsesList: false,
      attachedFiles: [],
      isUploading: false,
      replyType: REPLY_EDITOR_MODES.REPLY,
    };
  },
  computed: {
    ...mapGetters({ currentChat: 'getSelectedChat' }),
    isPrivate() {
      if (this.currentChat.can_reply) {
        return this.replyType === REPLY_EDITOR_MODES.NOTE;
      }
      return true;
    },
    inboxId() {
      return this.currentChat.inbox_id;
    },
    inbox() {
      return this.$store.getters['inboxes/getInbox'](this.inboxId);
    },
    messagePlaceHolder() {
      return this.isPrivate
        ? this.$t('CONVERSATION.FOOTER.PRIVATE_MSG_INPUT')
        : this.$t('CONVERSATION.FOOTER.MSG_INPUT');
    },
    isMessageLengthReachingThreshold() {
      return this.message.length > this.maxLength - 50;
    },
    charactersRemaining() {
      return this.maxLength - this.message.length;
    },
    isReplyButtonDisabled() {
      const isMessageEmpty = !this.message.trim().replace(/\n/g, '').length;

      if (this.hasAttachments) return false;
      return (
        isMessageEmpty ||
        this.message.length === 0 ||
        this.message.length > this.maxLength
      );
    },
    conversationType() {
      const { additional_attributes: additionalAttributes } = this.currentChat;
      const type = additionalAttributes ? additionalAttributes.type : '';
      return type || '';
    },
    maxLength() {
      if (this.isPrivate) {
        return MESSAGE_MAX_LENGTH.GENERAL;
      }

      if (this.isAFacebookInbox) {
        return MESSAGE_MAX_LENGTH.FACEBOOK;
      }
      if (this.isATwilioSMSChannel) {
        return MESSAGE_MAX_LENGTH.TWILIO_SMS;
      }
      if (this.isATwitterInbox) {
        if (this.conversationType === 'tweet') {
          return MESSAGE_MAX_LENGTH.TWEET;
        }
      }
      return MESSAGE_MAX_LENGTH.GENERAL;
    },
    showFileUpload() {
      return (
        this.isAWebWidgetInbox ||
        this.isAFacebookInbox ||
        this.isATwilioWhatsappChannel ||
        this.isAPIInbox
      );
    },
    replyButtonLabel() {
      if (this.isPrivate) {
        return this.$t('CONVERSATION.REPLYBOX.CREATE');
      }
      if (this.conversationType === 'tweet') {
        return this.$t('CONVERSATION.REPLYBOX.TWEET');
      }
      return this.$t('CONVERSATION.REPLYBOX.SEND');
    },
    replyBoxClass() {
      return {
        'is-private': this.isPrivate,
        'is-focused': this.isFocused || this.hasAttachments,
      };
    },
    hasAttachments() {
      return this.attachedFiles.length;
    },
  },
  watch: {
    currentChat(conversation) {
      const { can_reply: canReply } = conversation;
      if (canReply) {
        this.replyType = REPLY_EDITOR_MODES.REPLY;
      } else {
        this.replyType = REPLY_EDITOR_MODES.NOTE;
      }
    },
    message(updatedMessage) {
      if (this.isPrivate) {
        return;
      }
      const isSlashCommand = updatedMessage[0] === '/';
      const hasNextWord = updatedMessage.includes(' ');
      const isShortCodeActive = isSlashCommand && !hasNextWord;
      if (isShortCodeActive) {
        this.showCannedResponsesList = true;
        if (updatedMessage.length > 1) {
          const searchKey = updatedMessage.substr(1, updatedMessage.length);
          this.$store.dispatch('getCannedResponse', { searchKey });
        } else {
          this.$store.dispatch('getCannedResponse');
        }
      } else {
        this.showCannedResponsesList = false;
      }
    },
  },
  mounted() {
    document.addEventListener('keydown', this.handleKeyEvents);
  },
  destroyed() {
    document.removeEventListener('keydown', this.handleKeyEvents);
  },
  methods: {
    handleKeyEvents(e) {
      if (isEscape(e)) {
        this.hideEmojiPicker();
        this.hideCannedResponse();
      } else if (isEnter(e)) {
        if (!hasPressedShift(e)) {
          e.preventDefault();
          this.sendMessage();
        }
      }
    },
    async sendMessage() {
      if (this.isReplyButtonDisabled) {
        return;
      }
      if (!this.showCannedResponsesList) {
        const newMessage = this.message;
        const messagePayload = this.getMessagePayload(newMessage);
        this.clearMessage();
        try {
          await this.$store.dispatch('sendMessage', messagePayload);
          this.$emit('scrollToMessage');
        } catch (error) {
          // Error
        }
        this.hideEmojiPicker();
      }
    },
    replaceText(message) {
      setTimeout(() => {
        this.message = message;
      }, 100);
    },
    setReplyMode(mode = REPLY_EDITOR_MODES.REPLY) {
      const { can_reply: canReply } = this.currentChat;

      if (canReply) this.replyType = mode;
      this.$refs.messageInput.focus();
    },
    emojiOnClick(emoji) {
      this.message = `${this.message}${emoji} `;
    },
    clearMessage() {
      this.message = '';
      this.attachedFiles = [];
    },
    toggleEmojiPicker() {
      this.showEmojiPicker = !this.showEmojiPicker;
    },
    hideEmojiPicker() {
      if (this.showEmojiPicker) {
        this.toggleEmojiPicker();
      }
    },
    hideCannedResponse() {
      this.showCannedResponsesList = false;
    },
    onTypingOn() {
      this.toggleTyping('on');
    },
    onTypingOff() {
      this.toggleTyping('off');
    },
    onBlur() {
      this.isFocused = false;
    },
    onFocus() {
      this.isFocused = true;
    },
    toggleTyping(status) {
      if (this.isAWebWidgetInbox && !this.isPrivate) {
        const conversationId = this.currentChat.id;
        this.$store.dispatch('conversationTypingStatus/toggleTyping', {
          status,
          conversationId,
        });
      }
    },
    onFileUpload(file) {
      this.attachedFiles = [];
      if (!file) {
        return;
      }
      const reader = new FileReader();
      reader.readAsDataURL(file.file);

      reader.onloadend = () => {
        this.attachedFiles.push({
          currentChatId: this.currentChat.id,
          resource: file,
          isPrivate: this.isPrivate,
          thumb: reader.result,
        });
      };
    },
    removeAttachment(itemIndex) {
      this.attachedFiles = this.attachedFiles.filter(
        (item, index) => itemIndex !== index
      );
    },
    getMessagePayload(message) {
      const [attachment] = this.attachedFiles;
      const messagePayload = {
        conversationId: this.currentChat.id,
        message,
        private: this.isPrivate,
      };

      if (this.inReplyTo) {
        messagePayload.contentAttributes = { in_reply_to: this.inReplyTo };
      }

      if (attachment) {
        messagePayload.file = attachment.resource.file;
      }

      return messagePayload;
    },
  },
};
</script>

<style lang="scss" scoped>
.send-button {
  margin-bottom: 0;
}

.attachment-preview-box {
  padding: 0 var(--space-normal);
  background: transparent;
}

.reply-box {
  border-top: 1px solid var(--color-border);
  background: white;

  &.is-private {
    background: var(--y-50);
  }
}
.send-button {
  margin-bottom: 0;
}

.reply-box__top {
  padding: 0 var(--space-normal);
  border-top: 1px solid var(--color-border);
  margin-top: -1px;
}

.emoji-dialog {
  top: unset;
  bottom: 12px;
  left: -320px;
  right: unset;

  &::before {
    right: -16px;
    bottom: 10px;
    transform: rotate(270deg);
    filter: drop-shadow(0px 4px 4px rgba(0, 0, 0, 0.08));
  }
}
</style>
