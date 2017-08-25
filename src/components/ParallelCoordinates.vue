<template>
  <div class="chart-block">
    <div class="chart-header">
      <span class="chart-title">{{chart.type}} Chart {{chart.id}}</span>
      <div class="chart-options">
        <input class="input is-small height-input" type="text" placeholder="Height" v-model="chartHeight">
        <label class="checkbox" v-if="rows.length">
          <input type="checkbox" v-model="showSamples">
          Show Samples
        </label>
      </div>
      <a class="delete delete-button" @click="deleteChart"></a>
    </div>
    <div :id="'chart'+chart.id" class="parcoords" style="width:100%;" :style="{'height': chartHeight+'px'}"></div>
    <div class="chart-options">
      <div class="variable field" v-for="(v, i) in variables">
        <input type="checkbox" v-model="v.show" @click="toggleShow()">
        <select class="select" v-model="v.dataIndex" @change="dataIndexChanged(v)">
          <option v-for="(h, j) in headers" v-bind:value="j">
            {{h}}
          </option>
        </select>
        <input type="radio" :value="i" v-model="colorBy">
        &nbsp;&nbsp;
      </div>
      <a class="button is-small" @click="addVariable"><icon name="plus" scale="0.6"></icon>&nbsp;Variable</a>
    </div>
    <div v-show="showSamples" class="sample-table">
      <table class="table is-narrow">
        <thead>
          <tr>
            <th v-for="v in variables">
              <div class="number-cell">
                {{headers[v.dataIndex]}}
              </div>
            </th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="row in table" @mouseover="mouseoverRow(row)" @mouseout="mouseoutRow(row)">
            <td v-for="cell in row" class="number-cell">
              {{cell}}
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script>
import * as d3 from "d3"
import ParCoords from 'parcoord-es'

export default {
  name: 'parallel-coordinates',
  props: ['chart', 'headers', 'rows', 'showTable', 'showCharts', 'groupOptions', 'groupColors'],
  data () {
    return {
      chartHeight: 320,
      variables: [],
      dimensions: {},
      data: [],
      colorBy: 0,
      table: [],
      parcoords: null,
      showSamples: false
    }
  },
  watch: {
    showTable: function (val) {
      this.resizeChart()
    },
    showCharts: function (val) {
      this.resizeChart()
    },
    colorBy: function(val) {
      this.$nextTick(function(){
        this.rerender()
      })
    },
    chartHeight: function(val) {
      this.$nextTick(function(){
        this.rerender()
      })
    }
  },
  methods: {
    deleteChart () {
      this.$emit('delete-chart', this.chart)
    },
    addVariable () {
      var variable = {
        index: this.variables.length,
        dataIndex: 0,
        show: true,
        type: 'number'
      }
      this.variables.push(variable)
      this.updateData(variable)
      this.rerender()
    },
    updateData (v) {
      delete v.max
      delete v.min
      for(var i=0;i<this.data.length;i++){
        var row = this.data[i]
        var value = this.rows[i][v.dataIndex]
        if(isNaN(value)){
          v.type = 'string'
        }else{
          v.type = 'number'
          value = Number(value)
          v.max = v.max == undefined ? value : Math.max(v.max, value)
          v.min = v.min == undefined ? value : Math.min(v.min, value)
        }
        if(v.index < row.length){
          row[v.index] = value
        }else{
          row.push(value)
        }
      }
      this.sample(this.data)

      this.dimensions[v.index] = {
        type: v.type,
        title: this.headers[v.dataIndex]
      }
    },
    rerender () {
      document.getElementById("chart"+this.chart.id).innerHTML = ""
      var hideAxis = []
      for(var i=0;i<this.variables.length;i++){
        var variable = this.variables[i]
        if(!variable.show){
          hideAxis.push(variable.index)
        }
      }

      this.parcoords = ParCoords()("#chart"+this.chart.id)
        .color(this.color)
        .alpha(0.4)
        .data(this.data)
        .dimensions(this.dimensions)
        .hideAxis(hideAxis)
        .composite("darker")
        .margin({ top: 30, left: 100, bottom: 30, right: 100 })
        .render()
        .shadows()
        .brushMode("1D-axes")

      this.parcoords.svg.selectAll(".dimension")
        .selectAll(".label")
        .style("font-size", "14px")

      this.parcoords.on("brush", this.onBrush)
    },
    color (d) {
      var v = this.variables[this.colorBy]
      var groupOption = null
      for(var i=0;i<this.groupOptions.length;i++){
        if(this.groupOptions[i].dataIndex == v.dataIndex){
          groupOption = this.groupOptions[i]
          break
        }
      }

      var val = d[this.colorBy]
      var c
      if(groupOption){
        var groupValues = Object.keys(groupOption.values)
        var index = groupValues.indexOf(val.toString())
        c = this.groupColors[index]
      }else{
        var band = d3.scale.linear()
                  .domain([v.min, v.max])
                  .range(["steelblue", "brown"])
                  .interpolate(d3.interpolateLab)
        c = band(val)
      }
      return c
    },
    dataIndexChanged (v) {
      this.updateData(v)
      this.rerender()
    },
    resizeChart (height) {
      this.rerender()
    },
    toggleShow () {
      this.$nextTick(function(){
        this.rerender()
      })
    },
    mouseoverRow (row) {
      this.parcoords.highlight([row])
    },
    mouseoutRow (row) {
      this.parcoords.unhighlight()
    },
    onBrush (rows) {
      this.sample(rows)
    },
    sample (rows) {
      this.table = []
      if(rows.length <= 10) {
        this.table = rows.slice()
        return
      }
      var interval = Math.floor(rows.length / 10)
      var i = 0
      while(i < rows.length){
        this.table.push(rows[i])
        i += interval
      }
    }
  },
  mounted () {
    var v = {index: 0, dataIndex: 0, type: 'number', show: true}
    this.variables.push(v)
    for(var i=0;i<this.rows.length;i++){
      var value = Number(this.rows[i][0])
      this.data.push([value])
      v.max = v.max == undefined ? value : Math.max(v.max, value)
      v.min = v.min == undefined ? value : Math.min(v.min, value)
    }
    this.sample(this.data)
    this.dimensions[0] = {
      type: 'number',
      title: 'Row'
    }
    this.rerender()
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

.sample-table {
  max-width: 100%;
  overflow-x: auto;
}

.number-cell {
  text-align: right;
  font-size: 14px;
}


</style>
