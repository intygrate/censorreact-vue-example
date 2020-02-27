<template>
  <div class="cr-demo-img">
    <div v-if="supportsImagePreview">
      <div
        ref="container"
        class="cr-demo-img__preview"
        :style="{maxWidth: previewWidth + 'px', height: previewHeight + 'px'}"
      >
        <canvas
          ref="previewCanvas"
          class="cr-demo-img__preview-pic"
          tabindex="0"
          :class="computedClasses"
          @drag.stop.prevent
          @dragover.stop.prevent
          @dragstart.stop.prevent
          @dragend.stop.prevent
          @dragenter.stop.prevent="onDragEnter"
          @dragleave.stop.prevent="onDragLeave"
          @drop.stop.prevent="onFileDrop"
          @click.prevent="onClick"
          @keyup.enter="onClick"
          :style="{height: previewHeight + 'px'}"
        ></canvas>
        <div
          v-if="!imageSelected"
          class="cr-demo-img__pic-inner"
          :style="{top: -previewHeight + 'px', marginBottom: -previewHeight + 'px'}"
        >
          <span class="cr-demo-img__pic-inner-text" v-html="strings.tap"></span>
        </div>
      </div>
      <button
        v-if="imageSelected && hideChangeButton"
        @click.prevent="selectImage"
        class="cr-demo-img__mod-button"
      >{{ strings.change }}</button>
      <button
        v-if="imageSelected && removable"
        @click.prevent="removeImage"
        class="cr-demo-img__rem-button"
      >{{ strings.remove }}</button>
    </div>
    <div v-else>
      <button
        v-if="imageSelected"
        @click.prevent="selectImage"
        class="cr-demo-img__mod-button"
      >{{ strings.select }}</button>
      <div v-else>
        <button
          v-if="hideChangeButton"
          @click.prevent="selectImage"
          class="cr-demo-img__mod-button"
        >{{ strings.change }}</button>
        <button
          v-if="!removable"
          @click.prevent="removeImage"
          class="cr-demo-img__rem-button"
        >{{ strings.remove }}</button>
      </div>
    </div>
    <input
      ref="fileInput"
      type="file"
      name="name"
      id="img-input"
      :accept="accept"
      @change="onFileChange"
    />
    <div class="cr-demo-img__stats">
      <button
        class="cr-demo-img__button"
        type="button"
        @click="runModeration()"
        :disabled="loading"
        :loading="loading"
      >Try Image API</button>
    </div>
  </div>
</template>

