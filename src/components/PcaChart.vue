<template>
  <div class="chart-block">
    <div class="chart-header">
      <span class="chart-title">{{chart.type}} Chart {{chart.id}}</span>
      <div class="chart-options">
        <input class="input is-small height-input" type="text" placeholder="Height" v-model="chartHeight" @change="chartHeightChanged">
        <label class="checkbox">
          <input type="checkbox" v-model="showPoints">
          Points
        </label>
        <label class="checkbox">
          <input type="checkbox" v-model="showErrors">
          Errors
        </label>
      </div>
      <a class="delete delete-button" @click="deleteChart"></a>
    </div>
    <div :id="'chart'+chart.id" @mouseup="chartMouseUp"></div>
    <div class="pca-info"> 
      <div v-if="variances">Variances: {{variances}}</div>
      <div v-if="xyVectors">X Vector: {{xyVectors[0]}}</div>
      <div v-if="xyVectors">Y Vector: {{xyVectors[1]}}</div>
    </div>
    <div>
      <div class="variable field" v-for="(v, i) in variables">
        <select class="select" v-model="v.dataIndex" @change="dataIndexChanged(v)">
          <option v-for="(h, j) in headers" v-bind:value="j">
            {{h}}
          </option>
        </select>
        <a class="delete delete-variable-button" @click="deleteVariable(v)"></a>
        &nbsp;&nbsp;
      </div>
      <a class="button is-small" @click="addVariable"><icon name="plus" scale="0.6"></icon>&nbsp;Variable</a>
      &nbsp;&nbsp;
      <a class="button is-small" :class="{'is-loading': computing}" v-show="showCompute" @click="computePCA">Compute</a>
    </div>
    <div class="pca-info" v-if="groups.length">
      <span v-for="(g, i) in groups">
        <label class="radio group-label">
          <input type="radio" :value="i" v-model="activeGroup">
          Group{{i}}
        </label>
        <a v-if="i > 0" class="delete delete-variable-button" @click="deleteGroup(i)"></a>
        &nbsp;&nbsp;
      </span>
      <a class="button is-small" @click="addGroup"><icon name="plus" scale="0.6"></icon>&nbsp;Group</a>
      <a class="button is-small" @click="makeExportUrl">Export</a>
      <a v-if="exportUrl" :href="exportUrl" target="_blank" download="groups.csv">Export Url</a>
    </div>
  </div>
</template>

<script>
import c3 from 'c3'
import PCA from 'ml-pca'

