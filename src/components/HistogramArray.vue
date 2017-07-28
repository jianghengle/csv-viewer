<template>
  <div class="chart-block">
    <div class="chart-header">
      <span class="chart-title">{{chart.type}} Chart {{chart.id}}</span>
      <div class="chart-options">
        <input class="input is-small height-input" type="text" placeholder="Size" v-model="chartSize" @change="chartSizeChanged">
        <input class="input is-small height-input" type="text" placeholder="Height" v-model="chartHeight" @change="chartHeightChanged">
        <input class="input is-small height-input" type="text" placeholder="Height" v-model="bins" @change="binsChanged">
        &nbsp;
        <label class="checkbox">
          <input type="checkbox" v-model="sortByEntropy">
          Sort by Entropy
        </label>
        &nbsp;
        <select class="select" v-model="groupOptionIndex">
          <option v-for="(opt, i) in groupOptions" v-bind:value="i">
            {{opt.name}}
          </option>
        </select>
      </div>
      <a class="delete delete-button" @click="deleteChart"></a>
    </div>
    <div>
      <div v-for="(v, i) in variables" :key="'chart'+chart.id+'_'+i" class="small-histogram" v-if="v.data" 
        :class="{'active-histogram': i==activeIndex}"
        @click="activeIndex = i">
        <div :id="'chart'+chart.id+'_'+i"></div>
        <div class="small-histogram-name" :style="{'width': chartSize + 'px'}">{{v.name}}</div>
      </div>
    </div>
    <div :id="'chart'+chart.id"></div>
  </div>
</template>

<script>
import c3 from 'c3'

