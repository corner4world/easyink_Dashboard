<template>
  <div>
    <RightContainer>
      <template v-slot:search>
        <el-form v-show="showSearch" ref="queryForm" :model="queryParams" :inline="true" label-width="68px">
          <el-form-item prop="title">
            <el-input
              v-model="queryParams.title"
              placeholder="请输入系统模块"
              clearable
              style="width: 240px;"
              size="small"
              @keyup.enter.native="handleQuery"
            />
          </el-form-item>
          <el-form-item prop="operName">
            <el-input
              v-model="queryParams.operName"
              placeholder="请输入操作人员"
              clearable
              style="width: 240px;"
              size="small"
              @keyup.enter.native="handleQuery"
            />
          </el-form-item>
          <el-form-item prop="businessType">
            <el-select
              v-model="queryParams.businessType"
              placeholder="请选择操作类型"
              clearable
              size="small"
              style="width: 240px"
            >
              <el-option
                v-for="dict in typeOptions"
                :key="dict.dictValue"
                :label="dict.dictLabel"
                :value="dict.dictValue"
              />
            </el-select>
          </el-form-item>
          <el-form-item>
            <DatePicker
              style="width:240px"
              align="right"
              type="daterange"
              :time.sync="dateRange"
              value-format="yyyy-MM-dd"
            />
          </el-form-item>
          <el-form-item>
            <el-button
              v-preventReClick="200"
              type="primary"
              :loading="searchButtonLoading"
              @click="()=>{
                searchButtonLoading = true;
                handleQuery()
              }"
            >查询</el-button>
            <el-button
              v-preventReClick="200"
              class="btn-reset"
              :loading="resetButtonLoading"
              @click="()=>{
                resetButtonLoading = true;
                resetQuery()
              }"
            >重置</el-button>
          </el-form-item>
        </el-form>
      </template>
      <template v-slot:operate-btn>
        <el-row :gutter="10" class="mb8">
          <el-col :span="1.5">
            <el-button
              v-hasPermi="['monitor:operlog:remove']"
              size="mini"
              @click="handleDelete"
            >批量删除</el-button>
          </el-col>
          <el-col :span="1.5">
            <el-button
              v-hasPermi="['monitor:operlog:remove']"
              size="mini"
              @click="handleClean"
            >一键清空</el-button>
          </el-col>
          <!-- <el-col :span="1.5">
            <el-button
              type="warning"
              icon="el-icon-download"
              size="mini"
              @click="handleExport"
              v-hasPermi="['system:config:export']"
            >导出</el-button>
          </el-col> -->
        </el-row>
      </template>
      <template v-slot:data>
        <el-table v-loading="loading" :data="list" @selection-change="handleSelectionChange">
          <el-table-column type="selection" width="55" align="center" />
          <el-table-column label="日志编号" align="center" prop="operId" />
          <el-table-column label="系统模块" align="center" prop="title" />
          <el-table-column label="操作类型" align="center" prop="businessType" :formatter="typeFormat" />
          <el-table-column label="请求方式" align="center" prop="requestMethod" />
          <el-table-column label="操作人员" align="center" prop="operName" />
          <el-table-column label="主机" align="center" prop="operIp" width="130" :show-overflow-tooltip="true" />
          <el-table-column label="操作地点" align="center" prop="operLocation" :show-overflow-tooltip="true" />
          <el-table-column label="操作状态" align="center" prop="status" :formatter="statusFormat" />
          <el-table-column label="操作日期" align="center" prop="operTime" width="180">
            <template slot-scope="scope">
              <span>{{ parseTime(scope.row.operTime) }}</span>
            </template>
          </el-table-column>
          <el-table-column label="操作" align="center" class-name="small-padding fixed-width">
            <template slot-scope="scope">
              <el-button
                v-hasPermi="['monitor:operlog:query']"
                size="mini"
                type="text"
                @click="handleView(scope.row,scope.index)"
              >
                详情
              </el-button>
            </template>
          </el-table-column>
        </el-table>
        <pagination
          v-show="total>0"
          :total="total*1"
          :page.sync="queryParams.pageNum"
          :limit.sync="queryParams.pageSize"
          :select-data-len="ids.length"
          :disabled="loading"
          @pagination="getList"
        />
      </template>
    </RightContainer>
    <!-- 操作日志详细 -->
    <el-dialog title="操作日志详情" :visible.sync="open" width="700px" append-to-body>
      <el-form ref="form" :model="form" label-width="100px" size="mini">
        <el-row>
          <el-col :span="12">
            <el-form-item label="操作模块：">{{ form.title }} / {{ typeFormat(form) }}</el-form-item>
            <el-form-item
              label="登录信息："
            >{{ form.operName }} / {{ form.operIp }} / {{ form.operLocation }}</el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="请求地址：">{{ form.operUrl }}</el-form-item>
            <el-form-item label="请求方式：">{{ form.requestMethod }}</el-form-item>
          </el-col>
          <el-col :span="24">
            <el-form-item label="操作方法：">{{ form.method }}</el-form-item>
          </el-col>
          <el-col :span="24">
            <el-form-item label="请求参数：">{{ form.operParam }}</el-form-item>
          </el-col>
          <el-col :span="24">
            <el-form-item label="返回参数：">{{ form.jsonResult }}</el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="操作状态：">
              <div v-if="form.status === 0">正常</div>
              <div v-else-if="form.status === 1">失败</div>
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="操作时间：">{{ parseTime(form.operTime) }}</el-form-item>
          </el-col>
          <el-col :span="24">
            <el-form-item v-if="form.status === 1" label="异常信息：">{{ form.errorMsg }}</el-form-item>
          </el-col>
        </el-row>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="open = false">关 闭</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import DatePicker from '@/components/DatePicker';
