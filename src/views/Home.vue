<template>
  <div class="page">
    <el-tabs v-model="activeIndex">
      <el-tab-pane label="图片平均分阶" name="1" />
      <el-tab-pane label="图片统计分阶" name="2" />
      <el-tab-pane label="对比度调整" name="3" />
    </el-tabs>
    <div class="image-process-container">
      <!-- 参数设置 -->
      <div class="parameter-setting">
        <h3> 参数设置 </h3>
        <el-form label-width="80px" inline>
          <el-form-item v-if="activeIndex !== '3'" label="分阶">
            <el-input-number
              v-model="currentData.gradation"
              :min="1"
              :step="1"
              size="small" />
          </el-form-item>
          <el-form-item v-if="activeIndex !== '3'" label="灰度">
            <el-radio-group v-model="currentData.grayscale">
              <el-radio :label="0">否</el-radio>
              <el-radio :label="1">是</el-radio>
            </el-radio-group>
          </el-form-item>
          <el-form-item v-if="activeIndex === '3'" label="对比度">
            <el-input-number
              v-model="currentData.contrast"
              :min="0"
              :step="0.1"
              size="small" />
          </el-form-item>
        </el-form>
        <el-button :disabled="loading" type="primary" @click="requestImageProcess(currentData)">处理图片</el-button>
        <el-button type="primary" @click="openImageListDialog('download')">下载处理结果</el-button>
      </div>
      <!-- 原图 -->
      <div class="origin-image">
        <h3> 原图 </h3>
        <template v-if="currentData.originImage.file">
          <img :src="currentData.originImage.file.url" alt="原图" >
          <el-button type="primary" @click="openImageListDialog('choose')">更改图片</el-button>
        </template>
        <template v-else>
          <el-button type="primary" @click="openImageListDialog('choose')">选择图片</el-button>
        </template>
      </div>
      <!-- 处理结果 -->
      <div class="result-image">
        <h3> 处理结果 </h3>
        <div
          v-loading="loading"
          element-loading-text="图片处理中"
          element-loading-spinner="el-icon-loading"
          element-loading-background="rgba(0, 0, 0, 0.2)"
          style="min-height: 70%">
          <template v-if="currentData.results.length > 0">
            <img :src="currentData.results[currentData.resultIndex].file.url" alt="处理结果" >
          </template>
        </div>
      </div>
      <!-- 处理结果列表 -->
      <div v-if="currentData.results.length > 0" class="result-list">
        <h3> 处理记录 </h3>
        <div class="image-list">
          <thumbnail
            v-for="(item, index) in currentData.results"
            :key="'thumbnail-' + index"
            :file="item.file"
            :choose="currentData.resultIndex === index"
            @image-click="handleResultImageClick(index)"
            @preview="handlePreviewImage"
            @download="handleDownloadImage"
            @remove="handleRemoveImage(item.file)" />
        </div>
      </div>
    </div>

    <!-- 图片选择弹窗 -->
    <el-dialog
      :visible.sync="imageListDialog.show"
      :title="imageListDialog.type === 'choose' ? '选择图片' : '下载处理结果'"
      width="60%">
      <div class="image-choose">
        <el-checkbox
          v-if="imageListDialog.type === 'download'"
          v-model="imageListDialog.chooseAll"
          :indeterminate="imageListDialog.indeterminate"
          @change="handleChooseAllChange">
          全选
        </el-checkbox>
        <div v-for="item in imageList" :key="item.title">
          <div
            v-if="(item.list.length > 0 || item.title === '本地图片')
            && (imageListDialog.type !== 'download' || item.title !== '本地图片')">
            <h3>{{ item.title }}</h3>
            <div class="image-list">
              <thumbnail
                v-for="(item1, index1) in item.list"
                :key="'thumbnail-' + index1"
                :file="item1.file"
                :choose="item1.isChoose"
                @image-click="handleImageClick(item1)"
                @preview="handlePreviewImage"
                @download="handleDownloadImage"
                @remove="handleRemoveImage(item1.file)" />
            </div>
            <template v-if="item.title === '本地图片'">
              <el-upload
                :auto-upload="false"
                :on-change="handleFileChange"
                :file-list="imageUpload.fileList"
                list-type="picture-card"
                multiple
                accept="image/jpeg,image/png,image/jpg"
                action="#">
                <i slot="default" class="el-icon-plus" />
                <div slot="file" slot-scope="{file}">
                  <thumbnail :file="file" @preview="handlePreviewImage" />
                </div>
                <div slot="tip" class="el-upload__tip">只能上传jpg, png, jpeg格式的图片, 且不超过2MB</div>
              </el-upload>
            </template>
          </div>
        </div>
      </div>
      <span slot="footer" class="dialog-footer">
        <el-button @click="imageListDialog.show = false">取 消</el-button>
        <el-button v-if="imageListDialog.type === 'choose'" type="primary" @click="handleImageListDialogConfirm">确 定</el-button>
        <el-button v-if="imageListDialog.type === 'download'" type="primary" @click="requestDownloadImage">确 定</el-button>
      </span>
    </el-dialog>

    <!-- 图片预览弹窗 -->
    <el-dialog :visible.sync="previewDialog.show">
      <img :src="previewDialog.url" width="100%" alt="">
    </el-dialog>
  </div>
