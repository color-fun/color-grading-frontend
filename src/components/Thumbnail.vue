<template>
  <div class="el-upload-list--picture-card img-container" @click="handleImageClick">
    <img
      :src="file.url"
      :class="{ 'el-upload-list__item-thumbnail': true, 'is-choose': choose }"
      alt=""
    >
    <span class="el-upload-list__item-actions">
      <span
        class="el-upload-list__item-preview"
        @click="handlePictureCardPreview(file)"
      >
        <i class="el-icon-zoom-in"/>
      </span>
      <!-- <span
        class="el-upload-list__item-delete"
        @click="handleDownload(file)"
      >
        <i class="el-icon-download"/>
      </span> -->
      <span
        class="el-upload-list__item-delete"
        @click="handleRemove(file)"
      >
        <i class="el-icon-delete"/>
      </span>
    </span>
    <span v-show="choose" class="confirm">
      <i class="el-icon-circle-check"/>
    </span>
  </div>
</template>

<script>
export default {
  props: {
    file: {
      type: Object,
      required: true
    },
    choose: {
      type: Boolean,
      default: false
    }
  },
  methods: {
    handlePictureCardPreview(file) {
      this.$emit('preview', file)
    },
    handleDownload(file) {
      this.$emit('download', file)
    },
    handleRemove(file) {
      this.$emit('remove', file)
    },
    handleImageClick() {
      this.$emit('image-click')
    }
  }
}
</script>

<style scoped>
.confirm {
  position: absolute;
  bottom: 0;
  right: 0;
  font-size: 40px;
}
.confirm i {
  color: #06ce13 !important;
}
.is-choose {
  opacity: 0.5;
}
.img-container {
  width: 146px;
  height: 146px;
  border-radius: 4px;
  position: relative;
  overflow: hidden;
  margin-right: 5px;
  margin-bottom: 5px;
}
</style>