import moment from 'moment';
import { list, delOperlog, cleanOperlog, exportOperlog } from '@/api/monitor/operlog';
import { PAGE_LIMIT } from '@/utils/constant/index';
import RightContainer from '@/components/RightContainer';
import loadingMixin from '@/mixin/loadingMixin';
export default {
  name: 'Operlog',
  components: { RightContainer, DatePicker },
  mixins: [loadingMixin],
  data() {
    return {
      // 遮罩层
      loading: true,
      // 选中数组
      ids: [],
      // 非多个禁用
      multiple: true,
      // 显示搜索条件
      showSearch: true,
      // 总条数
      total: 0,
      // 表格数据
      list: [],
      // 是否显示弹出层
      open: false,
      // 类型数据字典
      typeOptions: [],
      // 类型数据字典
      statusOptions: [],
      // 日期范围
      dateRange: [moment().subtract(1, 'month').format('YYYY-MM-DD'), moment().format('YYYY-MM-DD')],
      // 表单参数
      form: {},
      // 查询参数
      queryParams: {
        pageNum: 1,
        pageSize: PAGE_LIMIT,
        title: undefined,
        operName: undefined,
        businessType: undefined,
        status: undefined
      }
    };
  },
  created() {
    this.getList();
    this.getDicts('sys_oper_type').then(response => {
      this.typeOptions = response.data;
    });
    this.getDicts('sys_common_status').then(response => {
      this.statusOptions = response.data;
    });
  },
  methods: {
    /** 查询登录日志 */
    getList() {
      this.loading = true;
      list(this.addDateRange(this.queryParams, this.dateRange)).then(response => {
        this.list = response.rows;
        this.total = response.total;
        this.loading = false;
      }).finally(() => {
        this.modifyButtonStatus();
      });
    },
    // 操作日志状态字典翻译
    statusFormat(row, column) {
      return this.selectDictLabel(this.statusOptions, row.status);
    },
    // 操作日志类型字典翻译
    typeFormat(row, column) {
      return this.selectDictLabel(this.typeOptions, row.businessType);
    },
    /** 搜索按钮操作 */
    handleQuery() {
      this.queryParams.pageNum = 1;
      this.getList();
    },
    /** 重置按钮操作 */
    resetQuery() {
      this.dateRange = [moment().subtract(1, 'month').format('YYYY-MM-DD'), moment().format('YYYY-MM-DD')];
      this.resetForm('queryForm');
      this.handleQuery();
    },
    // 多选框选中数据
    handleSelectionChange(selection) {
      this.ids = selection.map(item => item.operId);
      this.multiple = !selection.length;
    },
    /** 详细按钮操作 */
    handleView(row) {
      this.open = true;
      this.form = row;
    },
    /** 删除按钮操作 */
    handleDelete(row) {
      if (!this.ids.length) {
        this.msgWarn('请至少选择一条日志');
        return false;
      }
      const operIds = row.operId || this.ids;
      this.confirmModal({
        msg: '是否确认删除日志编号为"' + operIds + '"的数据项?'
      }, () => {
        return delOperlog(operIds)
          .then(() => {
            this.getList();
            this.msgSuccess('删除成功');
          }).catch(function() {});
      });
    },
    /** 清空按钮操作 */
    handleClean() {
      this.confirmModal({
        msg: '是否确认清空所有操作日志数据项?'
      }, () => {
        return cleanOperlog()
          .then(() => {
            this.getList();
            this.msgSuccess('清空成功');
          }).catch(function() {});
      });
    },
    /** 导出按钮操作 */
    handleExport() {
      const queryParams = this.queryParams;
      this.confirmModal({
        msg: '是否确认导出所有操作日志数据项?'
      }, () => {
        return exportOperlog(queryParams)
          .then(response => {
            this.download(response.msg);
          }).catch(function() {});
      });
    }
  }
};
</script>

