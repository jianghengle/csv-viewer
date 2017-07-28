<template>
  <div class="chart-block">
    <div class="chart-header">
      <span class="chart-title">{{chart.type}} Chart {{chart.id}}</span>
      <div class="chart-options">
        <input class="input is-small height-input" type="text" placeholder="Size" v-model="cellSize">
        <input class="input is-small height-input" type="text" placeholder="Height" v-model="chartHeight" @change="chartHeightChanged">
        <input class="input is-small cell-input" type="text" placeholder="X" v-model="activeX">
        <input class="input is-small cell-input" type="text" placeholder="Y" v-model="activeY">
      </div>
      <a class="delete delete-button" @click="deleteChart"></a>
    </div>
    <div class="matrix-container">
      <div class="matrix">
        <div v-for="(r, i) in matrix" class="row" :style="{'height': cellSize + 'px'}">
          <div v-for="(c, j) in r" class="cell" :style="{'width': cellSize+'px', 'height': cellSize+'px', 'background-color': c.color}"
            :class="{'active-cell': i==activeX && j==activeY}"
            @click="setActive(i, j)">
          </div>
        </div>
      </div>
      <div class="color-band-container">
        <div class="color-band">
        </div>
        <div class="color-annotation">
          <div class="top">1.0</div>
          <div class="middle">0.0</div>
          <div class="bottom">-1.0</div>
        </div>
      </div>
    </div>
    <div :id="'chart'+chart.id"></div>
  </div>
</template>

<script>
import c3 from 'c3'

export default {
  name: 'correlation-matrix',
  props: ['chart', 'headers', 'rows', 'showTable', 'showCharts'],
  data () {
    return {
      bins: 20,
      self: {},
      activeIndex: 0,
      chartHeight: 320,
      cellSize: 8,
      variables: [],
      matrix: [],
      activeX: 0,
      activeY: 0
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
    activeX: function (val) {
      if(val >= 0 && val <= this.variables.length)
        this.updateChart()
    },
    activeY: function (val) {
      if(val >= 0 && val <= this.variables.length)
        this.updateChart()
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
    chartHeightChanged () {
      this.$nextTick(function(){
        this.resizeChart(this.chartHeight)
      })
    },
    loadVariables () {
      var variables = []
      for(var i=1;i<this.headers.length;i++){
        var isNum = true
        var sum = 0
        var sqSum = 0
        var values = []
        for(var j=0;j<this.rows.length;j++){
          var value = Number(this.rows[j][i])
          if(isNaN(value)){
            isNum = false
            break
          }else{
            sum += value
            sqSum += value * value
            values.push(value)
          }
        }
        if(isNum){
          var variable = {name: this.headers[i], values: values, sum: sum, sqSum: sqSum}
          variables.push(variable)
        }
      }
      this.variables = variables
    },
    computeMatrix () {
      this.matrix = []
      for(var i=0;i<this.variables.length;i++){
        var row = []
        for(var j=0;j<this.variables.length;j++){
          var x = this.variables[i]
          var y = this.variables[j]
          if(i == j){
            row.push({x: x, y: y, cor: 1.0, color: this.color(1.0)})
          }else if(i > j){
            var cor = this.matrix[j][i].cor
            row.push({x: x, y: y, cor: cor, color: this.color(cor)})
          }else{
            var xySum = 0
            var n = this.len
            for(var k=0;k<n;k++){
              xySum +=  x.values[k] * y.values[k]
            }
            var cor = (n*xySum - x.sum*y.sum) / ( Math.sqrt(n*x.sqSum - x.sum*x.sum) * Math.sqrt(n*y.sqSum - y.sum*y.sum) )
            row.push({x: x, y: y, cor: cor, color: this.color(cor)})
          }
        }
        this.matrix.push(row)
      }
    },
    color (v) {
      var band = d3.scale.linear()
                  .domain([-1.0, 0, 1.0])
                  .range(["#000080", "#fff", "#B22222"])
      return band(v)
    },
    setActive (i, j) {
      this.activeX = i
      this.activeY = j
    },
    updateChart () {
      var cell = this.matrix[this.activeX][this.activeY]
      var xData = ['data_x'].concat(cell.x.values)
      var data = ['data'].concat(cell.y.values)
      this.self.chart.load({ columns: [ xData, data ]})
      var names = {'data': cell.x.name + '--' + cell.y.name + ': ' + cell.cor}
      this.self.chart.data.names(names)
    }
  },
  mounted () {
    this.loadVariables()
    this.computeMatrix()
    this.$nextTick(function(){
      var cell = this.matrix[this.activeX][this.activeY]
      var xData = ['data_x'].concat(cell.x.values)
      var data = ['data'].concat(cell.y.values)
      this.self.chart = c3.generate({
        bindto: '#chart' + this.chart.id,
        data: { 
          xs: {data: 'data_x'},
          columns: [xData, data],
          type: 'scatter'
        },
        axis: {
          x: { tick: { fit: false } }
        }
      })
      var names = {'data': cell.x.name + '--' + cell.y.name + ': ' + cell.cor}
      this.self.chart.data.names(names)
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

.cell-input {
  width: 40px;
  display: inline-block;
}

.delete-button {
  position: absolute;
  right: 20px;
}

.variable {
  display: inline-block;
}

.matrix-container {
  overflow-x: auto;
  white-space: nowrap;
}

.matrix {
  display: inline-block;
}

.row {
  
  white-space: nowrap;
  margin-top: 2px;
  margin-bottom: 2px;
}

.cell {
  
  display: inline-block;
  margin: 1px;
  border-radius: 2px;
  cursor: pointer;
}

.color-band-container {
  display: inline-block;
  vertical-align: top;
  margin-top: 10px;
  margin-left: 10px;

  .color-band {
    width: 20px;
    height: 200px;
    background-color: red;
    border-radius: 3px;

    background: #B22222; /* For browsers that do not support gradients */
    background: -webkit-linear-gradient(#B22222, #fff, #000080); /* For Safari 5.1 to 6.0 */
    background: -o-linear-gradient(#B22222, #fff, #000080); /* For Opera 11.1 to 12.0 */
    background: -moz-linear-gradient(#B22222, #fff, #000080); /* For Firefox 3.6 to 15 */
    background: linear-gradient(#B22222, #fff, #000080); /* Standard syntax */
  }

  .color-annotation {
    position: relative;
    left: 25px;
    font-size: 14px;

    .top{
      position: relative;
      top: -210px;
    }

    .middle{
      position: relative;
      top: -130px;
    }

    .bottom{
      position: relative;
      top: -55px;
    }
  }
}

.active-cell {
  border: 1px solid black;
}

</style>
