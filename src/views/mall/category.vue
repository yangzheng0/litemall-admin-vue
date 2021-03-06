<template>
  <div class="app-container">
    <!-- 查询和其他操作 -->
    <div class="filter-container">
        <el-button v-permission="['POST /admin/category/create']" class="filter-item" type="primary" icon="el-icon-edit" @click="handleCreate">添加</el-button>
    </div>

    <!-- 查询结果 -->
    <el-table v-loading="listLoading" :data="list" element-loading-text="正在查询中。。。" row-key="id" border fit highlight-current-row>
      <el-table-column align="center" label="类目ID" prop="id"></el-table-column>
      <el-table-column align="center" label="类目名" prop="name"></el-table-column>
      <el-table-column align="center" label="类目图标" property="iconUrl">
        <template slot-scope="scope">
          <img v-if="scope.row.iconUrl" :src="scope.row.iconUrl" width="40">
        </template>
      </el-table-column>
      <el-table-column align="center" property="picUrl" label="类目图片">
        <template slot-scope="scope">
          <img v-if="scope.row.picUrl" :src="scope.row.picUrl" width="80">
        </template>
      </el-table-column>

      <el-table-column align="center" label="关键字" prop="keywords"/>

      <el-table-column align="center" label="简介" min-width="100" prop="desc"/>

      <el-table-column align="cener" label="级别" prop="level">
          <template slot-scope="scope">
            <el-tag :type="scope.row.level === 'L1' ? 'primary' : 'info'">{{scope.row.level === 'L1' ? '一级类目' : '二级类目'}}</el-tag>
          </template>
      </el-table-column>

      <el-table-column align="center" label="操作" width="200" class-name="small-padding fixed-width">
        <template slot-scope="scope">
          <el-button v-permission="['POST /admin/category/update']" type="primary" size="mini" @click="handleUpdate(scope.row)">编辑</el-button>
          <el-button v-permission="['POST /admin/category/delete']" type="danger" size="mini" @click="handleDelete(scope.row)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>

    <!-- 添加或者修改对话框 -->
    <el-dialog :title="textMap[dialogStatus]" :visible.sync="dialogFormVisible">
      <el-form ref="dataForm" :rules="rules" :model="dataForm" status-icon label-position="left" label-width="100px" style="width: 400px;margin-left: 50px; ">
        <el-form-item label="类目名称" prop="name">
          <el-input v-model="dataForm.name"/>
        </el-form-item>
        <el-form-item label="关键字" prop="keywords">
          <el-input v-model="dataForm.keywords"/>
        </el-form-item>
        <el-form-item label="级别" prop="level">
          <el-select v-model="dataForm.level" @change="onLevelChange">
            <el-option label="一级类目" value="L1"/>
            <el-option label="二级类目" value="L2"/>
          </el-select>
        </el-form-item>
        <el-form-item label="父类目" v-if="dataForm.level === 'L2'" prop="pid">
          <el-select v-model="dataForm.pid">
            <el-option v-for="item in catL1" :key="item.value" :label="item.value" :value="item.value"/>
          </el-select>
        </el-form-item>

        <el-form-item label="类目图标" prop="iconUrl">
          <el-upload
            :headers="headers"
            :action="uploadPath"
            :show-file-list="false"
            :on-success="uploadIconUrl"
            class="avatar-uploader"
            accept=".jpg,.png,.jepg,.gif">
            <img v-if="dataForm.iconUrl" :src="dataForm.iconUrl" class="avatar">
            <i v-else class="el-icon-plus avatar-uploader-icon"/>
          </el-upload>
        </el-form-item>

        <el-form-item label="类目图片" prop="picUrl">
          <el-upload
            :headers="headers"
            :action="uploadPath"
            :show-file-list="false"
            :on-success="uploadPicUrl"
            class="avatar-uploader"
            accept=".jpg,.png,.jepg,.gif">
            <img v-if="dataForm.picUrl" :src="dataForm.picUrl" class="avatar">
            <i v-else class="el-icon-plus avatar-uploader-icon"/>
          </el-upload>
        </el-form-item>

        <el-form-item label="类目简介" prop="desc">
          <el-input v-model="dataForm.desc"/>
        </el-form-item>
      </el-form>

      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible = false">取消</el-button>
        <el-button v-if="dialogStatus == 'create'" type="primary" @click="createData">确定</el-button>
        <el-button v-else type="primary" @click="updateData">确定</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import {createCategory, listCategory, listCatL1, updateCategory, deleteCategory} from '@/api/category'
