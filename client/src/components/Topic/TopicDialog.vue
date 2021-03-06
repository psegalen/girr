<template>
  <aside id="topic-dialog"
    class="mdc-dialog"
    role="alertdialog"
    aria-labelledby="my-mdc-dialog-label"
    aria-describedby="my-mdc-dialog-description">
    <div class="mdc-dialog__surface">
      <header class="mdc-dialog__header">
        <h2 id="my-mdc-dialog-label" class="mdc-dialog__header__title">
          {{ title }}
        </h2>
      </header>
      <section id="my-mdc-dialog-description" class="mdc-dialog__body mdc-dialog__body--scrollable">
        <div class="mdc-text-field mdc-text-field--fullwidth" v-bind:class="{ 'mdc-text-field--upgraded' : topic.title }">
          <input type="text" id="title" class="mdc-text-field__input" v-model.lazy="topic.title">
          <label for="title" class="mdc-text-field__label" v-bind:class="{ 'mdc-text-field__label--float-above' : topic.title }">Title</label>
        </div>
        <div class="mdc-text-field mdc-text-field--fullwidth mdc-text-field--textarea">
          <textarea id="description" class="mdc-text-field__input" rows="8" placeholder="Description" v-model.lazy="topic.description">{{ topic.description }}</textarea>
        </div>
        <div class="mdc-grid-list">
          <ul class="mdc-grid-list__tiles">
            <li class="mdc-grid-tile" v-for="media in medias">
              <div class="mdc-grid-tile__primary">
                <img class="mdc-grid-tile__primary-content" :src="media._id && media.uri ? media.uri + '?height=256' : media.uri" />
                <i class="material-icons" v-on:click="deleteMedia(media)">cancel</i>
              </div>
              <span class="mdc-grid-tile__secondary" v-if="media.label">
                <span class="mdc-grid-tile__title">{{ media.label }}</span>
              </span>
            </li>
            <li class="mdc-grid-tile add-tile">
              <div class="mdc-grid-tile__primary">
                <input type="file" name="file" accept="image/*" class="input-file" v-on:change="fileChange($event.target.name, $event.target.files);" style="display: none;">
                <img class="mdc-grid-tile__primary-content" v-on:click="$event.currentTarget.parentNode.querySelector('input').click()"/>
              </div>
              <span class="mdc-grid-tile__secondary">
                <span class="mdc-grid-tile__title mdc-text-field">
                  <input type="text" id="uri" class="mdc-text-field__input" v-on:change="uriChange($event)">
                  <label for="uri" class="mdc-text-field__label">URL</label>
                </span>
              </span>
            </li>
          </ul>
        </div>
      </section>
      <footer class="mdc-dialog__footer">
        <div style="margin-right: auto;">
          <button type="button" class="mdc-button mdc-dialog__footer__button mdc-dialog__footer__button--delete" v-on:click="deleteTopic(topic)">
            <i class="material-icons mdc-button__icon">delete</i>
            <span>Delete</span>
          </button>
        </div>
        <button type="button" class="mdc-button mdc-dialog__footer__button mdc-dialog__footer__button--cancel" v-on:click="close"><i class="material-icons mdc-button__icon">clear</i><span>Cancel</span></button>
        <button type="button" class="mdc-button mdc-dialog__footer__button mdc-dialog__footer__button--accept mdc-button--raised" v-on:click="confirm"><i class="material-icons mdc-button__icon">check</i><span>Update</span></button>
      </footer>
    </div>
    <div class="mdc-dialog__backdrop"></div>
  </aside>
</template>

<script>
import Event from '../../utils/EventBus.js'
import { dialog, textField } from 'material-components-web'
import assign from 'object-assign'
import validUrl from 'valid-url'

