<template>
  <div class="dic-container">
    <el-tabs type="border-card">
      <el-form :inline="true" :model="searchParam" class="demo-form-inline head-form">
          <el-form-item label="字典名称">
            <el-input clearable v-model="searchParam.title" size="small" placeholder="请输入字典名称"
                      style="width: 200px"/>
          </el-form-item>
          <el-form-item>
            <el-button :loading="tableLoading" @click="search" size="small" icon="el-icon-search" type="primary">搜索
            </el-button>
            <el-button @click="dicInfo()" size="small" icon="el-icon-plus" type="primary">添加字典
            </el-button>
            <el-button icon="el-icon-date" @click="batchDel()" size="small" class="add">批量删除</el-button>
            <el-button icon="el-icon-setting" @click="batchCache()" size="small" class="add">重置缓存</el-button>
            <el-button @click="initSearchParam();search()" icon="el-icon-refresh-right"
                       type="primary" size="small"
                       style="background-color: #fff; border: 1px solid #ccc ; color: #666">清空
            </el-button>
          </el-form-item>
        </el-form>
        <el-table stripe ref="table" :height="tableHeight" v-loading="tableLoading"
                  :data="dicList" border row-key="id">
          <el-table-column show-overflow-tooltip type="selection" align="center"/>
          <el-table-column show-overflow-tooltip prop="id" label="ID" align="center" width="80"/>
          <el-table-column show-overflow-tooltip prop="title" label="字典标题" align="center"/>
          <el-table-column show-overflow-tooltip prop="tag" label="内部标签" align="center"/>
          <el-table-column show-overflow-tooltip prop="typeName" label="字典类型" align="center"/>
          <el-table-column show-overflow-tooltip prop="status" label="状态" align="center" width="120">
            <template slot-scope="{row}">
              {{ row.status === 1 ? '正常' : '禁用' }}
              <el-switch v-if="!isAdmin||btnFunc.some(btn=>btn=='status')" v-model="row.status" :active-value="1"
                         :inactive-value="2" size="small"
                         @change="setStatus(row)"/>
            </template>
          </el-table-column>
          <el-table-column show-overflow-tooltip prop="note" label="备注" align="center"/>
          <el-table-column show-overflow-tooltip prop="sort" label="排序" align="center" width="100"/>
          <el-table-column show-overflow-tooltip prop="createUser" label="添加人" align="center" width="150"/>
          <el-table-column show-overflow-tooltip prop="createTime" label="添加时间" align="center" width="180"/>
          <el-table-column v-if="isAdmin||(btnFunc.filter(value=>value=='edit'||value=='drop').length>0)" label="操作" width="120" align="center" fixed="right">
            <template slot-scope="scoped">
              <el-row>
                <el-tooltip v-if="isAdmin||btnFunc.some(btn=>btn=='edit')" class="item" effect="dark" content="编辑" placement="bottom">
                  <el-button type="primary" size="mini" icon="el-icon-edit" @click="dicInfo(scoped.row)"/>
                </el-tooltip>
                <el-tooltip v-if="!isAdmin||btnFunc.some(btn=>btn=='drop')" class="item" effect="dark" content="删除" placement="bottom">
                  <el-button :loading="delBtnLoading" type="danger" size="mini" class="el-icon-delete" @click="dicDel(scoped.row)"/>
                </el-tooltip>
              </el-row>
            </template>
          </el-table-column>
        </el-table>
        <!-- 分页 -->
        <div  class="page-current">
          <el-pagination
            background
            :current-page="currentPage"
            :page-size="pageSize"
            prev-text="上一页"
            next-text="下一页"
            layout="total, prev, pager, next, jumper"
            style="display: inline-block;"
            :total="count"
            @current-change="handlePageChange"/>
        </div>
    </el-tabs>

    <!-- 编辑表单 -->
    <el-dialog :close-on-press-escape="false" :close-on-click-modal="false" center :visible.sync="addDialogVisible" :title="dialogTitle+'字典'">
      <editForm ref="editForm" :infoForm="infoForm" :typeList="typeList" style="width: 100%" @submit="submit" @closeWin="closeWin"></editForm>
    </el-dialog>

  </div>
</template>

