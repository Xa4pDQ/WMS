<template>
  <a-card :bordered="false">
    <!-- 查询区域 -->
    <div class="table-page-search-wrapper">
      <a-form layout="inline" @keyup.enter.native="searchQuery">
        <a-row :gutter="24">
          <a-col :md="6" :sm="8">
            <a-form-item label="仓库名称">
              <a-input placeholder="请输入仓库名称" v-model="queryParam.warehouseName"></a-input>
            </a-form-item>
          </a-col>
          <a-col :md="6" :sm="8">
            <a-form-item label="仓库编码">
              <a-input placeholder="请输入仓库编码" v-model="queryParam.warehouseCode"></a-input>
            </a-form-item>
          </a-col>
          <a-col :md="6" :sm="8">
            <a-form-item label="仓库地点">
              <a-input placeholder="请输入仓库地点" v-model="queryParam.warehouseLoc"></a-input>
            </a-form-item>
          </a-col>
          <template v-if="toggleSearchStatus">
            <a-col :md="6" :sm="8">
              <a-form-item label="仓库容量">
                <a-input placeholder="请输入仓库容量" v-model="queryParam.warehouseVolume"></a-input>
              </a-form-item>
            </a-col>
            <a-col :md="6" :sm="8">
              <a-form-item label="仓库类型">
                <!--<a-input placeholder="请输入仓库类型" v-model="queryParam.warehouseType"  dictCode="warehouse_type"></a-input>-->
                <j-dict-select-tag placeholder="请输入仓库类型" v-model="queryParam.warehouseType" dictCode="warehouse_type"/>
              </a-form-item>
            </a-col>
            <a-col :md="6" :sm="8">
              <a-form-item label="仓库启用状态">
                <j-dict-select-tag placeholder="请选择仓库启用状态" v-model="queryParam.isUse" dictCode="use_type"/>
              </a-form-item>
            </a-col>
            <a-col :md="6" :sm="8">
              <a-form-item label="仓库管理者">
                <a-input placeholder="请输入仓库管理者" v-model="queryParam.warehouseManager"></a-input>
              </a-form-item>
            </a-col>
          </template>
          <a-col :md="6" :sm="8" >
            <span style="float: left;overflow: hidden;" class="table-page-search-submitButtons">
              <a-button type="primary" @click="searchQuery" icon="search">查询</a-button>
              <a-button type="primary" @click="searchReset" icon="reload" style="margin-left: 8px">重置</a-button>
              <a @click="handleToggleSearch" style="margin-left: 8px">
                {{ toggleSearchStatus ? '收起' : '展开' }}
                <a-icon :type="toggleSearchStatus ? 'up' : 'down'"/>
              </a>
            </span>
          </a-col>

        </a-row>
      </a-form>
    </div>
    <!-- 查询区域-END -->
    
    <!-- 操作按钮区域 -->
    <div class="table-operator">
      <a-button @click="handleAdd" type="primary" icon="plus">新增</a-button>
      <a-button type="primary" icon="download" @click="handleExportXls('仓库管理')">导出</a-button>
      <a-upload name="file" :showUploadList="false" :multiple="false" :headers="tokenHeader" :action="importExcelUrl" @change="handleImportExcel">
        <a-button type="primary" icon="import">导入</a-button>
      </a-upload>
      <a-dropdown v-if="selectedRowKeys.length > 0">
        <a-menu slot="overlay">
          <a-menu-item key="1" @click="batchDel"><a-icon type="delete"/>删除</a-menu-item>
        </a-menu>
        <a-button style="margin-left: 8px"> 批量操作 <a-icon type="down" /></a-button>
      </a-dropdown>
    </div>

    <!-- table区域-begin -->
    <div>
      <div class="ant-alert ant-alert-info" style="margin-bottom: 16px;">
        <i class="anticon anticon-info-circle ant-alert-icon"></i> 已选择 <a style="font-weight: 600">{{ selectedRowKeys.length }}</a>项
        <a style="margin-left: 24px" @click="onClearSelected">清空</a>
      </div>

      <a-table
        ref="table"
        size="middle"
        bordered
        rowKey="id"
        :columns="columns"
        :dataSource="dataSource"
        :pagination="ipagination"
        :loading="loading"
        :rowSelection="{fixed:true,selectedRowKeys: selectedRowKeys, onChange: onSelectChange}"
        
        @change="handleTableChange">

        <template slot="htmlSlot" slot-scope="text">
          <div v-html="text"></div>
        </template>
        <template slot="imgSlot" slot-scope="text">
          <span v-if="!text" style="font-size: 12px;font-style: italic;">无此图片</span>
          <img v-else :src="getImgView(text)" height="25px" alt="图片不存在" style="max-width:80px;font-size: 12px;font-style: italic;"/>
        </template>
        <template slot="fileSlot" slot-scope="text">
          <span v-if="!text" style="font-size: 12px;font-style: italic;">无此文件</span>
          <a-button
            v-else
            :ghost="true"
            type="primary"
            icon="download"
            size="small"
            @click="uploadFile(text)">
            下载
          </a-button>
        </template>

        <span slot="action" slot-scope="text, record">
          <a @click="handleEdit(record)">编辑</a>

          <a-divider type="vertical" />
          <a-dropdown>
            <a class="ant-dropdown-link">更多 <a-icon type="down" /></a>
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

    <wmsWarehouse-modal ref="modalForm" @ok="modalFormOk"></wmsWarehouse-modal>
  </a-card>
