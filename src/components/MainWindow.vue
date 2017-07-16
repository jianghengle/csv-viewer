<template>
  <div>
    <div>
      <div class="columns">
        <div class="column">
          <input type="file" class="files-input" @change="onFileChange">
          &nbsp;&nbsp;
          <label class="checkbox" v-if="rows.length">
            <input type="checkbox" v-model="showTable">
            Table
          </label>
          &nbsp;&nbsp;
          <label class="checkbox" v-if="rows.length">
            <input type="checkbox" v-model="showCharts">
            Charts
          </label>
        </div>
      </div>
    </div>

    <div class="columns">
      <div class="column" v-show="showTable"
        :class="{'half-width': showTable && showCharts, 'full-width': showTable && !showCharts}">
        <div v-if="totalCount">
          <nav class="pagination">
            <ul class="pagination-list">
              <li v-show="currentPage > 0"><a class="pagination-link default-btn" @click="showPage(0)">P1</a></li>
              <li v-show="currentPage-1 > 0"><a class="pagination-link default-btn" @click="showPage(currentPage-1)">Pre</a></li>
              <li><a class="pagination-link is-current active-btn">{{currentRange}}</a></li>
              <li v-show="currentPage+1 < totalPages-1"><a class="pagination-link default-btn" @click="showPage(currentPage+1)">Next</a></li>
              <li v-show="currentPage < totalPages-1"><a class="pagination-link default-btn" @click="showPage(totalPages-1)">P{{totalPages}}</a></li>
            </ul>
          </nav>
        </div>
        <div class="csv-table">
          <table class="table is-narrow" v-if="totalCount">
            <thead>
              <tr>
                <th class="number-cell">{{totalCount}}</th>
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
              <tr v-for="(row, i) in page">
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
          <a class="button" @click="addSimpleChart"><icon name="plus"></icon>&nbsp;Simple</a>
          <a class="button" @click="addSimpleXY"><icon name="plus"></icon>&nbsp;XY</a>
          <a class="button" @click="addParallelCoordinates"><icon name="plus"></icon>&nbsp;Parallel Coordinates</a>
          <a class="button" @click="addPcaChart"><icon name="plus"></icon>&nbsp;PCA</a>
        </div>
        <div v-for="c in charts" :key="c.id">
          <simple-chart
            v-if="c.type == 'Simple'"
            :chart="c"
            :headers="headers"
            :rows="rows"
            :showTable="showTable"
            :showCharts="showCharts"
            @delete-chart="deleteChart">
          </simple-chart>
          <simple-xy
            v-if="c.type == 'XY'"
            :chart="c"
            :headers="headers"
            :rows="rows"
            :showTable="showTable"
            :showCharts="showCharts"
            @delete-chart="deleteChart">
          </simple-xy>
          <parallel-coordinates
            v-if="c.type == 'Parallel Coordinates'"
            :chart="c"
            :headers="headers"
            :rows="rows"
            :showTable="showTable"
            :showCharts="showCharts"
            @delete-chart="deleteChart">
          </parallel-coordinates>
          <pca-chart
            v-if="c.type == 'PCA'"
            :chart="c"
            :headers="headers"
            :rows="rows"
            :showTable="showTable"
            :showCharts="showCharts"
            @delete-chart="deleteChart">
          </pca-chart>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import SimpleChart from './SimpleChart'
import SimpleXy from './SimpleXy'
import ParallelCoordinates from './ParallelCoordinates'
import PcaChart from './PcaChart'

export default {
  name: 'main-window',
  components: {
    SimpleChart,
    SimpleXy,
    ParallelCoordinates,
    PcaChart
  },
  data () {
    return {
      showTable: true,
      showCharts: true,
      headers: [],
      rows: [],
      sortIndex: 0,
      asc: true,
      charts: [],
      currentPage: 0,
      pageSize: 1000
    }
  },
  computed: {
    page () {
      var index = this.currentPage * this.pageSize
      var page = this.rows.slice(index, index + this.pageSize)
      return page
    },
    totalCount () {
      return this.rows.length
    },
    totalPages () {
      return Math.ceil(this.totalCount / this.pageSize)
    },
    currentRange () {
      var start = this.currentPage * this.pageSize
      var end = (this.currentPage + 1) * this.pageSize - 1
      if(end > this.totalCount - 1){
        end = this.totalCount - 1
      }
      return start + '~' + end
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
        var re=/\r\n|\n\r|\n|\r/g
        var lines = text.replace(re, '\n').split('\n')
        for(var i=0;i<lines.length;i++){
          if(!lines[i].trim().length) continue
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
        type: 'Simple'
      }
      this.charts.unshift(chart)
    },
    addSimpleXY () {
      var id = this.charts.length ? (this.charts[0].id + 1) : 0
      var chart = {
        id: id,
        type: 'XY'
      }
      this.charts.unshift(chart)
    },
    addParallelCoordinates () {
      var id = this.charts.length ? (this.charts[0].id + 1) : 0
      var chart = {
        id: id,
        type: 'Parallel Coordinates'
      }
      this.charts.unshift(chart)
    },
    addPcaChart () {
      var id = this.charts.length ? (this.charts[0].id + 1) : 0
      var chart = {
        id: id,
        type: 'PCA'
      }
      this.charts.unshift(chart)
    },
    deleteChart (chart) {
      var index = this.charts.indexOf(chart)
      if(index > -1){
        this.charts.splice(index, 1)
      }
    },
    showPage(pageNumber){
      this.currentPage = pageNumber
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
  max-width: 100%;
}

.half-width {
  width: 50%;
  max-width: 50%;
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
  font-size: 14px;
}

.buttons {
  padding-left: 20px;
}

</style>