<script>
  import dicApi from '@/api/system/dic'
  import editForm from './edit'

  export default {
    name: "dic",
    data() {
      return {
        tableLoading: false,
        // 当前页
        currentPage: 1,
        // 总条数
        count: 0,
        // 每页条数
        pageSize: 10,
        // 显示编辑角色弹框
        addBtnLoading: false,
        addDialogVisible: false,
        delBtnLoading: false,
        searchParam: {
          title: '',
        },
        // 状态
        status: [{
          value: 1,
          label: '正常'
        }, {
          value: 2,
          label: '禁用'
        }],
        dicList: [],
        typeList: [],
        infoForm: {
          title: '',
          status: 1,
          sort: 125
        },
        loading: false,
        tableHeight: document.body.clientHeight - 245,
        btnFunc: [],
        isAdmin: false,
        dialogTitle: "添加",
      }
    },
    components: {editForm},
    created() {
      const {btnFunc, isAdmin} = this.$store.state.user
      this.btnFunc = btnFunc
      this.isAdmin = isAdmin
      this.pageSize = parseInt((document.body.clientHeight - 344) / 50)
    },
    mounted() {
      // 开启加载动画
      this.tableLoading = true
      // 获取数据列表
      this.getDicList()
      // 获取字典类型
      this.getDicTypeList()
    },
    watch: {

    },
    methods: {
      // 初始化搜索条件
      initSearchParam() {
        this.searchParam = {
          name: '',
        }
      },
      // 获取数据列表
      getDicList() {
        let searchParam = JSON.parse(JSON.stringify(this.searchParam))
        searchParam.pageIndex = this.currentPage
        searchParam.pageSize = this.pageSize
        dicApi.dicList(searchParam).then(res => {
          this.dicList = res.data.records
          this.count = res.data.total
          this.tableLoading = false
        }).catch(e => {
          this.dicList = []
          this.tableLoading = false
        })
      },
      // 获取编辑数据
      dicInfo(data){
        // 初始化表单
        this.initInfoForm()
        // 显示弹窗
        this.addDialogVisible=true
        // 设置标题
        if (data == null) {
          this.dialogTitle = "添加";
        } else {
        this.dialogTitle = "编辑";
        }
        this.$nextTick(function () {
          try {
            dicApi.dicInfo({id: data.id}).then(res => {
              this.infoForm = res.data
              this.$refs.editForm.$refs['infoForm'].resetFields()
            }).catch(e => {
            })
          } catch (e) {

          }
        })
      },
      // 初始化表单
      initInfoForm() {
        this.infoForm = {
          title: '',
          status: 1,
          sort: 125
        }
      },
      // 删除记录
      dicDel(data) {
        this.$confirm('您确定要删除字典吗？', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.delBtnLoading = true
          dicApi.dicDel({id: data.id}).then(res => {
            this.$message({
              type: 'success',
              message: '删除成功！'
            })
            this.delBtnLoading = false
            this.tableLoading = true
            this.getDicList()
          }).catch(e => {
            this.delBtnLoading = false
          })
        }).catch(() => {
          this.$message({
            type: 'info',
            message: '已取消删除'
          })
        })
      },
      // 获取字典类型
      getDicTypeList() {
        dicApi.getDicTypeList().then(res => {
          this.typeList = res.data ? res.data : []
        }).catch(e => {
          this.typeList = []
        })
      },
      // 设置状态
      setStatus(data) {
        var content = ''
        if (data.status === 1) {
          content = '确定要启用吗？'
        } else {
          content = '确定要禁用吗？'
        }
        this.$confirm(content, '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          let params = {
            status: data.status,
            id: data.id
          }
          this.tableLoading = true
          dicApi.dicStatus(params).then(res => {
            this.$message({
              type: 'success',
              message: '操作成功！'
            })
            this.getDicList()
          }).catch(e => {
            let dicList = JSON.parse(JSON.stringify(this.dicList))
            for (let i = 0; i < dicList.length; i++) {
              if (dicList[i].id == data.id) {
                dicList[i].status = data.status == 1 ? 2 : 1
              }
            }
            this.dicList = Object.assign([], dicList)
            this.tableLoading = false
          })
        }).catch(() => {
          let dicList = JSON.parse(JSON.stringify(this.dicList))
          for (let i = 0; i < dicList.length; i++) {
            if (dicList[i].id == data.id) {
              dicList[i].status = data.status == 1 ? 2 : 1
            }
          }
          this.dicList = Object.assign([], dicList)
          this.$message({
            type: 'info',
            message: '已取消该操作！'
          })
        })
      },
      handlePageChange(val) {
        this.currentPage = val
        this.tableLoading = true
        this.$refs.table.bodyWrapper.scrollTop = 0;
        this.getDicList()
      },
      // 搜索
      search() {
        this.currentPage = 1
        this.tableLoading = true
        this.$refs.table.bodyWrapper.scrollTop = 0;
        this.getDicList()
      },
      // 提交表单后逻辑处理
      submit(type) {
        if (type == 0) {
          // 添加
          this.search()
        } else {
          // 编辑
          this.tableLoading=true
          this.getDicList()
          this.$refs.table.bodyWrapper.scrollTop = 0;
          this.addDialogVisible = false
        }
      },
      // 关闭窗体
      closeWin() {
        console.log("关闭")
        // 调用初始化表单方法
        this.initInfoForm()
        // 设置弹窗不可见
        this.addDialogVisible = false
      }
    }
  }
</script>

<style lang="less">
  .dic-container {
    width: 100%;

    .page-current {
      text-align: center;
      padding: 15px 5px;
      padding-bottom: 0px;

      .sure-page-btn {
        display: inline-block;
        height: 28px;
        line-height: 0;
        width: 46px;
        padding: 0;
        vertical-align: top;
      }
    }
  }

</style>
