<template>
  <div class="de_gas" ref="nx_ems">
    <Card cardStyles="margin-bottom:16px;padding:12px 32px;">
      <div class="div__tool-wrap">
        <el-row :gutter="40">
          <el-col :span="6">
            <div class="select__wrap">
              <div>选择时间：</div>
              <div>
                <el-date-picker
                  v-model="time"
                  align="right"
                  type="month"
                  placeholder="选择日期"
                  size="small"
                  :clearable="false"
                  :editable="false"
                  :picker-options="pickerOptions"
                ></el-date-picker>
              </div>
            </div>
          </el-col>
          <el-col :span="10">
            <SelOrg
              @selectArear="selectArear"
              @selectPlant="selectPlant"
              :showDevice="false"
              :showSystem="false"
              :showUnit="false"
              :showAll="false"
            />
          </el-col>
          <el-col :span="7">
            <el-button
            v-if="desulphurize_add"
              @click="onAdd"
              style="position:relative;right:-20px"
              type="primary"
              size="mini"
            >新增</el-button>
            <BtnList
              style="width:auto;display:inline-block"
              resetContent="导出"
              btnStyle="textAlign:left;margin-left:32px"
              @check="checkList"
              @reset="onExport"
            />
          </el-col>
        </el-row>
      </div>
      <div class="div__content-wrap">
        <layoutTable>
          <span slot="title" class="title">
            <u class="u">{{ year }}</u> 年
            <u class="u">{{ month }}</u> 月
            <u class="u">{{ area_plant }}</u> 脱硝装置耗用统计表
          </span>
          <div
            class="content-table"
            ref="zcurd"
            slot="table"
            v-if="tableShow"
            v-loading="tableLoading"
          >
            <Zcurd
              ref="curd"
              :projectCode="projectCode"
              :year="year"
              :month="month"
              :height="tableHeight"
              :unitList="unitList"
              :tableData="tableData"
            />
          </div>
        </layoutTable>
      </div>
    </Card>
  </div>
</template>

<script>
import layoutTable from "../../../components/tableLayout/index";
import { get_nx_ems } from "../../../api/report/nx_ems";
import moment from "moment";
import "moment/locale/zh-cn";
import Zcurd from "./zcrud";
import { hasNoDataDays } from "./util";
import {excel} from "@/api/common";
import {mapGetters} from 'vuex'
let area = "";
let plant = "";
export default {
  props: {},
  data() {
    return {
      tableLoading: false,
      tableHeight: 400,
      year: moment(Date.now()).year(),
      month: moment(Date.now()).month() + 1,
      projectCode: "",
      time: Date.now(),
      pickerOptions: {
        disabledDate(time) {
          return time.getTime() > Date.now();
        }
      },
      tableData: [],
      query: {
        year: moment(Date.now()).year(),
        month: moment(Date.now()).month() + 1
      },
      area: "",
      plant: "",
      area_plant: "",
      tableShow: true,
      unitList: ["#1", "#2"]
    };
  },
  components: { layoutTable, Zcurd },
  computed: {
    isSucess() {
      return this.$store.state.nx_ems.isSucess;
    },
    ...mapGetters(["permissions"])
  },
  created() {
      this.desulphurize_add = this.permissions["txconsume_add"]; // 新增
  },
  mounted() {
    this.tableShow = false;
    this.$nextTick(() => {
      this.tableHeight = this.$refs.nx_ems.offsetHeight - 230;
      this.tableShow = true;
    });
  },
  watch: {
    time(value) {
      this.query.year = moment(value).year();
      this.query.month = moment(value).month() + 1;
    },
    isSucess(value) {
      if (value == true) {
        this.checkList();
        this.$store.commit("SET_ISSUCESS", false);
      }
    }
  },
  methods: {
    checkList() {
      if (!this.query.projectCode) {
        return this.$message.error("请选择电厂");
      }
      this.year = this.query.year;
      this.month = this.query.month;
      this.projectCode = this.query.projectCode;
      this.area_plant = area + "/" + plant;
      this.get_nx_ems(this.query);
    },
    onExport() {
      if (!this.query.projectCode) {
        return this.$message.error("请先选择电厂");
      }
      // this.downloadFile("/datamonitor/hd_txconsume/export", {
      //   ...this.query,
      //   plantName: plant
      // });
      excel("/datamonitor/hd_txconsume/export",
        {...this.query,plantName:plant}
      ).then(res=>{
        let data = res.data;
        let excelName = `${this.year}年 ${this.month}月 ${this.area_plant} 脱硝装置耗用统计表.xls`;
        this.excel(data,excelName);
      })
    },
    selectArear(data) {
      area = data.name;
      delete this.query.projectCode;
    },
    selectPlant(data) {
      if (data == "all") {
        this.query = _.omit(this.query, "plant");
      } else {
        plant = data.name;
        this.query.projectCode = data.deptCode;
      }
    },
    get_nx_ems(query) {
      this.tableLoading = true;
      get_nx_ems(query).then(res => {
        this.tableLoading = false;
        let datas = res.data.data;
        let { units, data } = datas;
        this.unitList = units;
        this.tableData = data;
      });
    },
    // 点击新增按钮
    onAdd() {
      if (!this.query.projectCode) {
        return this.$message.error("请选择电厂");
      } else if (
        !this.tableData.length ||
        this.query.year + this.query.month != this.year + this.month
      ) {
        return this.$message.error("请先查询");
      }
      let curd = this.$refs.curd;
      curd.dialogFormVisible = true;
      let dayArray = hasNoDataDays(
        this.tableData,
        "day",
        this.query.year + "-" + this.query.month
      );
      curd.options = dayArray.map(item => {
        return {
          label: this.query.year + "-" + this.query.month + "-" + item,
          value: item
        };
      });
      curd.date = dayArray[dayArray.length - 1];
      curd.title = "新增记录";
      this.$store.commit("SET_QUERY", this.query);
      let datas =JSON.parse(JSON.stringify(this.tableData[0]));
      for (var key in datas) {
        datas[key] = null;
      }
      this.$store.commit("SET_ROW", datas);
    }
  }
};
</script>

<style scoped lang="scss">
.de_gas {
  height: 100%;
  .div__tool-wrap {
    padding: 20px 8px;
    .select__wrap {
      height: 42px;
      line-height: 42px;
      display: table;
      width: 100%;
      & > div:nth-child(1) {
        display: table-cell;
        width: 0.1%;
        min-width: 90px;
        // height: 72px;
        font-size: 14px;
        font-family: PingFangSC-Regular, PingFang SC;
        font-weight: 400;
        color: rgba(0, 0, 0, 0.84);
        line-height: 12px;
        text-align: right;
      }
      & > div:nth-child(2) {
        display: table-cell;
      }
    }
  }
  .div__content-wrap {
    padding: 0 32px;

    .title {
      font-size: 18px;
      font-family: PingFang-SC-Heavy, PingFang-SC;
      font-weight: 800;
      color: rgba(51, 51, 51, 1);
    }
    .content-table {
      padding-bottom: 20px;
    }
  }
}
</style>
<style lang="scss">
.de_gas {
  .el-card__body {
    padding: 0;
  }
  .avue-crud__menu {
    height: 0;
    min-height: 0;
    margin-bottom: 0;
  }
  .el-table thead.is-group th {
    background: #fff;
  }
  .el-table__fixed-footer-wrapper tbody td {
    background-color: #fff;
  }
  .el-table__footer-wrapper tbody td,
  .el-table__header-wrapper tbody td {
    background-color: #fff;
  }
}
</style>
