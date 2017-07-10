<template>
  <div class="chart-block">
    <div class="chart-header">
      <span class="chart-title">{{chart.type}} Chart {{chart.id}}</span>
      <div class="chart-options">
        <select class="select" v-model="chartType" @change="chartTypeChanged">
          <option>line</option>
          <option>pie</option>
          <option>donut</option>
        </select>
        <input class="input is-small height-input" type="text" placeholder="Height" v-model="chartHeight" @change="chartHeightChanged">
      </div>
      <a class="delete delete-button" @click="deleteChart"></a>
    </div>
    <div :id="'chart'+chart.id"></div>
    <div class="chart-options">
      <div class="variable field" v-for="(v, i) in variables">
        <input type="checkbox" v-model="v.show" @click="toggleShow(v)">
        <select class="select" v-model="v.dataIndex" @change="dataIndexChanged(v)">
          <option v-for="(h, j) in headers" v-bind:value="j">
            {{h}}
          </option>
        </select>
        <select class="select" v-model="v.lineType" @change="lineTypeChanged(v)">
          <option>line</option>
          <option>area</option>
          <option>spline</option>
          <option>area-spline</option>
          <option>step</option>
          <option>area-step</option>
          <option>bar</option>
        </select>
        <input class="input is-small group-input" type="text" placeholder="Group" v-model="v.group" @change="groupChanged(v)">
        &nbsp;&nbsp;
      </div>
      <a class="button is-small" @click="addVariable"><icon name="plus" scale="0.6"></icon>&nbsp;Variable</a>
    </div>
  </div>
</template>

<script>
import c3 from 'c3'

export default {
  name: 'simple-chart',
  props: ['chart', 'headers', 'rows', 'showTable', 'showCharts'],
  data () {
    return {
      variables: [],
      dataMap: {},
      chartType: 'line',
      chartHeight: 320,
      self: {}
    }
  },
  watch: {
    showTable: function (val) {
      this.resizeChart()
    },
    showCharts: function (val) {
      this.resizeChart()
    },
    rows: function (val) {
      this.reloadAllColumns()
    }
  },
  methods: {
    updateColumn (variable) {
      var data = this.dataMap[variable.dataIndex]
      data = ['data' + variable.index].concat(data)
      this.self.chart.load({ columns: [ data ]})
      var names = {}
      names['data' + variable.index] = variable.name
      this.self.chart.data.names(names)
    },
    deleteChart () {
      this.$emit('delete-chart', this.chart)
    },
    dataIndexChanged (variable) {
      var name = this.headers[variable.dataIndex]
      variable.name = name
      this.loadData(variable)
      this.updateColumn(variable)
    },
    addVariable () {
      var variable = {
        index: this.variables.length,
        dataIndex: 0,
        name: 'Row',
        show: true,
        lineType: 'line',
        group: ''
      }
      this.variables.push(variable)
      this.loadData(variable)
      this.updateColumn(variable)
    },
    loadData (variable) {
      if(!this.dataMap[variable.dataIndex]){
        var data = []
        for(var i=0;i<this.rows.length;i++){
          data.push(Number(this.rows[i][variable.dataIndex]))
        }
        this.dataMap[variable.dataIndex] = data
      }
    },
    reloadData () {
      var indexes = Object.keys(this.dataMap)
      for(var i=0;i<indexes.length;i++){
        var index = indexes[i]
        var data = []
        for(var j=0;j<this.rows.length;j++){
          data.push(Number(this.rows[j][index]))
        }
        this.dataMap[index] = data 
      }
    },
    resizeChart (height) {
      if(height){
        this.self.chart.resize({height: height})
      }else{
        this.self.chart.resize()
      }
    },
    reloadAllColumns () {
      this.reloadData()
      for(var i=0;i<this.variables.length;i++){
        var v = this.variables[i]
        this.updateColumn(v)
      }
    },
    toggleShow (variable) {
      this.$nextTick(function(){
        var v = this.variables[variable.index]
        if(v.show){
          this.self.chart.show(['data' + v.index])
        }else{
          this.self.chart.hide(['data' + v.index])
        }
      })
    },
    lineTypeChanged (variable) {
      this.$nextTick(function(){
        var v = this.variables[variable.index]
        this.self.chart.transform(v.lineType, 'data' + v.index)
      })
    },
    groupChanged (variable) {
      this.$nextTick(function(){
        var groups = {}
        for(var i=0;i<this.variables.length;i++){
          var v = this.variables[i]
          var g = v.group.trim()
          if(!g) continue
          if(groups[g]){
            groups[g].push('data' + v.index)
          }else{
            groups[g] = ['data' + v.index]
          }
        }
        this.self.chart.groups(Object.values(groups))
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