</template>

<script>
import Thumbnail from '../components/Thumbnail.vue'

export default {
  components: {
    Thumbnail
  },
  data() {
    return {
      activeIndex: '1',
      imageUpload: {
        fileList: []
      },
      imageList: [
        { title: '本地图片', list: [] },
        { title: '图片平均分阶', list: [] },
        { title: '图片统计分阶', list: [] },
        { title: '对比度调整', list: [] }
      ],
      imageProcessData: [
        {
          apiUrl: 'api/equal_distribution',
          originImage: {},
          title: '图片平均分阶',
          gradation: 3,
          grayscale: 0,
          results: [],
          resultIndex: 0
        },
        {
          apiUrl: 'api/statistical_distribution',
          originImage: {},
          title: '图片统计分阶',
          gradation: 3,
          grayscale: 0,
          results: [],
          resultIndex: 0
        },
        {
          apiUrl: 'api/adjust_contrast',
          originImage: {},
          title: '对比度调整',
          contrast: 1,
          results: [],
          resultIndex: 0
        }
      ],
      currentData: {},
      imageListDialog: {
        show: false,
        type: 'choose',
        chooseAll: false,
        indeterminate: false
      },
      previewDialog: {
        show: false,
        url: ''
      },
      loading: false,
      resultMaps: [{}, {}, {}] // url -> results
    }
  },
  watch: {
    activeIndex: {
      handler(val) {
        const oldData = this.currentData
        this.currentData = this.imageProcessData[val - 1]
        if (oldData.originImage.file) {
          for (const key in oldData.originImage) {
            this.currentData.originImage[key] = oldData.originImage[key]
          }
        }
      },
      immediate: true
    }
  },
  methods: {
    /**
     * 上传图片时图片格式和大小的校验
     * @param {*} file
     * @param {*} fileList
     */
    handleFileChange(file, fileList) {
      const allowType = ['image/jpeg', 'image/png', 'image/jpg']
      const isLt2M = file.size / 1024 / 1024 < 2
      const typeCorrect = allowType.includes(file.raw.type)

      if (!typeCorrect) {
        this.$message.error('上传图片只能是 JPG/PNG/JPEG 格式!')
      }
      if (!isLt2M) {
        this.$message.error('上传图片大小不能超过 2MB!')
      }
      if (typeCorrect && isLt2M) {
        this.imageUpload.fileList = []
        const item = { file, isChoose: false }
        this.imageList[0].list.push(item)
        this.handleImageClick(item)
      } else {
        fileList.pop()
      }
    },

    /**
     * 图片列表窗口中图片缩略图点击事件
     * @param {*} item
     */
    handleImageClick(item) {
      const newValue = !item.isChoose
      if (this.imageListDialog.type === 'choose') {
        this.setAllImageChoose(false)
      }
      item.isChoose = newValue
      if (this.imageListDialog.type === 'download') {
        this.checkIndeterminate()
      }
      this.$forceUpdate()
    },

    /**
     * 图片预览
     * @param {*} file
     */
    handlePreviewImage(file) {
      this.previewDialog.url = file.url
      this.previewDialog.show = true
    },

    /**
     * 图片下载
     * @param {*} file
     */
    handleDownloadImage(file) {
      const a = document.createElement('a')
      a.href = file.url
      a.download = file.name
      a.click()
      a.remove()
    },

    /**
     * 删除图片
     * @param {*} file
     */
    handleRemoveImage(file) {
      const url = file.url
      // 清除图片列表中的图片
      for (const item of this.imageList) {
        item.list = item.list.filter(item1 => item1.file.url !== url)
      }
      // 清除图片处理数据中的图片
      for (const item of this.imageProcessData) {
        if (item.originImage.file && item.originImage.file.url === url) {
          item.originImage = {}
        }
        for (const item1 of item.results) {
          if (item1.file.url === url) {
            item.results.splice(item.results.indexOf(item1), 1)
            item.resultIndex = item.results.length - 1
          }
        }
      }
      // 清除历史记录中的图片
      for (const item of this.resultMaps) {
        if (item[url]) {
          delete item[url]
        }
        for (const key in item) {
          item[key] = item[key].filter(item1 => item1.file.url !== url)
        }
      }
    },

    /**
     * 图片列表窗口中确认按钮点击事件
     */
    /* eslint-disable */
    handleImageListDialogConfirm() {
      outerLoop:
      for (const item0 of this.imageList) {
        for (const item of item0.list) {
          if (item.isChoose) {
            if (this.currentData.originImage.file && this.currentData.originImage.file.url === item.file.url) {
              break outerLoop
            }
            this.currentData.originImage = item
            const resultHistory = this.resultMaps[parseInt(this.activeIndex) - 1]
            if (resultHistory[item.file.url]) {
              this.currentData.results = resultHistory[item.file.url]
            } else {
              this.currentData.results = []
              resultHistory[item.file.url] = this.currentData.results
            }
            break outerLoop
          }
        }
      }
      this.imageListDialog.show = false
    },
    /* eslint-enable */

    /**
     * 上传图片
     */
    requestUploadImage(item) {
      if (item.path) {
        return Promise.resolve({ file_path: item.path })
      }
      return new Promise((resolve, reject) => {
        const formData = new FormData()
        formData.append('file', item.file.raw)
        this.$axios.post('api/upload', formData).then(res => {
          item.path = res.data.file_path.replace('\\', '/')
          resolve(item)
        }).catch(err => {
          reject(err)
        })
      })
    },

    /**
     * 请求图片处理接口
     * @param {*} data
     */
    requestImageProcess(data) {
      if (!data.originImage.file) {
        this.$message.error('请选择图片')
        return
      }
      this.loading = true
      this.requestUploadImage(data.originImage).then(() => {
        const params = new URLSearchParams({
          file_path: data.originImage.path
        })
        if (this.activeIndex !== '3') {
          if (isNaN(data.gradation) || data.gradation <= 0) {
            this.$message.error('分阶必须是大于0的整数')
            return
          }
          params.append('gradation', parseInt(data.gradation))
          params.append('grayscale', data.grayscale)
        } else {
          if (isNaN(data.contrast) || data.contrast < 0) {
            this.$message.error('对比度必须是不小于0的数字')
            return
          }
          params.append('contrast', data.contrast)
        }
        this.$axios.post(data.apiUrl, params).then(res => {
          const path = res.data.file_path.replace('\\', '/')
          const result = {
            path: path,
            file: {
              url: this.$fileUrl + path
            },
            gradation: data.gradation,
            grayscale: data.grayscale,
            contrast: data.contrast
          }
          data.results.push(result)
          data.resultIndex = data.results.length - 1
          this.imageList[parseInt(this.activeIndex)].list.push(result)
        }).catch(err => {
          this.$message.error(err)
        }).finally(() => {
          this.loading = false
        })
      }).catch(() => {
        this.loading = false
      })
    },

    /**
     * 批量下载处理结果
     */
    requestDownloadImage() {
      const chooseList = []
      for (const item of this.imageList) {
        for (const item1 of item.list) {
          if (item1.isChoose) {
            chooseList.push(item1.path)
          }
        }
      }
      if (chooseList.length === 0) {
        this.$message.error('请选择要下载的图片')
        return
      }
      this.$axios.post('api/download', { file_path_list: chooseList }, { responseType: 'blob' }).then(res => {
        const blob = new Blob([res.data], { type: 'application/zip' })
        const url = window.URL.createObjectURL(blob)
        const a = document.createElement('a')
        a.href = url
        a.download = chooseList.length > 1 ? '处理结果.zip' : chooseList[0].split('/').pop()
        a.click()
        window.URL.revokeObjectURL(url)
      }).catch(err => {
        this.$message.error(err)
      })
    },

    /**
     * 处理结果列表中图片点击事件
     * @param {*} index
     */
    handleResultImageClick(index) {
      this.currentData.resultIndex = index
      const result = this.currentData.results[index]
      this.currentData.grayscale = result.grayscale
      this.currentData.gradation = result.gradation
      this.currentData.contrast = result.contrast
    },

    /**
     * 打开图片选择弹窗
     * @param {*} type
     */
    openImageListDialog(type) {
      this.imageListDialog.show = true
      this.imageListDialog.type = type
      // 清除图片选择状态
      this.setAllImageChoose(false)
    },

    /**
     * 全选按钮点击事件
     */
    handleChooseAllChange() {
      this.setAllImageChoose(this.imageListDialog.chooseAll)
    },

    /**
     * 设置全部图片的选择状态
     * @param {*} value
     */
    setAllImageChoose(value) {
      for (const item1 of this.imageList) {
        for (const item2 of item1.list) {
          item2.isChoose = value
        }
      }
      this.imageListDialog.chooseAll = value
      this.imageListDialog.indeterminate = false
    },

    /**
     * 检查图片是否已经选择
     */
    checkIndeterminate() {
      let chooseCount = 0
      let totalCount = 0
      for (let i = 1; i < this.imageList.length; i++) {
        const item1 = this.imageList[i]
        for (const item2 of item1.list) {
          totalCount++
          if (item2.isChoose) {
            chooseCount++
          }
        }
      }
      this.imageListDialog.indeterminate = chooseCount > 0 && chooseCount < totalCount
      this.imageListDialog.chooseAll = chooseCount === totalCount && totalCount > 0
    }
  }
}
</script>

<style scoped>
.page {
  padding: 20px;
}
.image-process-container {
  display: flex;
  flex-wrap: wrap;
  text-align: left;
}
.parameter-setting {
  width: 100%;
}
.origin-image {
  width: 40%;
  margin-right: 20px;
  margin-bottom: 10px;
}
.origin-image img {
  width: 100%;
}
.result-image {
  width: 40%;
  margin-right: 20px;
  margin-bottom: 10px;
}
.result-image img {
  width: 100%;
}
.result-list {
  width: 100%;
}
.image-list {
  display: flex;
  flex-wrap: wrap;
}
.image-choose {
  text-align: left;
}
</style>