export default {
  name: 'TopicDialog',
  data () {
    return {
      dialog: null,
      title: null,
      topic: {},
      medias: []
    }
  },
  mounted () {
    this.dialog = new dialog.MDCDialog(this.$el)
    textField.MDCTextField.attachTo(this.$el.querySelector('.mdc-text-field'))
    this.addTileTextfield = new textField.MDCTextField(this.$el.querySelector('.add-tile .mdc-text-field'))
    Event.$off('topicDialog.show').$on('topicDialog.show', this.show)
    Event.$off('topicDialog.close').$on('topicDialog.close', this.close)
  },
  methods: {
    show: function (topic, medias) {
      this.title = topic.title
      this.topic = assign({}, topic)
      this.medias = assign([], medias)
      this.$el.querySelector('textarea').value = this.topic.description ? this.topic.description : ''
      this.dialog.show()
    },
    close: function () {
      this.dialog.close()
    },
    confirm: function () {
      Event.$emit(`topics.${this.topic._id}.update`, this.topic, this.medias)
      this.updateTopic(this.topic)
    },
    fileChange: function (name, files) {
      if (files.length > 0) {
        let FR = new FileReader()
        FR.addEventListener('load', (e) => {
          this.medias.push({uri: e.target.result, label: files[0].name, file: files[0]})
        })
        FR.readAsDataURL(files[0])
      }
    },
    uriChange: function (event) {
      if (validUrl.isUri(event.target.value)) {
        this.medias.push({uri: event.target.value})
        event.target.value = ''
        this.addTileTextfield.valid = true
      } else {
        this.addTileTextfield.valid = false
      }
    },
    deleteMedia: function (media) {
      const index = this.medias.indexOf(this.medias.find(function (dialogMedia) {
        return dialogMedia === media
      }))
      if (index > -1) this.medias.splice(index, 1)
    },
    updateTopic: function (topic) {
      Event.$emit('progressbar.toggle', true)
      this.$http.put(`/api/programs/${this.$route.params.programId}/episodes/${this.$route.params.episodeId}/topics/${topic._id}`, topic).then(
        function (response) {
          Event.$emit('progressbar.toggle', false)
          Event.$emit('topic.updated', response.body)
          Event.$emit('snackbar.message', `Topic ${response.body.title} updated`)
          this.close()
        },
        function (response) {
          Event.$emit('progressbar.toggle', false)
          Event.$emit('http.error', response)
        }
      )
    },
    deleteTopic: function (topic) {
      Event.$emit('progressbar.toggle', true)
      this.$http.delete(`/api/programs/${this.$route.params.programId}/episodes/${this.$route.params.episodeId}/topics/${topic._id}`).then(
        function (response) {
          Event.$emit('progressbar.toggle', false)
          Event.$emit('topic.deleted', topic)
          Event.$emit('snackbar.message', `Topic ${topic.title} deleted`)
          this.close()
        },
        function (response) {
          Event.$emit('progressbar.toggle', false)
          Event.$emit('http.error', response)
        }
      )
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.mdc-text-field--textarea {
  margin-top: 16px;
}

.mdc-text-field--textarea textarea {
  padding-top: 16px;
}

.mdc-dialog__body--scrollable {
  max-height: calc(80vh - 56px - 52px); /* main - header - footer */
}

/* fix mdc-text-field--fullwidth padding */
.mdc-text-field--fullwidth:not(.mdc-text-field--textarea) .mdc-text-field__input {
  padding: 10px 0;
}

.mdc-grid-list {
  margin: 0 auto;
}

.mdc-grid-tile {
  --mdc-grid-list-tile-width: 192px;
  margin: 2px auto;
}

.mdc-grid-tile img {
  object-fit: contain;
}

.mdc-button__icon {
  vertical-align: text-bottom;
}

.mdc-grid-tile__primary i {
  position: absolute;
  top: 0;
  right: 0;
  z-index: 3;
  cursor: pointer;
}

.add-tile img {
  cursor: pointer;
  background: transparent;
  background-repeat: no-repeat;
  background-position: center center;
  background-image: url("\
  data:image/svg+xml;utf8, \
    <svg xmlns='http://www.w3.org/2000/svg' version='1.1' width='170px' height='50px'> \
      <text x='85' y='28' \
        style='text-anchor: middle' font-size='16'> \
        Click here to upload \
      </text> \
    </svg>\
  ");
}

.add-tile .mdc-grid-tile__secondary {
  padding: 0 16px;
  height: inherit;
}

.add-tile .mdc-grid-tile__secondary .mdc-text-field:not(.mdc-text-field--invalid) label {
  color: white;
}

.add-tile .mdc-grid-tile__secondary .mdc-text-field:not(.mdc-text-field--invalid) input {
  border-color: white;
}

.add-tile .mdc-grid-tile__secondary .mdc-text-field input {
  color: white;
}

.mdc-button .mdc-button__icon {
  line-height: 32px;
  vertical-align: initial
}

@media screen and (max-width: 480px) {
  .mdc-button span {
    display: none;
  }
}
@media screen and (min-width: 481px) {
  .mdc-button .mdc-button__icon {
    display: none;
  }
}

.mdc-dialog__footer__button--delete {
  color: red;
}
</style>
