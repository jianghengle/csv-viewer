<template>
  <div class="chart-block">
    <div class="chart-header">
      <span class="chart-title">{{chart.type}} Chart {{chart.id}}</span>
      <div class="chart-options">
        <select class="select" v-model="chartType" @change="chartTypeChanged">
          <option>line</option>
          <option>scatter</option>
          <option>pie</option>
          <option>donut</option>
        </select>
        <input class="input is-small height-input" type="text" placeholder="Height" v-model="chartHeight" @change="chartHeightChanged">
        <select class="select" v-model="groupOptionIndex" v-if="xys.length == 1">
          <option v-for="(opt, i) in groupOptions" v-bind:value="i">
            {{opt.name}}
          </option>
        </select>
      </div>
      <a class="delete delete-button" @click="deleteChart"></a>
    </div>
    <div :id="'chart'+chart.id"></div>
    <div class="chart-options">
      <div class="variable field" v-for="(v, i) in xys">
        <input type="checkbox" v-model="v.show" @click="toggleShow(v)">
        <select class="select" v-model="v.xIndex" @change="xyIndexChanged(v)">
          <option v-for="(h, j) in headers" v-bind:value="j">
            {{h}}
          </option>
        </select>
        <select class="select" v-model="v.yIndex" @change="xyIndexChanged(v)">
          <option v-for="(h, j) in headers" v-bind:value="j">
            {{h}}
          </option>
        </select>
        &nbsp;&nbsp;
      </div>
      <a class="button is-small" @click="addXY"><icon name="plus" scale="0.6"></icon>&nbsp;XY</a>
    </div>
  </div>
</template>

<script>
import c3 from 'c3'

export default {
  name: 'simple-xy',
  props: ['chart', 'headers', 'rows', 'showTable', 'showCharts', 'groupOptions', 'groupColors'],
  data () {
    return {
      xys: [],
      dataMap: {},
      chartType: 'line',
      chartHeight: 320,
      self: {},
      groupOptionIndex: 0,
      groupRows: {}
    }
  },
  watch: {
    showTable: function (val) {
      this.resizeChart()
    },
    showCharts: function (val) {
      this.resizeChart()
    },
    groupOptionIndex: function (val) {
      this.self.chart.unload()
      var vm = this
      setTimeout(function(){
        if(val){
          var groupOption = vm.groupOptions[vm.groupOptionIndex]
          var groupValues = Object.keys(groupOption.values)
          var groupRows = {}
          for(var i=0;i<groupValues.length;i++){
            var groupValue = groupValues[i]
            groupRows[groupValue] = []
          }
          for(var i=0;i<vm.rows.length;i++){
            var row = vm.rows[i]
            var groupValue = row[groupOption.dataIndex]
            groupRows[groupValue].push(row)
          }
          vm.groupRows = groupRows
        }
        for(var i=0;i<vm.xys.length;i++){
          var xy = vm.xys[i]
          vm.updateColumn(xy)
        }
      }, 500)
    }
  },
  methods: {
    updateColumn (xy) {
      if(this.groupOptionIndex){
        this.updateColumnWithGroup(xy)
      }else{
        var xData = this.dataMap[xy.xIndex]
        xData = ['data' + xy.index + '_x'].concat(xData)
        var yData = this.dataMap[xy.yIndex]
        yData = ['data' + xy.index].concat(yData)
        var xs = {}
        xs['data' + xy.index] = 'data' + xy.index + '_x'
        this.self.chart.load({ xs: xs, columns: [ xData, yData ] })
        var names = {}
        names['data' + xy.index] = xy.name
        this.self.chart.data.names(names)
      }
    },
    updateColumnWithGroup (xy) {
      var groupOption = this.groupOptions[this.groupOptionIndex]
      var groupValues = Object.keys(groupOption.values)
      for(var i=0;i<groupValues.length;i++){
        var groupValue = groupValues[i]
        var rows = this.groupRows[groupValue]
        var xData = ['data' + xy.index + '_' + i + '_x']
        var yData = ['data' + xy.index + '_' + i]
        for(var j=0;j<rows.length;j++){
          var row = rows[j]
          xData.push(Number(row[xy.xIndex]))
          yData.push(Number(row[xy.yIndex]))
        }
        var xs = {}
        xs['data' + xy.index + '_' + i] = 'data' + xy.index + '_' + i + '_x'
        var colors = {}
        colors['data' + xy.index + '_' + i] = this.groupColors[i]
        this.self.chart.load({ xs: xs, columns: [ xData, yData ], colors: colors })
        var names = {}
        names['data' + xy.index + '_' + i] = xy.name + '(' + groupValue + ')'
        this.self.chart.data.names(names)
      }
    },
    deleteChart () {
      this.$emit('delete-chart', this.chart)
    },
    xyIndexChanged (xy) {
      var xName = this.headers[xy.xIndex]
      var yName = this.headers[xy.yIndex]
      xy.name = xName + '-'+ yName
      this.loadData(xy)
      this.updateColumn(xy)
    },
    addXY () {
      var xy = {
        index: this.xys.length,
        xIndex: 0,
        yIndex: 0,
        name: 'Row-Row',
        show: true
      }
      if(this.xys.length){
        var pre = this.xys[this.xys.length-1]
        xy.xIndex = pre.xIndex
        xy.yIndex = pre.yIndex
        xy.name = pre.name
      }
      this.xys.push(xy)
      this.loadData(xy)
      if(this.xys.length > 1 && this.groupOptionIndex){
        this.groupOptionIndex = 0
      }else{
        this.updateColumn(xy)
      }
    },
    loadData (xy) {
      if(!this.dataMap[xy.xIndex]){
        var xData = []
        for(var i=0;i<this.rows.length;i++){
          xData.push(Number(this.rows[i][xy.xIndex]))
        }
        this.dataMap[xy.xIndex] = xData
      }
      if(!this.dataMap[xy.yIndex]){
        var yData = []
        for(var i=0;i<this.rows.length;i++){
          yData.push(Number(this.rows[i][xy.yIndex]))
        }
        this.dataMap[xy.yIndex] = yData
      }
    },
    resizeChart (height) {
      if(height){
        this.self.chart.resize({height: height})
      }else{
        this.self.chart.resize()
      }
    },
    toggleShow (xy) {
      this.$nextTick(function(){
        xy = this.xys[xy.index]
        if(xy.show){
          this.self.chart.show(['data' + xy.index])
        }else{
          this.self.chart.hide(['data' + xy.index])
        }
      })
    },
    chartTypeChanged () {
      this.$nextTick(function(){
        this.self.chart.transform(this.chartType)
      })
    },
    chartHeightChanged () {
      this.$nextTick(function(){
        this.resizeChart(this.chartHeight)
      })
    }
  },
  mounted () {
    this.$nextTick(function(){
      this.self.chart = c3.generate({
        bindto: '#chart' + this.chart.id,
        data: { columns: [] },
        type: this.chartType
      })
    })
  }
}
</script>


<style lang="scss" scoped>

.chart-block {
  margin-top: 10px;
}

.chart-header {
  text-align: center;
}

.chart-title {
  font-size: 16px;
  font-weight: bold;
}

.chart-options {
  display: inline-block;
}

.height-input {
  width: 50px;
  display: inline-block;
}

.delete-button {
  position: absolute;
  right: 20px;
}

.variable {
  display: inline-block;
}

.group-input {
  width: 50px;
  display: inline-block;
}



</style>
