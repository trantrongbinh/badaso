<template>
  <vs-col :vs-lg="size" vs-xs="12" class="badaso-upload-image-multiple__container">
    <vs-input :label="label" :placeholder="placeholder" @click="showOverlay" v-on:keyup.space="showOverlay" readonly v-model="imagesName" icon="attach_file" icon-after="true" />
    <vs-row>
      <vs-col vs-lg="4" vs-sm="12" v-for="(imageData, index) in value" :key="index">
        <div class="badaso-upload-image-multiple__preview">
          <vs-button class="badaso-upload-image-multiple__remove-button" color="danger" icon="close" @click="deleteFilePicked(imageData)" />
          <img :src="imageData" class="badaso-upload-image-multiple__preview-image" />
        </div>
      </vs-col>
    </vs-row>
    <div class="badaso-upload-image-multiple__popup-dialog" tabindex="0" v-if="show">
      <div class="badaso-upload-image-multiple__popup-container">
        <div class="badaso-upload-image-multiple__popup--top-bar">
          <h3>{{ $t('fileManager.title') }}</h3>
          <vs-spacer />
          <vs-button color="danger" type="relief" class="badaso-upload-image-multiple__popup-button--delete" v-if="getSelected !== 'url' &&   isImageSelected" @click="openDialog">
            <vs-icon icon="delete"></vs-icon>
          </vs-button>
        </div>
        <ul class="badaso-upload-image-multiple__popup--left-bar">
          <li :class="[getSelected === 'private'  ? 'active' : '' ]" @click="selected = 'private'" v-if="privateOnly || !privateOnly && !sharesOnly">Private</li>
          <li :class="[getSelected === 'shares' ? 'active' : '' ]" @click="selected = 'shares'" v-if="sharesOnly || !sharesOnly && !privateOnly">Shares</li>
          <li :class="[getSelected === 'url' ? 'active' : '', $store.state.badaso.isOnline ? '' : 'badaso-upload-image__menu--disabled' ]" @click="selected = 'url'">Insert by URL</li>
        </ul>
        <div class="badaso-upload-image-multiple__popup--right-bar" v-if="getSelected !== 'url'">
          <div class="badaso-upload-image-multiple__popup-add-image" @click="pickFile">
            <vs-icon icon="add" color="#06bbd3" size="75px"></vs-icon>
          </div>
          <img :class="[activeImage.includes(index) ? 'active' : '', 'badaso-upload-image-multiple__popup-image']" :src="item.thumbUrl" v-for="(item, index) in images.items" :key="index" @click="selectImage(index)">
        </div>
        <div v-if="getSelected === 'url'" class="badaso-upload-image-multiple__popup--right-bar badaso-upload-image-multiple__popup--url-bar">
          <vs-input v-if="getSelected === 'url'" label="Paste an image URL here" placeholder="URL" v-model="inputByUrl" description-text="If your URL is correct, you'll see an image preview here. Large images may take a few minutes to appear. Only accept PNG and JPEG." @input="inputByUrl === '' ? dirty = false : dirty = true" ></vs-input>
          <p v-if="!isValidImage && getSelected === 'url' && dirty" class="is-error">Only valid image (PNG and JPEG) is accepted</p>
          <img accept="image/png" v-if="getSelected === 'url'" :src="inputByUrl" alt="" @load="isValidImage = true" @error="isValidImage = false" class="badaso-upload-image-multiple__preview--small">
        </div>
        <div class="badaso-upload-image-multiple__popup--bottom-bar">
          <div class="badaso-upload-image-multiple__popup-button--footer">
            <vs-button color="primary" type="relief" @click="emitInput" :disabled="isSubmitDisable">
              {{ $t('button.submit') }}
            </vs-button>
            <vs-button color="danger" type="relief" @click="closeOverlay">
              {{ $t('button.close') }}
            </vs-button>
          </div>
        </div>
      </div>
    </div>
    <input type="file" class="badaso-upload-image__input--hidden" ref="image" accept="image/*" @change="onFilePicked" multiple />
    <div v-if="additionalInfo" v-html="additionalInfo"></div>
    <div v-if="alert">
      <div v-if="$helper.isArray(alert)">
        <span class="badaso-upload-image-multiple__input--error" v-for="(info, index) in alert" :key="index">
          {{ info }}
        </span>
      </div>
      <div v-else>
        <span class="badaso-upload-image-multiple__input--error" v-html="alert"></span>
      </div>
    </div>
  </vs-col>
</template>

