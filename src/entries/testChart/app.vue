<template>
    <div id="app">
        <div class="header">
        </div>
        <div class="container">
            <div style="padding-top:36px">
                <h1>{{proj_name}}项目成长图
    
                    <el-popover title="手机扫描二维码访问当前分享页面" ref="QRpopover" placement="bottom" trigger="click">
                        <div id="urlQrcode" draggable="false" style="padding:10px;"></div>
                    </el-popover>
                    <el-button class="btn-share" type="info" size="large" icon="share" v-popover:QRpopover @click="shareQrcode">手机二维码访问</el-button>
    
                </h1>
                <div>
                    <h3 class="chartH">用例增长图</h3>
                    <div id="vx-echart1"></div>
                    <h3 class="chartH">Bug消减图</h3>
                    <div id="vx-echart2"></div>
                </div>
            </div>
        </div>
        <div class="footer">
            {{version}}
        </div>
    </div>
</template>
<script>
import filters from "../../filters/filters";
var BeijingTime = filters.BeijingTime;
import Qrcode from "../../assets/qrcode.js";
import { queryParser } from "../../assets/utils";
import { apiHost as serverUrl } from "../../config";
import pageConfig from "../../sysConfig/page";
var ui = {};

function AllotChartHeight() {
  if (!ui.e1) {
    ui.e1 = document.getElementById("vx-echart1");
  }
  if (!ui.e2) {
    ui.e2 = document.getElementById("vx-echart2");
  }
  var avaliHeight = innerHeight - 310,
    h1 = ~~(avaliHeight / 2 - 10),
    h2 = avaliHeight - h1;
  ui.e1.style.height = h1 + "px";
  ui.e2.style.height = h2 + "px";
}

function isOK(res) {
  return res.data && res.data.code == 200;
}

var echarts = require("echarts"),
  chartBinds;
function renderChart() {
  var colors = ["#f00", "#d14a61", "#ac5"],
    Data = chartBinds,
    xMarks = Data.map((v, i) => ++i),
    Total = Data.map(v => v.total),
    Skipped = Data.map(v => v.skipped),
    Errors = Data.map(v => v.errors),
    Filures = Data.map(v => v.failures),
    Max = [Skipped, Errors, Filures].map(arr => Math.max.apply(null, arr)),
    formatter = params => {
      var i = params[0].dataIndex,
        data = Data[i] || {};
      return `${i + 1}<br>
                            版本信息 : ${data.pro_version} <br>
                            创建时间 : ${BeijingTime(data.rc_time)} <br>
    ${params
      .map(
        x =>
          `<i style="display:inline-block;margin-right:5px;border-radius:10px;width:9px;height:9px;background-color:${
            x.color
          }"></i>${x.seriesName} : ${x.value}<br>`
      )
      .join("")}

                        `;
    };
  Max = Math.max.apply(null, Max);
  if (!ui.m1) {
    ui.m1 = echarts.init(ui.e1);
  }
  if (!ui.m2) {
    ui.m2 = echarts.init(ui.e2);
  }
  ui.m1.setOption({
    color: colors,
    // title: {
    //     text: '用例增长图'
    // },
    tooltip: {
      trigger: "axis",
      axisPointer: {
        type: "cross"
      },
      formatter
    },
    toolbox: {
      show: true,
      feature: { saveAsImage: {} }
    },
    legend: {
      data: ["总次数"]
    },
    grid: {
      top: 30,
      bottom: 20
    },
    xAxis: {
      type: "category",
      boundaryGap: !1,
      data: xMarks
    },
    yAxis: {
      type: "value"
    },
    series: [
      {
        name: "总次数",
        type: "line",
        smooth: true,
        data: Total
      }
    ]
  });

  ui.m2.setOption({
    color: colors,
    // title: {
    //     text: 'Bug消减图'
    // },
    tooltip: {
      trigger: "axis",
      axisPointer: {
        type: "cross"
      },
      formatter
    },
    toolbox: {
      show: true,
      feature: { saveAsImage: {} }
    },
    legend: {
      data: ["错误次数", "失败次数", "跳过次数"]
    },
    grid: {
      top: 40,
      bottom: 20
    },
    xAxis: {
      type: "category",
      boundaryGap: !1,
      data: xMarks
    },
    yAxis: {
      type: "value"
    },
    series: [
      ["错误次数", Errors],
      ["失败次数", Filures],
      ["跳过次数", Skipped]
    ].map(x => ({
      name: x[0],
      type: "line",
      smooth: true,
      data: x[1]
    }))
  });
}
var qs = queryParser(),
  resizeId;
window.onresize = function() {
  if (resizeId > 0) {
    clearTimeout(resizeId);
  }
  resizeId = setTimeout(function() {
    AllotChartHeight();
    if (ui.m1) {
      ui.m1.resize();
    }
    if (ui.m2) {
      ui.m2.resize();
    }
  }, 1e2);
};
export default {
  data() {
    return {
      stoken: "",
      proj_name: "",
      data: [],
      isloading: !0,
      version: pageConfig.version
    };
  },
  methods: {
    shareQrcode() {
      var el = document.getElementById("urlQrcode");
      !el.querySelector("img") &&
        new Qrcode(el, {
          text: location.href,
          width: 260,
          height: 260,
          correctLevel: Qrcode.CorrectLevel.H
        });
    },
    getRecord() {
      var vm = this;
      vm.isloading = !0;
      function onfail() {
        vm.isloading = !1;
        vm.$alert("获取项目成长信息发生错误", "提示", { type: "error" });
      }
      vm.$http
        .get(`${serverUrl}share/get-pro-share-data/`, {
          params: {
            stoken: vm.stoken,
            page_cap: 30
          }
        })
        .then(res => {
          vm.isloading = !1;
          if (isOK(res)) {
            vm.data = res.data.data.page_data.reverse();
            vm.proj_name = (vm.data[0] || {}).pro_name;
            AllotChartHeight();
            chartBinds = vm.data;
            renderChart();
          } else {
            onfail();
          }
        })
        .catch(onfail);
    }
  },
  created() {
    var vm = this,
      stoken = qs.stoken;
    if (!(vm.stoken = stoken)) {
      return vm.$alert("页面参数无效", { type: "warn" });
    }
    vm.getRecord();
  }
};
</script>
<style scoped lang=less rel="stylesheet/less">
#app {
  .header {
    height: 40px;
    line-height: 40px;
    background: linear-gradient(180deg, #545c61 0%, #363e43 100%);
    position: fixed;
    width: 100%;
    z-index: 99;
  }
  @media only screen and (max-width: 920px) {
    .btn-share {
      display: none;
    }
  }
  .btn-share {
    float: right;
    margin-right: 80px;
  }
  .chartH {
    width: 80%;
    margin: 0 auto;
  }

  .container {
    min-height: 89vh;
    #vx-echart1 {
      width: 80%;
      /* height: 320px; */
      margin: 30px auto;
    }
    #vx-echart2 {
      width: 80%;
      /* height: 360px; */
      margin: 30px auto;
    }
    h1 {
      display: block;
      font-size: 1.8rem;
      margin: 0.6rem auto;
      width: 95%;
      text-align: left;
    }
  }
  .footer {
    height: 40px;
    line-height: 40px;
    background: linear-gradient(180deg, #545c61 0%, #363e43 100%);
    color: white;
    text-align: center;
    font-size: 14px;
    vertical-align: middle;
    position: fixed;
    width: 100%;
    bottom: 0px;
    span {
      float: right;
      font-size: 12px;
      margin-right: 10px;
      color: rgba(86, 123, 146, 0.69);
    }
  }
}
</style>






