<template>
  <div>
    <div>
      <div class="columns">
        <div class="column">
          <input type="file" class="files-input" @change="onFileChange">
          &nbsp;&nbsp;
          <label class="checkbox">
            <input type="checkbox" v-model="showTable">
            Show Table
          </label>
          &nbsp;&nbsp;
          <label class="checkbox">
            <input type="checkbox" v-model="showCharts">
            Show Charts
          </label>
        </div>
      </div>
    </div>

    <div class="columns">
      <div class="column" v-show="showTable"
        :class="{'half-width': showTable && showCharts, 'full-width': showTable && !showCharts}">
        <div class="csv-table">
          <table class="table is-narrow" v-if="rows.length">
            <thead>
              <tr>
                <th class="number-cell">{{rows.length}}</th>
                <th class="number-cell" v-for="(h, i) in headers">
                  <div class="csv-header" @click="sortData(i)">
                    <span>{{h}}</span>
                    <span class="sort-icon" v-if="sortIndex==i">
                      <icon class="asc-icon" name="sort-asc" v-if="asc"></icon>
                      <icon name="sort-desc" v-if="!asc"></icon>
                    </span>
                  </div>
                </th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(row, i) in rows">
                <td class="number-cell">{{i}}</td>
                <td class="number-cell" v-for="cell in row">
                  {{cell}}
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div class="column" v-show="showCharts"
        :class="{'half-width': showTable && showCharts, 'full-width': !showTable && showCharts}">
        <div v-show="rows.length" class="buttons">
          <a class="button" @click="addSimpleChart"><icon name="plus"></icon>&nbsp;Simple Chart</a>
        </div>
        <div v-for="c in charts" :key="c.id">
          <simple-chart
            :chart="c"
            :headers="headers"
            :rows="rows"
            :showTable="showTable"
            :showCharts="showCharts"
            @delete-chart="deleteChart">
          </simple-chart>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import SimpleChart from './SimpleChart'

export default {
  name: 'main-window',
  components: {
    SimpleChart
  },
  data () {
    return {
      showTable: true,
      showCharts: true,
      headers: [],
      rows: [],
      sortIndex: 0,
      asc: true,
      charts: []
    }
  },
  methods: {
    onFileChange (e) {
      var files = e.target.files || e.dataTransfer.files
      if (!files.length)
        return
      var file = files[0]

      this.headers = []
      this.rows = []
      this.charts = []

      var vm = this
      var reader = new FileReader()
      reader.onload = function(e) {
        var text = reader.result
        var lines = text.split('\n')
        for(var i=0;i<lines.length;i++){
          var ss = lines[i].split(',')
          if(!ss.length) continue
          if(i == 0){
            vm.headers.push('Row')
            for(var j=0;j<ss.length;j++){
              vm.headers.push(ss[j].trim())
            }
          }else{
            var row = [vm.rows.length]
            for(var j=0;j<ss.length;j++){
              row.push(ss[j].trim())
            }
            vm.rows.push(row)
          }
        }
      }
      reader.readAsText(file)
    },
    sortData (index) {
      if(this.sortIndex == index){
        this.asc = !this.asc
      }else{
        this.sortIndex = index
      }
      var vm = this
      vm.rows.sort(function(a, b){
        return vm.compareData(a, b)
      })
    },
    compareData (a, b) {
      if(isNaN(a[this.sortIndex]) || isNaN(b[this.sortIndex])){
        if(this.asc){
          return a[this.sortIndex].localeCompare(b[this.sortIndex])
        }
        return -a[this.sortIndex].localeCompare(b[this.sortIndex])
      }

      if(this.asc){
        return a[this.sortIndex] - b[this.sortIndex]
      }
      return b[this.sortIndex] - a[this.sortIndex]
    },
    addSimpleChart () {
      var id = this.charts.length ? (this.charts[0].id + 1) : 0
      var chart = {
        id: id,
        type: 'simple'
      }
      this.charts.unshift(chart)
    },
    deleteChart (chart) {
      var index = this.charts.indexOf(chart)
      if(index > -1){
        this.charts.splice(index, 1)
      }
    }
  },
  mounted () {
    
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="scss" scoped>

.files-input {
  font-size: 14px;
}

.full-width {
  width: 100%;
}

.half-width {
  width: 50%;
}

.csv-table {
  max-width: 100%;
  overflow-x: auto;
}

.csv-header {
  white-space: nowrap;
  cursor: pointer;
}

.asc-icon {
  position: relative;
  top: 5px;
}

.number-cell {
  text-align: right;
}

.buttons {
  text-align: center;
}

</style>