export default {
  name: 'pca-chart',
  props: ['chart', 'headers', 'rows', 'showTable', 'showCharts'],
  data () {
    return {
      variables: [],
      chartHeight: 320,
      self: {},
      showCompute: false,
      computing: false,
      pca: null,
      groups: [],
      activeGroup: 0,
      variances: null,
      xyVectors: null,
      showPoints: true,
      showErrors: false,
      xRange: null,
      yRange: null,
      exportUrl: null
    }
  },
  watch: {
    showTable: function (val) {
      this.resizeChart()
    },
    showCharts: function (val) {
      this.resizeChart()
    },
    showPoints: function (val) {
      this.unselectAll()
      if(!this.pca) return
      for(var i=0;i<this.groups.length;i++){
        if(val){
          this.self.chart.show(['G' + i])
        }else{
          this.self.chart.hide(['G' + i])
        }
      }
    },
    showErrors: function (val) {
      this.unselectAll()
      if(!this.pca) return
      for(var i=0;i<this.groups.length;i++){
        if(val){
          this.self.chart.show(['G' + i + '_error'])
        }else{
          this.self.chart.hide(['G' + i + '_error'])
        }
      }
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
      var vm = this
      setTimeout(function(){
        vm.rescaleChart()
      }, 500)
    },
    chartHeightChanged () {
      this.resizeChart(this.chartHeight)
    },
    dataIndexChanged (variable) {
      if(this.variables.length > 1){
        this.showCompute = true
      }
    },
    addVariable () {
      var variable = {
        dataIndex: 0
      }
      this.variables.push(variable)
      if(this.variables.length > 1){
        this.showCompute = true
      }
    },
    deleteVariable (v) {
      var index = this.variables.indexOf(v)
      if(index > -1){
        this.variables.splice(index, 1)
      }
      if(this.variables.length > 1){
        this.showCompute = true
      }
    },
    computePCA () {
      this.groups = []
      this.activeGroup = 0
      this.xRange = null
      this.yRange = null
      this.self.chart.unload()

      this.computing = true
      var dataset = this.makeDataSet()
      this.pca = new PCA(dataset)
      this.variances = this.pca.getExplainedVariance()
      this.xyVectors = this.pca.getEigenvectors().slice(0, 2)
      var vectors = this.pca.predict(dataset)
      var group = {}
      for(var i=0;i<vectors.length;i++){
        var vector = vectors[i]
        var point = {}
        var rowNum = this.rows[i][0]
        point.rowNum = rowNum
        point.x = vector[0]
        point.y = vector[1]
        if(this.xRange){
          this.xRange[0] = Math.min(this.xRange[0], point.x)
          this.xRange[1] = Math.max(this.xRange[1], point.x)
          this.yRange[0] = Math.min(this.yRange[0], point.y)
          this.yRange[1] = Math.max(this.yRange[1], point.y)
        }else{
          this.xRange = [point.x, point.x]
          this.yRange = [point.y, point.y]
        }
        var sum = 0
        for(var j=2;j<vector.length;j++){
          sum += vector[j]*vector[j]
        }
        point.error = Math.sqrt(sum)
        var key = point.x + '_' + point.y
        if(!group[key]){
          group[key] = [point]
        }else{
          group[key].push(point)
        }
      }
      this.groups.push(group)

      var vm = this
      setTimeout(function(){
        vm.drawPCA()
        vm.rescaleChart()
        vm.computing = false
        vm.showCompute = false
      }, 500)
    },
    makeDataSet () {
      var dataset = []
      for(var i=0;i<this.rows.length;i++){
        var row = this.rows[i]
        var data = []
        for(var j=0;j<this.variables.length;j++){
          var index = this.variables[j].dataIndex
          var cell = row[index]
          data.push(Number(cell))
        }
        dataset.push(data)
      }
      return dataset
    },
    drawPCA () {
      for(var i=0;i<this.groups.length;i++){
        var points = []
        Object.values(this.groups[i]).forEach(function(pg){
          points = points.concat(pg)
        })

        var xData = points.map(function(p){
          return p.x
        })
        var xError = ['G'+i+'_error_x'].concat(xData)
        xData.unshift('G'+i+'_x')
        var yData = points.map(function(p){
          return p.y
        })
        yData.unshift('G'+i)
        var zError = points.map(function(p){
          return p.error
        })
        zError.unshift('G'+i+'_error')
        var xs = {}
        xs['G'+i] = 'G'+i+'_x'
        xs['G'+i+'_error'] = 'G'+i+'_error_x'
        this.self.chart.load({ xs: xs, columns: [ xData, yData, xError, zError], type: 'scatter' })
        if(this.showPoints){
          this.self.chart.show(['G' + i])
        }else{
          this.self.chart.hide(['G' + i])
        }
        if(this.showErrors){
          this.self.chart.show(['G' + i + '_error'])
        }else{
          this.self.chart.hide(['G' + i + '_error'])
        }
      }
    },
    rescaleChart () {
      var el = this.$el.querySelector('rect.c3-event-rect')
      if(!el) return
      var width = el.getAttribute('width')
      var height = el.getAttribute('height')
      var xInterval = (width - 10) / (this.xRange[1] - this.xRange[0])
      var yInterval = (height - 40) / (this.yRange[1] - this.yRange[0])
      if(xInterval > yInterval){
        var xCenter = (this.xRange[1] + this.xRange[0])/2
        var xRadius = (width - 10) / yInterval / 2
        var xRange = [xCenter - xRadius, xCenter + xRadius]
        this.self.chart.axis.range({
          max: {x: xRange[1], y: this.yRange[1]},
          min: {x: xRange[0], y: this.yRange[0]}
        })
      }else{
        var yCenter = (this.yRange[1] + this.yRange[0])/2
        var yRadius = (height - 40) / xInterval / 2
        var yRange = [yCenter - yRadius, yCenter + yRadius]
        this.self.chart.axis.range({
          max: {x: this.xRange[1], y: yRange[1]},
          min: {x: this.xRange[0], y: yRange[0]}
        })
      }
    },
    unselectAll () {
      for(var i=0;i<this.groups.length;i++){
        this.self.chart.unselect(['G'+i, 'G'+i+'_error'])
      }
    },
    chartMouseUp () {
      var vm = this
      setTimeout(function(){
        var selected = vm.self.chart.selected()
        if(selected.length){
          var aGroup = vm.groups[vm.activeGroup]
          for(var i=0;i<selected.length;i++){
            var pt = selected[i]
            if(pt.id.indexOf('error') > -1)
              continue
            var key = pt.x + '_' + pt.value
            if(aGroup[key])
              continue
            for(var j=0;j<vm.groups.length;j++){
              var g = vm.groups[j]
              if(g[key]){
                aGroup[key] = g[key]
                delete g[key]
                break;
              }
            }
          }
          vm.unselectAll()
          setTimeout(function(){
            vm.drawPCA ()
          }, 500)
        }
      }, 500)
    },
    addGroup () {
      this.groups.push({})
    },
    deleteGroup (index) {
      var group0 = this.groups[0]
      var group = this.groups[index]
      var keys = Object.keys(group)
      for(var i=0;i<keys.length;i++){
        var key = keys[i]
        group0[key] = group[key]
      }
      this.groups.splice(index, 1)
      this.activeGroup = 0
      
      this.self.chart.unload()
      var vm = this
      setTimeout(function(){
        vm.drawPCA ()
      }, 500)
    },
    makeExportUrl () {
      var points = {}
      for(var i=0;i<this.groups.length;i++){
        var group = this.groups[i]
        var pts = []
        Object.values(group).forEach(function(g){
          pts = pts.concat(g)
        })
        pts.forEach(function(p){
          points[p.rowNum] = i
        })
      }

      var rows = {}
      for(var i=0;i<this.rows.length;i++){
        var r = this.rows[i]
        var row = r.slice(1)
        row.push(points[r[0]])
        rows[r[0]] = row
      }

      var keys = Object.keys(rows)
      keys.sort(function(a, b){
        return Number(a) - Number(b)
      })
      var headers = this.headers.slice(1)
      headers.push('group')
      var csv = headers.join(',') + '\r\n'
      for(var i=0;i<keys.length;i++){
        csv += rows[keys[i]].join(',') + '\r\n'
      }
      var blob = new Blob([csv], {type: 'file'})
      this.exportUrl = URL.createObjectURL(blob)
    }
  },
  mounted () {
    this.$nextTick(function(){
      this.self.chart = c3.generate({
        bindto: '#chart' + this.chart.id,
        data: { 
          columns: [], 
          selection: {
            enabled: true,
            draggable: true
          }
        },
        type: 'scatter',
        axis: {
          x: { tick: { fit: false } }
        }
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

.delete-variable-button {
  position: relative;
  top: 3px;
}

.pca-info {
  font-size: 14px;
  margin-bottom: 3px;
}

.group-label {
  margin-top: 5px;
}


</style>