<script>
import * as _ from "lodash"
export default {
  name: "BadasoUploadImageMultiple",
  props: {
    size: {
      type: String,
      default: "12",
    },
    label: {
      type: String,
      default: "",
    },
    placeholder: {
      type: String,
      default: "Upload Image Multiple",
    },
    value: {
      type: Array,
      default: () => {
        return [];
      },
    },
    additionalInfo: {
      type: String,
      default: "",
    },
    alert: {
      type: String | Array,
      default: "",
    },
    sharesOnly: {
      type: Boolean,
    },
    privateOnly: {
      type: Boolean,
    },
  },
  watch: {
    selected: {
      handler(val) {
        this.getImages()
        this.isImageSelected = false
      }
    },
    value: {
      handler(val) {
        this.imageUrl = val
      }
    }
  },
  data() {
    return {
      imagesName: "",
      dialog: false,
      activeImage: [],
      show: false,
      selected: 'private',
      images: {
        display: "",
        items: [],
        paginator: {},
      },
      files: [],
      imageUrl: [],
      inputByUrl: "",
      isValidImage: false,
      isImageSelected: false,
      dirty: false,
    };
  },
  mounted() {
    if (this.sharesOnly) {
      this.selected = "shares"
    }
    
    this.imageUrl = this.value
    this.imagesName = this.imageUrl.join(', ')
  },
  computed: {
    getSelected() {
      this.activeImage = []
      return this.selected
    },
    getSelectedFolder() {
      if (this.getSelected === 'shares') return '/shares'
      else if (this.getSelected === 'url') return
      else return this.getUserFolder
    },
    getUserFolder() {
      return '/' + this.$store.state.badaso.user.id
    },
    isSubmitDisable() {
      if (!this.isImageSelected && this.selected !== 'url') {
        return true
      }
      if (!this.isValidImage && this.selected === 'url') {
        return true
      }
      if (this.activeImage.length === 0 && this.selected !== 'url') {
        return true
      }
      return false
    }
  },
  methods: {
    pickFile() {
      this.$refs.image.click();
    },
    showOverlay() {
      this.show = true
      document.body.style.setProperty('position', 'fixed')
      document.body.style.setProperty("width", "100%");
      this.getImages()
    },
    closeOverlay() {
      this.show = false
      document.body.style.removeProperty('position')
      document.body.style.removeProperty("width");
    },
    onFilePicked(e) {
      let files = e.target.files;
      files.forEach((file) => {
        if (file.size > 512000) {
          this.errorMessages = ["Out of limit size"];
          return;
        }
        this.files = file
        this.uploadImage()
      });
    },
    uploadImage() {
      const files = new FormData()
      files.append('upload', this.files)
      files.append('working_dir', this.getSelectedFolder)
      this.$api.badasoFile.uploadUsingLfm(files)
      .then(res => {
        let error = _.get(res, 'data.original.error', null)
        if (error) {
          this.$vs.notify({
            title: this.$t("alert.danger"),
            text: error.message,
            color: "danger",
          });
        }
        this.getImages()
      }).catch(error => {
        console.error(error);
      })
    },
    getImages() {
      this.images.items = []
      if (this.getSelectedFolder) {
        this.$api.badasoFile.browseUsingLfm({
          workingDir: this.getSelectedFolder
        })
        .then(res => {
          let error = _.get(res, 'data.original.error', null)
          if (error) {
            this.$vs.notify({
              title: this.$t("alert.danger"),
              text: error.message,
              color: "danger",
            });
          }
          const items = res.data.items.filter(val => {
            return val.thumbUrl !== null
          })
          this.images = res.data
          this.images.items = items
        })
        .catch(error => {
          console.log(error);
        })
      }
    },
    emitInput() {
      if (this.selected !== 'url') {
        let url = []
        this.activeImage.forEach(element => {
          url.push(this.images.items[element].url)
        });
        this.imagesName = url.join(', ')
        this.isPicked = true
        this.$emit('input', url)
      } else {
        this.$emit('input', this.inputByUrl)
      }
      this.closeOverlay()
    },
    deleteImage() {
      this.$api.badasoFile.deleteUsingLfm({
        workingDir: this.getSelectedFolder,
        'items[]': this.images.items[this.activeImage].name
      })
      .then(res => {
        let error = _.get(res, 'data.original.error', null)
        if (error) {
          this.$vs.notify({
            title: this.$t("alert.danger"),
            text: error.message,
            color: "danger",
          });
        }
        this.getImages()
      })
      .catch(error => {
        console.log(error);
      })
      this.activeImage = []
      this.dialog = false
    },
    openDialog() {
      this.dialog = true
    },
    selectImage(index) {
      if (!this.activeImage.includes(index)) {
        this.activeImage.push(index)
      } else {
        let idx = this.activeImage.indexOf(index)
        this.activeImage.splice(idx, 1)
      }
      this.isImageSelected = true
    },
    deleteFilePicked(item) {
      if (item === null || item === undefined) return
      if (typeof item === 'string' && item !== '') {
        let idx = this.imageUrl.indexOf(item)
        let activeIdx = this.activeImage.indexOf(idx)
        this.activeImage.splice(activeIdx, 1)
        this.imageUrl.splice(idx, 1)
        this.imagesName = this.imageUrl.join(', ')
        this.$emit('input', this.imageUrl)
      }
    }
  },
};
</script>
