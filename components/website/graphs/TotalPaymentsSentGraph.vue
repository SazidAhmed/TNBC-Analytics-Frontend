<template>
  <div class="relative w-full card">
    <highcharts
      v-if="totalTxs"
      :constructor-type="'stockChart'" 
      :options="transactions">
    </highcharts>
    <div v-else><CircleSpinner class="h-16 w-16 mx-auto" /></div>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'nuxt-property-decorator'
import CircleSpinner from '@/components/website/spinner/CircleSpinner.vue'
import moment from 'moment'

@Component({
  components: {
    CircleSpinner
  }
})
export default class GovernmentGraph extends Vue {
  totalTxs: any = null

  async fetch() {
    const startDate = moment().subtract(12, 'month').format('YYYY-MM-DD')
    const endDate = moment().format('YYYY-MM-DD')
    this.totalTxs = await fetch(`http://bank.tnbexplorer.com/stats/api/?start=${startDate}&end=${endDate}`)
      .then(res => res.json()).catch(err => console.log(err))
  }

  get graphData(): any {
    if (this.totalTxs && this.totalTxs.length) {
      const value = this.totalTxs[this.totalTxs.length - 1].total
      
      this.setDistributedCoins(value)
      this.reduceTotalData(this.totalTxs)  
      
      const cumulatedData = this.cumulateTotalData(this.totalTxs)
      return cumulatedData
    }
    return null
  }

  setDistributedCoins(value: number): void {
    this.$emit('setDistributedCoins', value)
  }

  reduceTotalData(array): any {
    return array.reduce((previousTotal: number, record: any) => {
        record.changeInCoins = record.total - previousTotal;
        return record.total;
      }, 0);
  }

  cumulateTotalData(array): any {
    let cumulatedData: any = []

    array.forEach((data: any) => {
      const date = moment.utc(data.date).format()
      const formatedDate = moment(data.date).valueOf()
      if (cumulatedData.length === 0) {
        cumulatedData.push([
          formatedDate,
          data.changeInCoins,
        ]);
      } else {
        const prev = cumulatedData[cumulatedData.length - 1]
        if (prev[0] !== date) {

          cumulatedData.push([
            formatedDate,
            data.changeInCoins,
          ]);
        } else {
          prev.changeInCoins += data.changeInCoins;
        }
      }
    })
    return cumulatedData;
  }

  get transactions(): any {
    let chartOptions: any =
    {
      chart: {
        type: 'areaspline',
        alignTicks: false
      },
      boost: {
        enabled: false
      },
      title: {
        text: '',
        margin: 30,
        align: 'left'
      },
      subtitle: {
        text: '',
        align: 'left'
      },
      tooltip: {
        shared: true,
        valueSuffix: ' TNBC'
      },
      rangeSelector: {
        selected: 1
      },
      series: [{
        name: 'Transactions',
        turboThreshold: 20000,
        data: this.graphData,
        dataGrouping: {
          approximation: 'sum',
          enabled: true,
          forced: true,
          units: [[
            'month', // unit name
            [1] // allowed multiples
          ]]
        },
      }]
    }

    return chartOptions
  }

}
</script>