import { getToken } from '@/utils/auth'
import { uploadPath } from '@/api/storage'
export default {
  name: 'category',
  data () {
    return {
      uploadPath,
      listLoading: false,
      list: [],
      textMap: {
        update: '编辑',
        create: '创建'
      },
      dialogStatus: '',
      dialogFormVisible: false,
      dataForm: {
        id: undefined,
        name: '',
        keywords: '',
        level: 'L2',
        pid: 0,
        desc: '',
        iconUrl: '',
        picUrl: ''
      },
      rules: {
        name: [{required: true, message: '类目名不能为空', trigger: 'blur'}]
      },
      catL1: {}
    }
  },
  computed: {
    headers () {
      return {
        'X-Litemall-Admin-Token': getToken()
      }
    }
  },
  created () {
    this.getList()
    this.getCatL1()
  },
  methods: {
    createData () {
      this.$refs['dataForm'].validate(valid => {
        if (valid) {
          createCategory(this.dataForm).then(response => {
            this.getList()
            // 更新L1目录
            this.getCatL1()
            this.dialogFormVisible = false
            this.$notify.success({
              title: '成功',
              message: '创建成功'
            })
          }).catch(response => {
            this.$notify.error({
              title: '失败',
              message: response.data.errmsg
            })
          })
        }
      })
    },
    updateData () {
      this.$refs['dataForm'].validate(valid => {
        if (valid) {
          updateCategory(this.dataForm).then(
            () => {
              this.getList()
              // 更新目录
              this.getCatL1()
              this.dialogFormVisible = false
              this.$notify.success({
                title: '成功',
                message: '更新成功'
              })
            }).catch(response => {
            this.$notify.error({
              title: '失败',
              message: response.data.errmsg
            })
          })
        }
      })
    },
    resetForm () {
      this.dataForm = {
        id: undefined,
        name: '',
        keywords: '',
        level: 'L2',
        pid: 0,
        desc: '',
        iconUrl: '',
        picUrl: ''
      }
    },
    handleCreate () {
      this.resetForm()
      this.dialogStatus = 'create'
      this.dialogFormVisible = true
      this.$nextTick(() => {
        this.$refs['dataForm'].clearValidate()
      })
    },
    getList () {
      this.listLoading = true
      listCategory().then(response => {
        this.list = response.data.data.list
        this.listLoading = false
      })
        .catch(() => {
          this.list = []
          this.listLoading = false
        })
    },
    getCatL1 () {
      listCatL1().then(response => {
        this.catL1 = response.data.data.list
      })
    },
    handleUpdate (row) {
      this.dataForm = Object.assign({}, row)
      this.dialogStatus = 'update'
      this.dialogFormVisible = true
      this.$nextTick(() => {
        this.$refs['dataForm'].clearValidate()
      })
    },
    handleDelete (row) {
      deleteCategory(row).then(response => {
        this.getList()
        // 更新L1目录
        this.getCatL1()
        this.$notify.success({
          title: '成功',
          message: '删除成功'
        })
      }).catch(response => {
        this.$notify.error({
          title: '失败',
          message: response.data.errmsg
        })
      })
    },
    onLevelChange: function (value) {
      if (value === 'L1') {
        this.dataForm.pid = 0
      }
    },
    uploadIconUrl: function (response) {
      this.dataForm.iconUrl = response.data.url
    },
    uploadPicUrl: function (response) {
      this.dataForm.picUrl = response.data.url
    }
  }
}
</script>

<style scoped>
.filter-item{
  margin-left: 100px;
}
.avatar-uploader .el-upload {
  border: 1px dashed #d9d9d9;
  border-radius: 6px;
  cursor: pointer;
  position: relative;
  overflow: hidden;
}
.avatar-uploader {
  border-color: #20a0ff;
}
.avatar-uploader-icon{
  font-size: 28px;
  color: #8c939d;
  width: 120px;
  height: 120px;
  line-height: 120px;
  text-align: center;
}
.avatar{
  width: 145px;
  height: 145px;
  display: block;
}
</style>
