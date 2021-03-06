<template>
  <div class="app-container calendar-list-container">

    <el-table :key='tableKey' :data="list" v-loading="listLoading" element-loading-text="给我一点时间" border fit highlight-current-row style="width: 100%">

      <el-table-column align="center" label="序号" width="65">
        <template slot-scope="scope">
          <span>{{Number(scope.$index + 1)}}</span>
        </template>
      </el-table-column>

      <el-table-column width="180px" align="center" label="时间">
        <template slot-scope="scope">
          <span>{{scope.row.time | parseTime }}</span>
        </template>
      </el-table-column>

      <el-table-column min-width="300px" label="标题">
        <template slot-scope="scope">
          <el-button type="text" @click=handleUpdate(scope.row,scope.row.id) size="medium" >{{scope.row.title}}</el-button>
          <!--<el-tag>{{scope.row.type | typeFilter}}</el-tag>-->
        </template>
      </el-table-column>

      <el-table-column align="center" label="操作" width="250">
        <template slot-scope="scope">
          <div style="float: left">
            <el-button v-show="scope.row.status==='top'" size="small" type="primary" @click="handleCancelTop(scope.row.id,scope.row)">取消置顶
            </el-button>
            <el-button v-if="scope.row.status!=='top' && scope.row.status!=='draft'" size="small" type="primary" @click="handleTop(scope.row.id,scope.$index,scope.row)">置顶
            </el-button>
            <el-button v-if="scope.row.status!=='published' && scope.row.status!=='top'" size="small" type="success" @click="handlePublish(scope.row.id,scope.row)">发布
            </el-button>
            <el-button v-if="scope.row.status!=='top' && scope.row.status!=='draft'" size="small" @click="handleDraft(scope.row.id,scope.row)">草稿
            </el-button>
            <el-button v-if="scope.row.status!=='deleted'" size="small" type="danger" @click="handleDelete(scope.row.id, scope.$index)">删除
            </el-button>
          </div>
        </template>
      </el-table-column>

    </el-table>

    <div v-show="!listLoading" class="pagination-container">
      <el-pagination @size-change="handleSizeChange" @current-change="handleCurrentChange" :current-page.sync="listQuery.page"
                     :page-sizes="[10,20,30, 50]" :page-size="listQuery.limit" layout="total, sizes, prev, pager, next, jumper" :total="total">
      </el-pagination>
    </div>

    <el-dialog :title="textMap[dialogStatus]" :visible.sync="dialogFormVisible" width="500px">
      <el-form class="small-space" :model="temp" label-position="left" label-width="70px" style='width: 400px; margin-left:50px;'>

        <el-form-item label="状态">
          <el-select class="filter-item" v-model="temp.status" placeholder="请选择">
            <el-option v-for="item in  statusOptions" :key="item" :label="item" :value="item">
            </el-option>
          </el-select>
        </el-form-item>

        <el-form-item label="时间">
          <el-date-picker v-model="temp.timestamp" placeholder="选择日期时间">
          </el-date-picker>
        </el-form-item>

        <el-form-item label="标题">
          <el-input v-model="temp.title"></el-input>
        </el-form-item>

      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible = false">取 消</el-button>
        <!--<el-button type="primary" @click="update">确 定</el-button>-->
        <router-link to="/article/newArticle">
          <el-button @click="edit" type="primary">编辑</el-button>
        </router-link>
      </div>
    </el-dialog>

    <el-dialog title="阅读数统计" :visible.sync="dialogPvVisible" >
      <el-table :data="pvData" border fit highlight-current-row style="width: 100%">
        <el-table-column prop="key" label="渠道"> </el-table-column>
        <el-table-column prop="pv" label="pv"> </el-table-column>
      </el-table>
      <span slot="footer" class="dialog-footer">
        <el-button type="primary" @click="dialogPvVisible = false">确 定</el-button>
      </span>
    </el-dialog>

  </div>
</template>