export default {
  name: 'histogram-array',
  props: ['chart', 'headers', 'rows', 'showTable', 'showCharts', 'groupOptions', 'groupColors'],
  data () {
    return {
      bins: 20,
      self: {},
      activeIndex: 0,
      chartHeight: 320,
      chartSize: 50,
      variables: [],
      sortByEntropy: false,
      groupOptionIndex: 0
    }
  },
  computed: {
    len () {
      return this.rows.length
    },
  },
  watch: {
    showTable: function (val) {
      this.resizeChart()
    },
    showCharts: function (val) {
      this.resizeChart()
    },
    activeIndex: function (val) {
      var variable = this.variables[val]
      this.self.chart.load({ columns: [ variable.data ]})
      this.self.chart.data.names({data: variable.name})

      if(this.groupOptionIndex){
        var colors = {}
        var names = {}
        var types = {}
        var groupOption = this.groupOptions[this.groupOptionIndex]
        var groupValues = Object.keys(groupOption.values)
        for(var i=0;i<groupValues.length;i++){
          colors['data' + i] = this.groupColors[i]
          names['data' + i] = groupValues[i]
          types['data' + i] = 'area-spline'
        }
        this.self.chart.load({ columns: variable.groupData, colors: colors, types: types })
        this.self.chart.data.names(names)
      }
    },
    sortByEntropy: function (val) {
      if(val){
        this.variables.sort(function(a, b){
          return b.entropy - a.entropy
        })
      }else{
        this.variables.sort(function(a, b){
          return a.dataIndex - b.dataIndex
        })
      }
      this.reloadCharts()
    },
    groupOptionIndex: function (val) {
      this.computeHistograms()
      this.$nextTick(function(){
        this.reloadCharts()
      })
    }
  },
  methods: {
    deleteChart () {
      this.$emit('delete-chart', this.chart)
    },
    resizeChart (height) {
      if(height){
        this.self.chart.resize({height: height})
      }else{
        this.self.chart.resize()
      }
    },
    formatXAxis (d) {
      var variable = this.variables[this.activeIndex]
      var bins = variable.bins
      var b = bins[d]
      var start = Math.round(b[0] * 100) / 100
      var end = Math.round(b[1] * 100) / 100
      return start + '~' + end
    },
    chartHeightChanged () {
      this.$nextTick(function(){
        this.resizeChart(this.chartHeight)
      })
    },
    chartSizeChanged () {
      this.$nextTick(function(){
        for(var i=0;i<this.self.charts.length;i++){
          var chart = this.self.charts[i]
          chart.resize({width: this.chartSize, height: this.chartSize})
        }
      })
    },
    binsChanged () {
      this.computeHistograms()
      this.$nextTick(function(){
        this.reloadCharts()
      })
    },
    reloadCharts () {
      var colors = {}
      var names = {}
      var types = {}
      if(this.groupOptionIndex){
        var groupOption = this.groupOptions[this.groupOptionIndex]
        var groupValues = Object.keys(groupOption.values)
        for(var i=0;i<groupValues.length;i++){
          colors['data' + i] = this.groupColors[i]
          names['data' + i] = groupValues[i]
          types['data' + i] = 'area-spline'
        }
      }

      for(var i=0;i<this.variables.length;i++){
        let chart = this.self.charts[i]
        let variable = this.variables[i]
        if(!chart) continue
        chart.unload()
        var vm = this
        setTimeout(function(){
          chart.load({ columns: [ variable.data ]})
          if(vm.groupOptionIndex){
            chart.load({ columns: variable.groupData, colors: colors, types: types })
          }
        }, 500)
      }

      this.self.chart.unload()
      var vm = this
      setTimeout(function(){
        let variable = vm.variables[vm.activeIndex]
        vm.self.chart.load({ columns: [ variable.data ]})
        vm.self.chart.data.names({data: variable.name})
        if(vm.groupOptionIndex){
          vm.self.chart.load({ columns: variable.groupData, colors: colors, types: types })
          vm.self.chart.data.names(names)
        }
      }, 500)
    },
    loadVariables () {
      var variables = []
      for(var i=1;i<this.headers.length;i++){
        var variable = {name: this.headers[i], dataIndex: i, min: null, max: null, bins: [], data: null}
        variables.push(variable)
      }

      for(var i=0;i<variables.length;i++){
        var variable = variables[i]
        for(var j=0;j<this.len;j++){
          var row = this.rows[j]
          var value = Number(row[variable.dataIndex])
          if(isNaN(value)) continue
          variable.min = variable.min === null ? value : Math.min(value, variable.min)
          variable.max = variable.max === null ? value : Math.max(value, variable.max)
        }
      }
      for(var i=variables.length-1;i>=0;i--){
        if(variables[i].max == variables[i].min){
          variables.splice(i, 1)
        }
      }
      this.variables = variables
    },
    computeHistograms () {
      var groupOption = null
      var groupValues = null
      if(this.groupOptionIndex){
        groupOption = this.groupOptions[this.groupOptionIndex]
        groupValues = Object.keys(groupOption.values)
      }

      for(var i=0;i<this.variables.length;i++){
        var variable = this.variables[i]
        if(variable.min == variable.max) continue
        variable.bins = []
        var interval = (variable.max - variable.min) / this.bins
        var counts = []
        for(var j=0;j<this.bins;j++){
          counts.push(0)
          var start = variable.min+j*interval
          var end = variable.min+(j+1)*interval
          if(j == this.bins-1){
            variable.bins.push([start, variable.max])
          }else{
            variable.bins.push([start, end])
          }
        }

        var groupCounts = {}
        if(groupOption){
          for(var j=0;j<groupValues.length;j++){
            var groupValue = groupValues[j]
            groupCounts[groupValue] = counts.slice()
          }
        }


        for(var j=0;j<this.len;j++){
          var row = this.rows[j]
          var value = Number(row[variable.dataIndex])
          if(value === NaN) continue
          for(var k=0;k<variable.bins.length;k++){
            var bin = variable.bins[k]
            if(k == variable.bins.length-1){
              if(value >= bin[0] && value <= bin[1]){
                counts[k] = counts[k] + 1
                if(groupOption){
                  var groupValue = row[groupOption.dataIndex]
                  groupCounts[groupValue][k] = groupCounts[groupValue][k] + 1
                }
              }
            }else{
              if(value >= bin[0] && value < bin[1]){
                counts[k] = counts[k] + 1
                if(groupOption){
                  var groupValue = row[groupOption.dataIndex]
                  groupCounts[groupValue][k] = groupCounts[groupValue][k] + 1
                }
                break
              }
            }
          }
        }

        var len = this.len
        variable.data = []
        variable.entropy = 0
        for(var j=0;j<counts.length;j++){
          var p = counts[j] / len
          variable.data.push(p * 100)
          if(p > 0){
            variable.entropy -= p *  Math.log2(p)
          }
        }
        variable.data.unshift('data')

        if(groupOption){
          variable.groupData = []
          for(var j=0;j<groupValues.length;j++){
            var groupValue = groupValues[j]
            var data = groupCounts[groupValue].map(function(c){
              return c / len * 100
            })
            data.unshift('data' + j)
            variable.groupData.push(data)
          }
        }
      }
      if(this.sortByEntropy){
        this.variables.sort(function(a, b){
          return b.entropy - a.entropy
        })
      }else{
        this.variables.sort(function(a, b){
          return a.dataIndex - b.dataIndex
        })
      }
    }
  },
  mounted () {
    this.loadVariables()
    this.computeHistograms()
    this.$nextTick(function(){
      this.self.charts = []
      for(var i=0;i<this.variables.length;i++){
        var variable = this.variables[i]
        var chart = null
        if(variable.data){
          chart = c3.generate({
            size: { height: this.chartSize, width: this.chartSize },
            bindto: '#chart' + this.chart.id + '_' + i,
            data: {
              columns:  [variable.data],
              types: {data: 'area-spline'}
            },
            axis: {
              x: {show:false},
              y: {min: 0, show:false}
            },
            point: { show: false },
            legend: { show: false },
            tooltip: { show: false }
          })
        }
        this.self.charts.push(chart)
      }

      var activeVariable = this.variables[this.activeIndex]
      this.self.chart = c3.generate({
        bindto: '#chart' + this.chart.id,
        axis: {
          x: {
            tick: { format: this.formatXAxis },
            padding: {right: 0.5},
          },
          y: {
            tick: {
              format: function(d){
                return Math.round(d*100) / 100
              }
            },
            label: {
              text: '%',
              position: 'outer-middle'
            }
          }
        },
        data: {
          columns: [ activeVariable.data ],
          types: {data: 'area-spline'}
        },
        point: { show: false }
      })

      this.self.chart.data.names({data: activeVariable.name})
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

.small-histogram {
  display: inline-block;
  padding: 5px;
  margin: 5px;
  cursor: pointer;
}

.small-histogram-name {
  margin-top: -10px;
  font-size: 14px;
  text-align: center;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.active-histogram {
  border-radius: 3px;
  background-color: #EEEEEE;
}

</style>
