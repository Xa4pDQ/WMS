<template>
  <a-card :bordered="false">
    <!-- 查询区域 -->
    <div class="table-page-search-wrapper">
      <a-form layout="inline" @keyup.enter.native="searchQuery">
        <a-row :gutter="24">
          <a-col :md="6" :sm="8">
            <a-form-item label="交易ID">
              <a-input placeholder="请输入交易ID" v-model="queryParam.transactionId"></a-input>
            </a-form-item>
          </a-col>
          <!--<a-col :md="6" :sm="8">-->
            <!--<a-form-item label="来源总单ID">-->
              <!--<a-input placeholder="请输入来源总单ID" v-model="queryParam.sourceId"></a-input>-->
            <!--</a-form-item>-->
          <!--</a-col>-->
          <template v-if="toggleSearchStatus">
            <!--<a-col :md="6" :sm="8">-->
              <!--<a-form-item label="来源总单编码">-->
                <!--<a-input placeholder="请输入来源总单编码" v-model="queryParam.sourceCode"></a-input>-->
              <!--</a-form-item>-->
            <!--</a-col>-->
            <a-col :md="6" :sm="8">
              <a-form-item label="来源类型">
                <j-dict-select-tag placeholder="请选择来源类型" v-model="queryParam.sourceType" dictCode="source_type"/>
              </a-form-item>
            </a-col>
            <a-col :md="6" :sm="8">
              <a-form-item label="货位ID">
                <a-input placeholder="请输入货位ID" v-model="queryParam.locId"></a-input>
              </a-form-item>
            </a-col>
            <a-col :md="6" :sm="8">
              <a-form-item label="货物ID">
                <a-input placeholder="请输入货物ID" v-model="queryParam.goodsId"></a-input>
              </a-form-item>
            </a-col>
            <a-col :md="12" :sm="16">
              <a-form-item label="创建时间">
                <j-date placeholder="请选择开始日期" class="query-group-cust" v-model="queryParam.createTime_begin"></j-date>
                <span class="query-group-split-cust"></span>
                <j-date placeholder="请选择结束日期" class="query-group-cust" v-model="queryParam.createTime_end"></j-date>
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
      <a-button type="primary" icon="download" @click="handleExportXls('交易表')">导出</a-button>
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
        :scroll="tableScroll"
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

    <wmsTransaction-modal ref="modalForm" @ok="modalFormOk"></wmsTransaction-modal>
  </a-card>
</template>

