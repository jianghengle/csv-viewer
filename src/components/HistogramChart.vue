<template>
  <div class="chart-block">
    <div class="chart-header">
      <span class="chart-title">{{chart.type}} Chart {{chart.id}}</span>
      <div class="chart-options">
        <input class="input is-small height-input" type="text" placeholder="Height" v-model="chartHeight" @change="chartHeightChanged">
        <input class="input is-small height-input" type="text" placeholder="Y Max" v-model="yMax" @change="yMaxChanged">
      </div>
      <a class="delete delete-button" @click="deleteChart"></a>
    </div>
    <div :id="'chart'+chart.id"></div>
    <div class="chart-options">
      <div class="variable field" v-for="(v, i) in variables">
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
        &nbsp;&nbsp;
      </div>
      <a class="button is-small" @click="addVariable"><icon name="plus" scale="0.6"></icon>&nbsp;Variable</a>&nbsp;&nbsp;
      <input class="input is-small bins-input" type="text" placeholder="Bins: [,]p1,[(n),]p2...[,]" v-model="binsInput" @change="binsChanged">
    </div>
  </div>
</template>

<script>
import c3 from 'c3'

export default {
  name: 'histogram-chart',
  props: ['chart', 'headers', 'rows', 'showTable', 'showCharts'],
  data () {
    return {
      variables: [],
      dataMap: {},
      bins: [],
      binsInput: '',
      chartHeight: 320,
      yMax: 11,
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
  },
  methods: {
    updateColumn (variable) {
      var data = this.computeHistogram(variable)
      data = ['data' + variable.index].concat(data)

      this.self.chart.load({ columns: [ data ]})
      var names = {}
      names['data' + variable.index] = variable.name
      this.self.chart.data.names(names)
      this.self.chart.axis.range({
        max: {y: 100},
        min: {y: 0}
      })
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
        lineType: 'line'
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
    resizeChart (height) {
      if(height){
        this.self.chart.resize({height: height})
      }else{
        this.self.chart.resize()
      }
    },
    lineTypeChanged (variable) {
      this.$nextTick(function(){
        var v = this.variables[variable.index]
        this.self.chart.transform(v.lineType, 'data' + v.index)
      })
    },
    chartHeightChanged () {
      this.$nextTick(function(){
        this.resizeChart(this.chartHeight)
      })
    },
    yMaxChanged () {
      this.$nextTick(function(){
        this.self.chart.axis.range({
          max: {y: this.yMax},
        })
      })
    },
    binsChanged () {
      var ss = this.binsInput.split(',')
      if(ss.length < 2) return
      var minRange = (ss[0].trim() == '')
      var maxRange = (ss[ss.length-1].trim() == '')
      if(minRange){
        ss.splice(0, 1)
      }
      if(maxRange){
        ss.splice(ss.length-1, 1)
      }

      var pts = []
      for(var i=0;i<ss.length;i++){
        var s = ss[i].trim()
        if(s == '') return
        if(s[0] == '(' && s[s.length-1] == ')'){
          var n = Number(s.slice(1, -1))
          if(n === NaN) return
          if(i == 0 || i == ss.length - 1) return
          if(Math.round(n) != n) return
          if(n <= 0) return
          if(n == 1) continue
          var min = pts[pts.length - 1]
          if(!ss[i+1].length || ss[i+1][0] == '(') return
          var max = Number(ss[i+1])
          if(max === NaN) return
          if(max <= min) return
          var interval = (max - min) / n
          for(var j=1;j<n;j++){
            pts.push(min + interval*j)
          }
        }else{
          var n = Number(s)
          if(n === NaN) return
          if(n <= pts[pts.length-1]) return
          pts.push(n)
        }
      }

      if(!pts.length) return
      var bins = []
      if(minRange){
        bins.push([null, pts[0]])
      }
      for(var i=0;i<pts.length-1;i++){
        bins.push([pts[i], pts[i+1]])
      }
      if(maxRange){
        bins.push([pts[pts.length-1], null])
      }

      this.bins = bins
      this.updateAllColumns()
    },
    updateAllColumns () {
      for(var i=0;i<this.variables.length;i++){
        this.updateColumn(this.variables[i])
      }
    },
    computeHistogram (variable) {
      var data = this.dataMap[variable.dataIndex]
      var counts = this.bins.map(function(b){
        return 0
      })
      for(var i=0;i<data.length;i++){
        var d = data[i]
        for(var j=0;j<this.bins.length;j++){
          var b = this.bins[j]
          if(b[0] === null){
            if(d < b[1]){
              counts[j] = counts[j] + 1
              break
            }
          }else if(b[1] === null){
            if(d >= b[0]){
              counts[j] = counts[j] + 1
              break
            }
          }else{
            if(d >= b[0] && d < b[1]){
              counts[j] = counts[j] + 1
              break
            }else if(j == this.bins.length-1 && d == b[1]){
              counts[j] = counts[j] + 1
              break
            }
          }
        }
      }

      var len = data.length

      return counts.map(function(c){
        return Math.round(c / len * 10000) / 100
      })
    },
    formatXAxis (d) {
      var b = this.bins[d]
      if(b[0] === null) return '<' + b[1]
      if(b[1] === null) return '>=' + b[0]
      return b[0] + '~' + b[1]
    }
  },
  mounted () {
    this.$nextTick(function(){
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
        data: { columns: [] }
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

.bins-input {
  width: 300px;
  display: inline-block;
}

</style>
