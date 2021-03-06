<template>
  <a-card :bordered="false">
    <!-- 查询区域 -->
    <div class="table-page-search-wrapper">
      <a-form layout="inline" @submit.prevent="searchQuery">
        <a-row :gutter="24">

          <a-col :md="6" :sm="12">
            <a-form-item label="老人姓名">
              <a-input placeholder="请输入老人姓名" v-model="queryParam.customerName"></a-input>
            </a-form-item>
          </a-col>

          <a-col :md="6" :sm="8">
            <a-form-item label="申请时间">
              <a-input placeholder="请选择申请时间" v-model="queryParam.createTime"></a-input>
            </a-form-item>
          </a-col>

          <template v-if="toggleSearchStatus">
            <a-col :md="6" :sm="8">
              <a-form-item label="健康状态">
                <a-select v-model="queryParam.customerStatus" placeholder="请选择健康状态">
                  <a-select-option value="">请选择</a-select-option>
                  <a-select-option value="1">健全</a-select-option>
                  <a-select-option value="2">残疾</a-select-option>
                </a-select>
              </a-form-item>
            </a-col>
          </template>

          <a-col :md="6" :sm="8">
            <span style="float: left;overflow: hidden;" class="table-page-search-submitButtons">
              <a-button type="primary" @click="searchQuery" icon="search">查询</a-button>
              <a-button type="primary" @click="searchReset" icon="reload" style="margin-left: 8px">重置</a-button>
              <a @click="handleToggleSearch" style="margin-left: 8px">
                {{ toggleSearchStatus ? '收起' : '展开' }}
                <a-icon :type="toggleSearchStatus ? 'up' : 'down'" />
              </a>
            </span>
          </a-col>
        </a-row>
      </a-form>
    </div>

    <!-- 操作按钮区域 -->
    <div class="table-operator" style="border-top: 5px">
      <!-- <a-button @click="handleActionAdd" type="primary" icon="plus">添加老人</a-button> -->
      <a-dropdown v-if="selectedRowKeys.length > 0">
        <a-menu slot="overlay" @click="handleMenuClick">
          <a-menu-item key="1">
            <a-icon type="delete" @click="batchDel" />
            删除
          </a-menu-item>
          <a-menu-item key="2">
            <a-icon type="lock" />
            导出
          </a-menu-item>

        </a-menu>
        <a-button style="margin-left: 8px">
          批量操作
          <a-icon type="down" />
        </a-button>
      </a-dropdown>
    </div>

    <!-- table区域-begin -->
    <div>
      <div class="ant-alert ant-alert-info" style="margin-bottom: 16px;">
        <i class="anticon anticon-info-circle ant-alert-icon"></i>已选择&nbsp;<a style="font-weight: 600">{{ selectedRowKeys.length }}</a>项&nbsp;&nbsp;
        <a style="margin-left: 24px" @click="onClearSelected">清空</a>
      </div>

      <a-table
        ref="table"
        bordered
        size="middle"
        rowKey="id"
        :columns="columns"
        :dataSource="dataSource"
        :pagination="ipagination"
        :loading="loading"
        :rowSelection="{selectedRowKeys: selectedRowKeys, onChange: onSelectChange}"
        @change="handleTableChange"
      >
        <span slot="action" slot-scope="text, record">
          <a @click="handleAction(record)">审批</a>

          <a-divider type="vertical" />

          <a-dropdown>
            <a class="ant-dropdown-link">
              更多 <a-icon type="down" />
            </a>
            <a-menu slot="overlay">
              <a-menu-item>
                <a-popconfirm title="确定删除吗?" @confirm="() => handleDelete(record.id)">
                  <a>删除</a>
                </a-popconfirm>
              </a-menu-item>
            </a-menu>
          </a-dropdown>
        </span>
      </a-table>
    </div>
    <password-modal ref="passwordmodal" @ok="passwordModalOk"></password-modal>
    <package-modal ref="packageModal"></package-modal>
  </a-card>
</template>

