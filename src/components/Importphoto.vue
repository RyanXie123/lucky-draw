<template>
  <el-dialog
    :visible="visible"
    :append-to-body="true"
    width="500px"
    @close="$emit('update:visible', false)"
    class="c-Importphoto"
  >
    <div slot="title">批量导入照片</div>
    <el-row>
      <label>选择文件夹</label>
      <span class="selectbg" :data-tip="folderTip">
        <input
          ref="uploadinput"
          class="upload"
          type="file"
          accept=".jpg,.png,.jpeg"
          multiple
          webkitdirectory
          directory
          @change="inputChange"
        />
      </span>
    </el-row>
    <el-row>
      <div class="tips">
        <p>照片命名格式：<strong>id.姓名.jpg</strong>（例如：2.果果.jpg）</p>
        <p>支持jpg、jpeg和png格式，建议尺寸为160*160px</p>
      </div>
    </el-row>
    <el-row v-if="photoList.length > 0" class="photo-list">
      <label>待导入照片列表（共{{ photoList.length }}张）</label>
      <div class="list-container">
        <div v-for="(item, index) in photoList" :key="index" class="photo-item">
          <img :src="item.value" alt="img" :width="60" :height="60" />
          <div class="photo-info">
            <p>ID: {{ item.id }}</p>
            <p>姓名: {{ item.name }}</p>
          </div>
        </div>
      </div>
    </el-row>
    <el-row v-if="errorList.length > 0" class="error-list">
      <label>解析失败的文件（{{ errorList.length }}个）</label>
      <div class="error-container">
        <p v-for="(error, index) in errorList" :key="index" class="error-item">
          {{ error }}
        </p>
      </div>
    </el-row>
    <el-row class="center">
      <el-button size="mini" type="primary" @click="saveHandler" :disabled="photoList.length === 0"
        >批量保存（{{ photoList.length }}张）</el-button
      >
      <el-button size="mini" @click="$emit('update:visible', false)"
        >取消</el-button
      >
    </el-row>
  </el-dialog>
</template>
<script>
import { database, DB_STORE_NAME } from '@/helper/db';