<script>
  import { fetchList, fetchContent, cancelTop, setTop, setPublish, setDraft, setDelete } from '@/api/article'
  import { Notification } from 'element-ui'

  const calendarTypeOptions = [
    { key: 'CN', display_name: '中国' },
    { key: 'US', display_name: '美国' },
    { key: 'JP', display_name: '日本' },
    { key: 'EU', display_name: '欧元区' }
  ]

  // arr to obj
  const calendarTypeKeyValue = calendarTypeOptions.reduce((acc, cur) => {
    acc[cur.key] = cur.display_name
    return acc
  }, {})

  export default {
    name: 'table_demo',
    data() {
      return {
        list: null,
        total: null,
        listLoading: false,
        listQuery: {
          page: 0,
          category: 'company'
        },
        temp: {
          id: undefined,
          remark: '',
          timestamp: 0,
          title: '',
          type: '',
          status: ''
        },
        importanceOptions: [1, 2, 3],
        calendarTypeOptions,
        sortOptions: [{ label: '按ID升序列', key: '+id' }, { label: '按ID降序', key: '-id' }],
        statusOptions: ['published', 'draft', 'deleted'],
        dialogFormVisible: false,
        dialogStatus: '',
        textMap: {
          update: '编辑',
          create: '创建'
        },
        dialogPvVisible: false,
        pvData: [],
        showAuditor: false,
        tableKey: 0
      }
    },
    mounted() {
      this.getList()
    },
    methods: {
      getList() {
        this.listLoading = true
        // 用来取每一页的所有行业热点
        fetchList(this.listQuery).then(response => {
          console.log(response.data.data)
          this.list = response.data.data
          // this.total = response.data.total
          this.listLoading = false
        })
          .catch(function(error) {
            console.log(error)
          })
      },
      edit() {
        fetchContent(this.temp.id).then(response => {
          console.log(response)
          response.data.id = this.temp.id
          response.data.time = response.data.time * 1000
          response.data.article_status = 'edit'
          this.$store.commit('setConditions', response.data)
        })
          .catch(function(error) {
            console.log(error)
          })
      },
      handleFilter() {
        this.listQuery.page = 1
        this.getList()
      },
      handleSizeChange(val) {
        // this.listQuery.limit = val
        this.getList()
      },
      handleCurrentChange(val) {
        this.listQuery.page = val - 1
        this.getList()
      },
      timeFilter(time) {
        if (!time[0]) {
          this.listQuery.start = undefined
          this.listQuery.end = undefined
          return
        }
        this.listQuery.start = parseInt(+time[0] / 1000)
        this.listQuery.end = parseInt((+time[1] + 3600 * 1000 * 24) / 1000)
      },
      handleCancelTop(id, row) {
        cancelTop(id).then(response => {
          console.log(response)
          console.log(response.data)
          if (response.data === 1) {
            row.status = 'published'
            Notification({
              title: '成功',
              message: '设置成功',
              type: 'success'
            })
          }
        })
      },
      handleTop(id, index, row) {
        setTop(id).then(response => {
          console.log(response)
          console.log(response.data)
          console.log(typeof response.data)
          if (response.data === 1) {
            this.list[index].status = 'top'
            console.log(index, row)
            Notification({
              title: '成功',
              message: '设置成功',
              type: 'success'
            })
          } else {
            Notification.error({
              title: '错误',
              message: '置顶失败'
            })
          }
        })
      },
      handlePublish(id, row) {
        setPublish(id).then(response => {
          console.log(response.data)
          if (response.data === 1) {
            console.log('xixii')
            console.log(row)
            row.status = 'published'
            Notification({
              title: '成功',
              message: '设置成功',
              type: 'success'
            })
          }
        })
      },
      handleDraft(id, row) {
        setDraft(id).then(response => {
          console.log(response.data)
          console.log(typeof response.data)
          if (response.data === 1) {
            console.log(row)
            row.status = 'draft'
            Notification({
              title: '成功',
              message: '设置成功',
              type: 'success'
            })
          }
        })
      },
      handleDelete(id, index) {
        setDelete(id).then(response => {
          if (response.data === 1) {
            this.list.splice(index, 1)
            Notification({
              title: '成功',
              message: '删除成功',
              type: 'success'
            })
          }
        })
      },
      handleUpdate(row, id) {
        console.log(row)
        this.temp = Object.assign({}, row)
        console.log(id)
        this.temp.id = id
        this.dialogStatus = 'update'
        this.dialogFormVisible = true
      },
      create() {
        this.temp.id = parseInt(Math.random() * 100) + 1024
        this.temp.timestamp = +new Date()
        this.list.unshift(this.temp)
        this.dialogFormVisible = false
        this.$notify({
          title: '成功',
          message: '创建成功',
          type: 'success',
          duration: 2000
        })
      },
      update() {
        this.temp.timestamp = +this.temp.timestamp
        for (const v of this.list) {
          if (v.id === this.temp.id) {
            const index = this.list.indexOf(v)
            this.list.splice(index, 1, this.temp)
            break
          }
        }
        this.dialogFormVisible = false
        this.$notify({
          title: '成功',
          message: '更新成功',
          type: 'success',
          duration: 2000
        })
      },
      resetTemp() {
        this.temp = {
          id: undefined,
          importance: 0,
          remark: '',
          timestamp: 0,
          title: '',
          status: 'published',
          type: ''
        }
      }
    }
  }
</script>