<script>
  import { frozenBatch } from '@/api/system'
  import { ProListMixin } from '@/utils/mixins/ProListMixin'
  import PasswordModal from './modules/PasswordModal'
  import PackageModal from './modules/PackageModal'

  export default {
    name: 'ElderApproval',
    mixins: [ ProListMixin ],
    components: {
      PasswordModal,
      PackageModal
    },
    data() {
      return {
        description: '这是用户管理页面',
        queryParam: {
          customerName: '',
          customerStatus: '',
          contact: '',
          mobile: '',
          packageName: '',
          accountStatus: '',
          createTime: ''
        },
        columns: [
          {
            title: '老人姓名',
            align: 'center',
            dataIndex: 'name',
            width: 120
          },
          {
            title: '年龄',
            align: 'center',
            width: 120,
            sorter: true,
            dataIndex: 'age'
          },
          {
            title: '健康状态',
            align: 'center',
            width: 100,
            dataIndex: 'healthStatus',
            customRender (t) {
              const statusMap = { 0: '健全', 1: '残疾'}
              return statusMap[t] || ''
            }
          },
          {
            title: '联系人',
            align: 'center',
            width: 100,
            dataIndex: 'contactName'
          },
          {
            title: '联系人电话',
            align: 'center',
            width: 100,
            dataIndex: 'mobile'
          },
          {
            title: '申请时间',
            align: 'center',
            width: 150,
            sorter: true,
            dataIndex: 'applyTime'
          },
          {
            title: '审批状态',
            align: 'center',
            width: 100,
            dataIndex: 'status',
            customRender (t) {
              const statusMap = {1: '审批通过', 2: '审批否决', 3: '驳回审批', 0: '等待审批'}
              return statusMap[t] || ''
            }
          },
          {
            title: '操作',
            dataIndex: 'action',
            scopedSlots: { customRender: 'action' },
            align: 'center',
            width: 170
          }
        ],
        url: {
          list: '/parent/getApproveParent',
          delete: '/mp/delConsumerDetail',
          deleteBatch: '/sys/user/deleteBatch'
        },
        path: {
          detail: this.$routePath.CPC_ELDER_APPROVE_DETAIL
        }
      }
    },
    methods: {
      getAvatarView (avatar) {
        // return this.url.imgServer + '/' + avatar
        return avatar
      },

      batchFrozen (status) {
        if (this.selectedRowKeys.length <= 0) {
          this.$message.warning('请选择一条记录！')
          return false
        } else {
          let ids = ''
          let that = this
          let isAdmin = false
          that.selectionRows.forEach(function (row) {
            if (row.username == 'admin') {
              isAdmin = true
            }
          });
          if (isAdmin) {
            that.$message.warning('管理员账号不允许此操作,请重新选择！')
            return
          }
          that.selectedRowKeys.forEach(function (val) {
            ids += val + ','
          })
          that.$confirm({
            title: '确认操作',
            content: '是否' + (status == 1 ? '解冻' : '冻结') + '选中账号?',
            onOk: function () {
              frozenBatch({ids: ids, status: status}).then((res) => {
                if (that.$isAjaxSuccess(res.code)) {
                  that.$message.success(res.message)
                  that.loadData()
                  that.onClearSelected()
                } else {
                  that.$message.warning(res.message)
                }
              })
            }
          })
        }
      },
      handleMenuClick (e) {
        if (e.key == 1) {
          this.batchDel()
        } else if (e.key == 2) {
          this.batchFrozen(2)
        } else if (e.key == 3) {
          this.batchFrozen(1)
        }
      },
      handleFrozen (id, status, username) {
        //TODO 后台校验管理员角色
        if ('admin' == username) {
          this.$message.warning('管理员账号不允许此操作！')
          return
        }
        frozenBatch({ids: id, status: status}).then((res) => {
          if (this.$isAjaxSuccess(res.code)) {
            this.$message.success(res.message)
            this.loadData()
          } else {
            this.$message.warning(res.message)
          }
        });
      },
      handleChangePassword (username) {
        this.$refs.passwordmodal.show(username)
      },
      passwordModalOk () {
        //TODO 密码修改完成
      },
      handlePackage () {
        this.$refs.packageModal.add()
        this.$refs.packageModal.title = '套餐'
        this.$refs.packageModal.disableSubmit = false
      }
    }

  }
</script>