<script>

  import { JeecgListMixin } from '@/mixins/JeecgListMixin'
  import WmsTransactionModal from './modules/WmsTransactionModal'
  import JDictSelectTag from '@/components/dict/JDictSelectTag.vue'
  import JDate from '@/components/jeecg/JDate.vue'
  import {initDictOptions, filterMultiDictText} from '@/components/dict/JDictSelectUtil'
  import { getAction } from "../../api/manage";

  export default {
    name: "WmsTransactionList",
    mixins:[JeecgListMixin],
    components: {
      JDictSelectTag,
      JDate,
      WmsTransactionModal
    },
    data () {
      return {
        description: '交易表管理页面',
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
            title:'交易ID',
            align:"center",
            dataIndex: 'transactionId'
          },
          {
            title:'交易编码',
            align:"center",
            dataIndex: 'transactionCode'
          },
          {
            title:'移动单ID',
            align:"center",
            dataIndex: 'moveId'
          },
          // {
          //   title:'来源总单ID',
          //   align:"center",
          //   dataIndex: 'sourceId'
          // },
          // {
          //   title:'来源总单编码',
          //   align:"center",
          //   dataIndex: 'sourceCode'
          // },
          // {
          //   title:'来源总单状态',
          //   align:"center",
          //   dataIndex: 'sourceState',
          //   customRender:(text)=>{
          //     if(!text){
          //       return ''
          //     }else{
          //       return filterMultiDictText(this.dictOptions['sourceState'], text+"")
          //     }
          //   }
          // },
          // {
          //   title:'来源总单创建时间',
          //   align:"center",
          //   dataIndex: 'sourceCreateTime',
          //   customRender:function (text) {
          //     return !text?"":(text.length>10?text.substr(0,10):text)
          //   }
          // },
          // {
          //   title:'来源总单创建人',
          //   align:"center",
          //   dataIndex: 'sourceCreateBy'
          // },
          // {
          //   title:'来源细单ID',
          //   align:"center",
          //   dataIndex: 'sourcedtlId'
          // },
          // {
          //   title:'来源细单编码',
          //   align:"center",
          //   dataIndex: 'sourcedtlCode'
          // },
          // {
          //   title:'来源细单状态',
          //   align:"center",
          //   dataIndex: 'sourcedtlState',
          //   customRender:(text)=>{
          //     if(!text){
          //       return ''
          //     }else{
          //       return filterMultiDictText(this.dictOptions['sourcedtlState'], text+"")
          //     }
          //   }
          // },
          // {
          //   title:'来源细单创建时间',
          //   align:"center",
          //   dataIndex: 'sourcedtlCreateTime',
          //   customRender:function (text) {
          //     return !text?"":(text.length>10?text.substr(0,10):text)
          //   }
          // },
          // {
          //   title:'来源细单创建人',
          //   align:"center",
          //   dataIndex: 'sourcedtlCreateBy'
          // },
          {
            title:'库存ID',
            align:"center",
            dataIndex: 'stockId'
          },
          {
            title:'来源类型',
            align:"center",
            dataIndex: 'sourceType',
            customRender:(text)=>{
              if(!text){
                return ''
              }else{
                return filterMultiDictText(this.dictOptions['sourceType'], text+"")
              }
            }
          },
          {
            title:'交易状态',
            align:"center",
            dataIndex: 'transactionState',
            customRender:(text)=>{
              if(!text){
                return ''
              }else{
                return filterMultiDictText(this.dictOptions['transactionState'], text+"")
              }
            }
          },
          {
            title:'区域ID',
            align:"center",
            dataIndex: 'areaId'
          },
          {
            title:'区域编码',
            align:"center",
            dataIndex: 'areaCode'
          },
          {
            title:'区域名称',
            align:"center",
            dataIndex: 'areaName'
          },
          {
            title:'货位ID',
            align:"center",
            dataIndex: 'locId'
          },
          {
            title:'货位编码',
            align:"center",
            dataIndex: 'locCode'
          },
          {
            title:'货位名称',
            align:"center",
            dataIndex: 'locName'
          },
          {
            title:'货物ID',
            align:"center",
            dataIndex: 'goodsId'
          },
          {
            title:'货物编码',
            align:"center",
            dataIndex: 'goodsCode'
          },
          {
            title:'货物名称',
            align:"center",
            dataIndex: 'goodsName'
          },
          {
            title:'货物规格',
            align:"center",
            dataIndex: 'goodsSize'
          },
          {
            title:'货物单位',
            align:"center",
            dataIndex: 'goodsUnit',
            customRender:(text)=>{
              if(!text){
                return ''
              }else{
                return filterMultiDictText(this.dictOptions['goodsUnit'], text+"")
              }
            }
          },
          {
            title:'货物类别',
            align:"center",
            dataIndex: 'goodsType',
            customRender: (text) => {
              if (!text) {
                return ''
              } else {
                let flag = false;
                this.arrayCategoryVal.forEach(function (item) {
                  if (item.value.toString() == text.toString()) {
                    text = item.text;
                    flag = true;
                  }
                })
                return flag ? text : "该货物类别已删除!!!";
              }
            }
          },
          {
            title:'货物花色',
            align:"center",
            dataIndex: 'goodsColor',
            customRender:(text)=>{
              if(!text){
                return ''
              }else{
                return filterMultiDictText(this.dictOptions['goodsColor'], text+"")
              }
            }
          },
          {
            title:'货物批号',
            align:"center",
            dataIndex: 'goodsBatchnumber'
          },
          {
            title:'货物数量',
            align:"center",
            dataIndex: 'goodsQuantity'
          },
          {
            title:'货物等级',
            align:"center",
            dataIndex: 'goodsLevel',
            customRender:(text)=>{
              if(!text){
                return ''
              }else{
                return filterMultiDictText(this.dictOptions['goodsLevel'], text+"")
              }
            }
          },
          {
            title:'创建时间',
            align:"center",
            dataIndex: 'createTime',
            customRender:function (text) {
              return !text?"":(text.length>10?text.substr(0,10):text)
            }
          },
          {
            title:'创建人',
            align:"center",
            dataIndex: 'createBy'
          },
          {
            title: '操作',
            dataIndex: 'action',
            align:"center",
            fixed:"right",
            width:147,
            scopedSlots: { customRender: 'action' }
          }
        ],
        url: {
          list: "/transaction/wmsTransaction/list",
          delete: "/transaction/wmsTransaction/delete",
          deleteBatch: "/transaction/wmsTransaction/deleteBatch",
          exportXlsUrl: "/transaction/wmsTransaction/exportXls",
          importExcelUrl: "transaction/wmsTransaction/importExcel",
          queryCategoryList: "/goods/wmsGoods/queryLastCategoryList",
        },
        dictOptions:{
         sourceState:[],
         sourcedtlState:[],
         sourceType:[],
         transactionState:[],
         goodsUnit:[],
         goodsColor:[],
         goodsLevel:[],
        },
        tableScroll:{x :33*147+50},
        arrayCategoryVal:[],
      }
    },
    computed: {
      importExcelUrl: function(){
        return `${window._CONFIG['domianURL']}/${this.url.importExcelUrl}`;
      }
    },
    created() {
      getAction(this.url.queryCategoryList).then((res) => {
        if (res.success) {
          res.result.list.filter(item => {
            if (item.id != "") {
              this.arrayCategoryVal.push({text: item.categoryName, value: item.categoryId });
            }
          })
        }
      });
    },
    methods: {
      initDictConfig(){
        // initDictOptions('source_state').then((res) => {
        //   if (res.success) {
        //     this.$set(this.dictOptions, 'sourceState', res.result)
        //   }
        // })
        // initDictOptions('source_state').then((res) => {
        //   if (res.success) {
        //     this.$set(this.dictOptions, 'sourcedtlState', res.result)
        //   }
        // })
        initDictOptions('source_type').then((res) => {
          if (res.success) {
            this.$set(this.dictOptions, 'sourceType', res.result)
          }
        })
        initDictOptions('transaction_state').then((res) => {
          if (res.success) {
            this.$set(this.dictOptions, 'transactionState', res.result)
          }
        })
        initDictOptions('goods_unit').then((res) => {
          if (res.success) {
            this.$set(this.dictOptions, 'goodsUnit', res.result)
          }
        })
        initDictOptions('color').then((res) => {
          if (res.success) {
            this.$set(this.dictOptions, 'goodsColor', res.result)
          }
        })
        initDictOptions('goods_level').then((res) => {
          if (res.success) {
            this.$set(this.dictOptions, 'goodsLevel', res.result)
          }
        })
      }
       
    }
  }
</script>
<style scoped>
  @import '~@assets/less/common.less'
</style>