export default {
  name: 'Importphoto',
  props: {
    visible: Boolean
  },
  computed: {
    config: {
      get() {
        return this.$store.state.config;
      }
    }
  },
  data() {
    return {
      photoList: [],
      errorList: [],
      folderTip: '点击选择文件夹'
    };
  },
  methods: {
    async inputChange(e) {
      const fileList = Array.from(e.target.files);
      this.photoList = [];
      this.errorList = [];
      
      if (fileList.length === 0) {
        return;
      }

      const validExtensions = ['.jpg', '.jpeg', '.png'];
      
      // 处理每个文件
      const promises = fileList.map(file => {
        return new Promise((resolve) => {
          const fileName = file.name;
          const fileExt = fileName.substring(fileName.lastIndexOf('.')).toLowerCase();
          
          // 检查文件扩展名
          if (!validExtensions.includes(fileExt)) {
            this.errorList.push(`${fileName}: 不支持的文件格式`);
            resolve();
            return;
          }

          // 解析文件名格式：id.姓名.jpg
          const nameWithoutExt = fileName.substring(0, fileName.lastIndexOf('.'));
          const parts = nameWithoutExt.split('.');
          
          if (parts.length < 2) {
            this.errorList.push(`${fileName}: 文件名格式错误，应为"id.姓名.jpg"`);
            resolve();
            return;
          }

          const id = parseInt(parts[0]);
          const name = parts.slice(1).join('.');

          if (isNaN(id) || id <= 0) {
            this.errorList.push(`${fileName}: ID必须是大于0的整数`);
            resolve();
            return;
          }

          // 读取文件（无大小限制）
          const reader = new FileReader();
          reader.onload = () => {
            this.photoList.push({
              id: id,
              name: name,
              value: reader.result,
              fileName: fileName
            });
            resolve();
          };
          reader.onerror = () => {
            this.errorList.push(`${fileName}: 读取文件失败`);
            resolve();
          };
          reader.readAsDataURL(file);
        });
      });

      await Promise.all(promises);

      // 按ID排序
      this.photoList.sort((a, b) => a.id - b.id);

      // 自动更新config.number为最大ID
      if (this.photoList.length > 0) {
        const maxId = Math.max(...this.photoList.map(p => p.id));
        const newConfig = {
          ...this.config,
          number: maxId
        };
        this.$store.commit('setConfig', newConfig);
        this.$message.success(`成功解析${this.photoList.length}张照片，已自动设置总人数为${maxId}`);
      }
      if (this.errorList.length > 0) {
        this.$message.warning(`${this.errorList.length}个文件解析失败`);
      }
    },
    async saveHandler() {
      if (this.photoList.length === 0) {
        return this.$message.error('请先选择照片');
      }

      const loading = this.$loading({
        lock: true,
        text: '正在保存...',
        spinner: 'el-icon-loading',
        background: 'rgba(0, 0, 0, 0.7)'
      });

      let successCount = 0;
      let failCount = 0;

      try {
        for (const photo of this.photoList) {
          try {
            const Data = await database.get(DB_STORE_NAME, photo.id);
            const param = {
              id: photo.id,
              value: photo.value,
              name: photo.name
            };
            
            const res = await database[Data ? 'edit' : 'add'](
              DB_STORE_NAME,
              Data ? photo.id : param,
              Data ? param : null
            );

            if (res) {
              successCount++;
            } else {
              failCount++;
            }
          } catch (err) {
            console.error(`保存ID ${photo.id} 失败:`, err);
            failCount++;
          }
        }

        loading.close();

        if (successCount > 0) {
          this.$message({
            message: `成功保存${successCount}张照片${failCount > 0 ? `，失败${failCount}张` : ''}`,
            type: successCount === this.photoList.length ? 'success' : 'warning'
          });
          
          this.$refs.uploadinput.value = '';
          this.photoList = [];
          this.errorList = [];
          this.folderTip = '点击选择文件夹';
          this.$emit('update:visible', false);
          this.$emit('getPhoto');
        } else {
          this.$message.error('保存失败');
        }
      } catch (err) {
        loading.close();
        this.$message.error(err.message);
      }
    }
  }
};
</script>
<style lang="scss">
.c-Importphoto {
  .el-dialog {
    min-height: 400px;
  }
  label {
    margin-right: 20px;
    vertical-align: top;
    font-weight: bold;
    color: #333;
  }
  .el-input {
    width: 140px;
  }
  .el-row {
    padding: 10px 0;
  }
  .center {
    text-align: center;
    margin-top: 20px;
  }
  .tips {
    background: #f0f9ff;
    border: 1px solid #b3d8ff;
    border-radius: 4px;
    padding: 10px;
    font-size: 12px;
    color: #606266;
    p {
      margin: 5px 0;
      strong {
        color: #409eff;
      }
    }
  }
  .selectbg {
    display: inline-block;
    border: 1px solid #dcdfe6;
    border-radius: 4px;
    width: 200px;
    height: 32px;
    position: relative;
    box-sizing: border-box;
    background: #fff;
    transition: border-color 0.2s;
    &:hover {
      border-color: #409eff;
    }
    &::before {
      content: attr(data-tip);
      width: 100%;
      height: 100%;
      text-align: center;
      position: absolute;
      top: 0;
      left: 0;
      line-height: 32px;
      cursor: pointer;
      overflow: hidden;
      font-size: 13px;
      color: #606266;
    }
    .upload {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      opacity: 0;
      z-index: 10;
      cursor: pointer;
    }
  }
  .photo-list {
    label {
      display: block;
      margin-bottom: 10px;
    }
    .list-container {
      max-height: 300px;
      overflow-y: auto;
      border: 1px solid #ebeef5;
      border-radius: 4px;
      padding: 10px;
      background: #fafafa;
    }
    .photo-item {
      display: inline-flex;
      align-items: center;
      margin: 5px 10px 5px 0;
      padding: 8px;
      background: #fff;
      border: 1px solid #e4e7ed;
      border-radius: 4px;
      img {
        border: 1px solid #dcdfe6;
        border-radius: 4px;
        object-fit: cover;
      }
      .photo-info {
        margin-left: 10px;
        font-size: 12px;
        p {
          margin: 3px 0;
          color: #606266;
        }
      }
    }
  }
  .error-list {
    label {
      display: block;
      margin-bottom: 10px;
      color: #f56c6c;
    }
    .error-container {
      max-height: 150px;
      overflow-y: auto;
      border: 1px solid #fde2e2;
      border-radius: 4px;
      padding: 10px;
      background: #fef0f0;
    }
    .error-item {
      margin: 3px 0;
      font-size: 12px;
      color: #f56c6c;
    }
  }
}
</style>