<script>
export default {
  name: 'ImageTesting',
  props: {
    token: String,
    showResultFn: Function,
  },
  data() {
    return {
      loading: false,
      profile: 'default',
      url: `https://dev.api.censorreact.intygrate.com/devv1/image`,
      margin: 0,
      hideChangeButton: true,
      crop: true,
      removable: true,
      accept: 'image/jpeg,image/png',
      toggleAspectRatio: false,
      autoToggleAspectRatio: false,
      imageSelected: false,
      draggingOver: false,
      previewHeight: 250,
      previewWidth: 250,
      canvasWidth: 250,
      canvasHeight: 250,
      image: '',
      prefill: '',
      strings: {
        upload: '<p>Your device does not support file uploading.</p>',
        tap: 'Drag an image or <br>click here to select a file',
        change: 'Change Photo',
        aspect: 'Landscape/Portrait',
        remove: 'Remove Photo',
        select: 'Select a Photo',
        selected: '<p>Photo successfully selected!</p>',
        fileSize: 'The file size exceeds the limit',
        fileType: 'This file type is not supported.',
      },
    };
  },
  mounted() {
    if (this.prefill) {
      this.preloadImage(this.prefill, this.prefillOptions);
    }
    this.$nextTick(() => {
      window.addEventListener('resize', this.onResize);
      this.onResize();
    });
    if (this.supportsImagePreview) {
      this.pixelRatio = Math.round(
        window.devicePixelRatio ||
          window.screen.deviceXDPI / window.screen.logicalXDPI,
      );
      const canvas = this.$refs.previewCanvas;

      if (canvas.getContext) {
        this.context = canvas.getContext('2d');
        this.context.scale(this.pixelRatio, this.pixelRatio);
      }
    }
    if (this.accept !== 'image/*') {
      this.fileTypes = this.accept.split(',');
      this.fileTypes = this.fileTypes.map((s) => s.trim());
    }
    this.$on('error', this.onError);
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.onResize);
    this.$off('error', this.onError);
  },
  methods: {
    onChange(image) {
      if (image) {
        this.showResultFn({ result: 'run test to see result' });
        this.image = image;
      }
    },
    onClick() {
      if (!this.imageSelected) {
        this.selectImage();
        return;
      }
      if (this.changeOnClick) {
        this.selectImage();
      }
      this.$emit('click');
    },
    onResize() {
      this.resizeCanvas();
      if (this.imageObject) {
        this.drawImage(this.imageObject);
      }
    },
    onDragEnter() {
      this.draggingOver = true;
    },
    onDragLeave() {
      this.draggingOver = false;
    },
    onFileDrop(e) {
      this.onDragLeave();
      this.onFileChange(e);
    },
    onFileChange(e, prefill) {
      const files = e.target.files || e.dataTransfer.files;
      if (!files.length) {
        return;
      }
      if (files[0].size <= 0 || files[0].size > this.size * 1024 * 1024) {
        this.$emit('error', {
          type: 'fileSize',
          fileSize: files[0].size,
          fileType: files[0].type,
          fileName: files[0].name,
          message: `${this.strings.fileSize} (${this.size} MB)`,
        });
        return;
      }
      this.file = files[0];
      this.fileName = files[0].name;
      this.fileSize = files[0].size;
      this.fileModified = files[0].lastModified;
      this.fileType = files[0].type;
      if (this.accept === 'image/*') {
        if (files[0].type.substr(0, 6) !== 'image/') {
          return;
        }
      } else if (this.fileTypes.indexOf(files[0].type) === -1) {
        this.$emit('error', {
          type: 'fileType',
          fileSize: files[0].size,
          fileType: files[0].type,
          fileName: files[0].name,
          message: this.strings.fileType,
        });
        return;
      }
      this.imageSelected = true;
      this.image = '';
      if (this.supportsImagePreview) {
        this.loadImage(files[0], prefill || false);
      } else if (prefill) {
        this.$emit('prefill');
      } else {
        this.$emit('change', this.image);
      }
    },
    loadImage(file, prefill) {
      const reader = new FileReader();
      reader.onload = (e) => {
        this.image = e.target.result;

        if (prefill) {
          this.$emit('prefill');
        } else {
          this.onChange(this.image);
        }

        this.imageObject = new Image();
        this.imageObject.onload = () => {
          if (this.autoToggleAspectRatio) {
            const canvasOrientation = this.getOrientation(
              this.canvasWidth,
              this.canvasHeight,
            );
            const imageOrientation = this.getOrientation(
              this.imageObject.width,
              this.imageObject.height,
            );

            if (canvasOrientation !== imageOrientation) {
              this.rotateCanvas();
            }
          }
          this.drawImage(this.imageObject);
        };
        this.imageObject.src = this.image;
      };

      reader.readAsDataURL(file);
    },
    drawImage(image) {
      this.imageWidth = image.width;
      this.imageHeight = image.height;
      this.imageRatio = image.width / image.height;
      let offsetX = 0;
      let offsetY = 0;
      let scaledWidth = this.previewWidth;
      let scaledHeight = this.previewHeight;
      const previewRatio = this.previewWidth / this.previewHeight;
      if (this.crop) {
        if (this.imageRatio >= previewRatio) {
          scaledWidth = scaledHeight * this.imageRatio;
          offsetX = (this.previewWidth - scaledWidth) / 2;
        } else {
          scaledHeight = scaledWidth / this.imageRatio;
          offsetY = (this.previewHeight - scaledHeight) / 2;
        }
      } else if (this.imageRatio >= previewRatio) {
        scaledHeight = scaledWidth / this.imageRatio;
        offsetY = (this.previewHeight - scaledHeight) / 2;
      } else {
        scaledWidth = scaledHeight * this.imageRatio;
        offsetX = (this.previewWidth - scaledWidth) / 2;
      }
      const canvas = this.$refs.previewCanvas;
      canvas.style.background = 'none';
      canvas.width = this.previewWidth * this.pixelRatio;
      canvas.height = this.previewHeight * this.pixelRatio;
      this.context.setTransform(1, 0, 0, 1, 0, 0);
      this.context.clearRect(0, 0, canvas.width, canvas.height);
      if (this.rotate) {
        this.context.translate(
          offsetX * this.pixelRatio,
          offsetY * this.pixelRatio,
        );
        this.context.translate(
          scaledWidth / (2 * this.pixelRatio),
          scaledHeight / (2 * this.pixelRatio),
        );
        this.context.rotate(this.rotate);
        offsetX = -scaledWidth / 2;
        offsetY = -scaledHeight / 2;
      }
      this.context.drawImage(
        image,
        offsetX * this.pixelRatio,
        offsetY * this.pixelRatio,
        scaledWidth * this.pixelRatio,
        scaledHeight * this.pixelRatio,
      );
    },
    selectImage() {
      this.$refs.fileInput.click();
    },
    removeImage() {
      this.$refs.fileInput.value = '';
      this.$refs.fileInput.type = '';
      this.$refs.fileInput.type = 'file';
      this.fileName = '';
      this.fileType = '';
      this.fileSize = 0;
      this.fileModified = 0;
      this.imageSelected = false;
      this.image = '';
      this.file = null;
      this.imageObject = null;
      this.$refs.previewCanvas.style.backgroundColor = 'rgba(200,200,200,.25)';
      this.$refs.previewCanvas.width = this.previewWidth * this.pixelRatio;
      this.$emit('remove');
    },
    resizeCanvas() {
      const previewRatio = this.canvasWidth / this.canvasHeight || 1;
      const newWidth = this.$refs.container.clientWidth;
      // const newheight = this.$refs.container.clientHeight;
      if (!this.toggleAspectRatio && newWidth === this.containerWidth) {
        return;
      }
      this.containerWidth = newWidth;
      this.previewWidth = Math.min(
        this.containerWidth - this.margin * 2,
        this.canvasWidth,
      );
      this.previewHeight = this.previewWidth / previewRatio;
    },
    getOrientation(width, height) {
      let orientation = 'square';
      if (width > height) {
        orientation = 'landscape';
      } else if (width < height) {
        orientation = 'portrait';
      }
      return orientation;
    },
    switchCanvasOrientation() {
      const { canvasWidth } = this;
      const { canvasHeight } = this;
      this.canvasWidth = canvasHeight;
      this.canvasHeight = canvasWidth;
    },
    rotateCanvas() {
      this.switchCanvasOrientation();
      this.resizeCanvas();
    },
    setOrientation(orientation) {
      this.rotate = false;
      if (orientation === 8) {
        this.rotate = -Math.PI / 2;
      } else if (orientation === 6) {
        this.rotate = Math.PI / 2;
      } else if (orientation === 3) {
        this.rotate = -Math.PI;
      }
    },
    getEXIFOrientation(file, callback) {
      const reader = new FileReader();
      reader.onload = (e) => {
        const view = new DataView(e.target.result);
        if (view.getUint16(0, false) !== 0xffd8) {
          return callback(-2);
        }
        const length = view.byteLength;
        let offset = 2;
        while (offset < length) {
          const marker = view.getUint16(offset, false);
          offset += 2;
          if (marker === 0xffe1) {
            if (view.getUint32(offset + 2, false) !== 0x45786966) {
              return callback(-1);
            }
            const little = view.getUint16((offset += 6), false) === 0x4949;
            offset += view.getUint32(offset + 4, little);
            const tags = view.getUint16(offset, little);
            offset += 2;
            for (let i = 0; i < tags; i += 1) {
              if (view.getUint16(offset + i * 12, little) === 0x0112) {
                return callback(view.getUint16(offset + i * 12 + 8, little));
              }
            }
          } else if ((marker && 0xff00) !== 0xff00) {
            break;
          } else {
            offset += view.getUint16(offset, false);
          }
        }
        return callback(-1);
      };
      reader.readAsArrayBuffer(file.slice(0, 65536));
    },
    preloadImage(source, options) {
      // ie 11 support
      let { File } = window;
      try {
        new File([], '');
      } catch (e) {
        File = class File extends Blob {
          constructor(chunks, filename, opts = {}) {
            super(chunks, opts);
            this.lastModifiedDate = new Date();
            this.lastModified = +this.lastModifiedDate;
            this.name = filename;
          }
        };
      }
      options = Object.assign({}, options);
      if (typeof source === 'object') {
        this.imageSelected = true;
        this.image = '';
        if (this.supportsImagePreview) {
          this.loadImage(source, true);
        } else {
          this.$emit('prefill');
        }
        return;
      }
      const headers = new Headers();
      headers.append('Accept', 'image/*');
      fetch(source, {
        method: 'GET',
        mode: 'cors',
        headers,
      })
        .then((response) => response.blob)
        .then((imageBlob) => {
          const e = { target: { files: [] } };
          const fileName = options.fileName || source.split('/').slice(-1)[0];
          let mediaType =
            options.mediaType ||
            `image/${options.fileType || fileName.split('.').slice(-1)[0]}`;
          mediaType = mediaType.replace('jpg', 'jpeg');
          e.target.files[0] = new File([imageBlob], fileName, {
            type: mediaType,
          });
          this.onFileChange(e, true);
        })
        .catch((err) => {
          this.$emit('error', {
            type: 'failedPrefill',
            message: `Failed loading prefill image: ${err}`,
          });
        });
    },
    onError(error) {
      console.log(error);
    },
    async runModeration() {
      try {
        this.loading = true;
        this.showResultFn({ result: 'processing...' });

        const fetchRequest = await fetch(this.url, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'x-api-key': this.token,
          },
          body: JSON.stringify({
            ImageBytes: this.image,
            profile: this.profile,
          }),
        });
        const data = await fetchRequest.json();

        this.showResultFn(data);
        this.loading = false;
      } catch (error) {
        this.loading = false;
        console.log('error');
      }
    },
  },
  computed: {
    supportsImagePreview() {
      return window.FileReader && !!window.CanvasRenderingContext2D;
    },
    computedClasses() {
      const classObject = {};
      classObject['dragging-over'] = this.draggingOver;
      return classObject;
    },
  },
  watch: {
    prefill() {
      if (this.prefill) {
        this.preloadImage(this.prefill, this.prefillOptions);
      } else {
        this.removeImage();
      }
    },
  },
};
</script>