</template>

<script>

  import { JeecgListMixin } from '@/mixins/JeecgListMixin'
  import WmsWarehouseModal from './modules/WmsWarehouseModal'
  import JDictSelectTag from '@/components/dict/JDictSelectTag.vue'
  import {initDictOptions, filterMultiDictText} from '@/components/dict/JDictSelectUtil'

  export default {
    name: "WmsWarehouseList",
    mixins:[JeecgListMixin],
    components: {
      JDictSelectTag,
      WmsWarehouseModal
    },
    data () {
      return {
        description: '仓库管理管理页面',
        // 表头
        columns: [
          {
            title: '#',
            dataIndex: '',
            key:'rowIndex',
            width:60,
            align:"center",
            customRender:function (t,r,index) {
              return parseInt(index)+1;
            }
          },
          {
            title:'仓库名称',
            align:"center",
            dataIndex: 'warehouseName'
          },
          {
            title:'仓库编码',
            align:"center",
            dataIndex: 'warehouseCode'
          },
          {
            title:'仓库地点',
            align:"center",
            dataIndex: 'warehouseLoc'
          },
          {
            title:'仓库容量',
            align:"center",
            dataIndex: 'warehouseVolume'
          },
          {
            title:'仓库类型',
            align:"center",
            dataIndex: 'warehouseType',
            customRender:(text)=>{
              if(!text){
                return ''
              }else{
                return filterMultiDictText(this.dictOptions['warehouseType'], text+"")
              }
            }
          },
          {
            title:'仓库启用状态',
            align:"center",
            dataIndex: 'isUse',
            customRender:(text)=>{
              if(!text){
                return ''
              }else{
                return filterMultiDictText(this.dictOptions['isUse'], text+"")
              }
            }
          },
          {
            title:'仓库管理者',
            align:"center",
            dataIndex: 'warehouseManager'
          },
          {
            title:'备注',
            align:"center",
            dataIndex: 'memo'
          },
          {
            title:'创建时间',
            align:"center",
            dataIndex: 'createTime',
            customRender:(text)=>{
              if(!text){
                return ''
              }else{
                return text.split(" ")[0]
              }
            }
          },
          {
            title: '操作',
            dataIndex: 'action',
            align:"center",
            scopedSlots: { customRender: 'action' }
          }
        ],
        url: {
          list: "/warehouse/wmsWarehouse/list",
          delete: "/warehouse/wmsWarehouse/delete",
          deleteBatch: "/warehouse/wmsWarehouse/deleteBatch",
          exportXlsUrl: "/warehouse/wmsWarehouse/exportXls",
          importExcelUrl: "warehouse/wmsWarehouse/importExcel",
        },
        dictOptions:{
         warehouseType:[],
         isUse:[],
        },
      }
    },
    computed: {
      importExcelUrl: function(){
        return `${window._CONFIG['domianURL']}/${this.url.importExcelUrl}`;
      }
    },
    methods: {
      initDictConfig(){
        initDictOptions('warehouse_type').then((res) => {
          if (res.success) {
            this.$set(this.dictOptions, 'warehouseType', res.result)
          }
        })
        initDictOptions('use_type').then((res) => {
          if (res.success) {
            this.$set(this.dictOptions, 'isUse', res.result)
          }
        })
      },
       
    }
  }
</script>
<style scoped>
  @import '~@assets/less/common.less'
</style>