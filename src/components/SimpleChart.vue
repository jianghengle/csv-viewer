<template>
  <div class="chart-block">
    <div class="chart-header">
      <span class="chart-title">Chart {{chart.id}}</span>
      <a class="delete delete-button" @click="deleteChart"></a>
    </div>
    <div :id="'chart'+chart.id"></div>
    <div class="chart-options">
      <div class="variable field" v-for="(v, i) in variables">
        <input type="checkbox" v-model="v.load" @click="toggleColumn(v)">
        <select class="select" v-model="v.newIndex" @change="selectChanged(v)">
          <option v-for="(h, j) in headers" v-bind:value="j">
            {{h}}({{i}})
          </option>
        </select>
        &nbsp;&nbsp;
      </div>
      <a class="button is-small" @click="addVariable"><icon name="plus" scale="0.6"></icon>&nbsp;Variable</a>
    </div>
  </div>
</template>

<script>
import c3 from 'c3'

var chart

export default {
  name: 'simple-chart',
  props: ['chart', 'headers', 'rows', 'showTable', 'showCharts'],
  data () {
    return {
      variables: [],
    }
  },
  computed: {
    allVariables () {
      var variables = []
      for(var i=0;i<this.headers.length;i++){
        var variable = {
          index: i,
          name: this.headers[i]
        }
        variables.push(variable)
      }
      return variables
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
      this.updateAllColumns()
    }
  },
  methods: {
    updateColumn (variable) {
      if(variable.oldIndex != null && variable.oldIndex != variable.newIndex){
        var oldName = this.headers[variable.oldIndex] + '(' + variable.index + ')'
        chart.unload({ ids: oldName })
      }

      var vm = this
      setTimeout(function () {
        var newName = vm.headers[variable.newIndex] + '(' + variable.index + ')'
        var data = [newName]
        for(var i=0;i<vm.rows.length;i++){
          data.push(Number(vm.rows[i][variable.newIndex]))
        }
        chart.load({ columns: [ data ]})
        variable.oldIndex = variable.newIndex
      }, 500)
    },
    deleteChart () {
      this.$emit('delete-chart', this.chart)
    },
    selectChanged (variable) {
      this.updateColumn(variable)
    },
    addVariable () {
      var variable = {
        index: this.variables.length,
        oldIndex: null,
        newIndex: 0,
        load: true
      }
      this.variables.push(variable)
      this.updateColumn(variable)
    },
    resizeChart () {
      chart.resize()
    },
    updateAllColumns () {
      for(var i=0;i<this.variables.length;i++){
        this.updateColumn(this.variables[i])
      }
    },
    toggleColumn (variable) {
      this.$nextTick(function(){
        var v = this.variables[variable.index]
        var name = this.headers[v.newIndex] + '(' + v.index + ')'
        if(v.load){
          var data = [name]
          for(var i=0;i<this.rows.length;i++){
            data.push(Number(this.rows[i][v.newIndex]))
          }
          chart.load({ columns: [ data ] })
        }else{
          chart.unload({ ids: name })
        }
      })
    }
  },
  mounted () {
    this.$nextTick(function(){
      chart = c3.generate({
        bindto: '#chart' + this.chart.id,
        data: { columns: [] },
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

.delete-button {
  position: absolute;
  right: 20px;
}

.chart-options {
  display: inline-block;
}

.variable {
  display: inline-block;
}

.suffix {
  font-size: 12px;
}

</style>
