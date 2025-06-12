<template>
  <div class="w-100">
    <div class="row">
      <div class="col-12 col-md-6">
        <div>
          <label for="file">Raw File</label><br>
          <small><i>Only accepts CSV</i></small>
          <input type="file" class="form-control" id="file" @change="onFileChange" accept=".csv" :multiple="false" />
        </div>
        <div class="mt-3">
          <label for="file">Employer Name</label><br>
          <input type="text" class="form-control" id="employer" v-model="data.employer">
        </div>
      </div>
      <div class="col-12 col-md-6">
        <div class="row justify-content-between align-items-end">
            <div class="col-5">
              <label for="curent-payroll-first">Current Payroll Dates</label>
              <select name="Current Payroll - First Due Date" id="curent-payroll-first" class="form-control" v-model="data.currentPayroll[0]" @change="recomputeOutput">
                <option :value="null"></option>
                <option v-for="value in payrollDates" :value="value" :key="`${value}_current_first_payroll`">{{ value }}</option>
              </select>
              <select name="Current Payroll - Second Due Date" id="curent-payroll-second" class="form-control mt-1" v-model="data.currentPayroll[1]" @change="recomputeOutput">
                <option :value="null"></option>
                <option v-for="value in payrollDates" :value="value" :key="`${value}_current_second_payroll`">{{ value }}</option>
              </select>
            </div>
            <div class="col-2 text-center">
              <h2 class="m-0">
                ➡️
              </h2>
              <h2 class="m-0 mt-1">
                ➡️
              </h2>
            </div>
            <div class="col-5">
              <label for="new-payroll-first">New Payroll Dates</label>
              <select name="New Payroll - First Due Date" id="new-payroll-first" class="form-control" v-model="data.newPayroll[0]" @change="recomputeOutput">
                <option :value="null"></option>
                <option v-for="value in payrollDates" :value="value" :key="`${value}_new_first_payroll`">{{ value }}</option>
              </select>
              <select name="New Payroll - Second Due Date" id="new-payroll-second" class="form-control mt-1" v-model="data.newPayroll[1]" @change="recomputeOutput">
                <option :value="null"></option>
                <option v-for="value in payrollDates" :value="value" :key="`${value}_new_second_payroll`">{{ value }}</option>
              </select>
            </div>
        </div>
      </div>
    </div>
    <hr class=" mt-4">
    <div class="row mt-3 w-100">
      <h5>Output</h5>
      <small v-if="(outputMapped || []).length > 1000"><i>Note: Only the first {{ displayLimit }} records (out of
          {{
          (outputMapped || []).length }})
          have been displayed.</i></small>
      <div class="my-2 px-0 overflow-scroll output border rounded shadow">
        <table class="table table-striped table-sm table-bordered table-hover">
          <thead>
            <tr class="table-primary">
              <th scope="col" v-for="header in headers" :key="header">
                <p class="mb-0">{{ header }}</p>
              </th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(row, rowIndex) in (outputMapped || []).slice(0, displayLimit)" :key="`row-${rowIndex}`">
              <td scope="row" v-for="(value, columnIndex) in row" :key="`${columnIndex}-column`">{{ value }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
    <hr class="mt-4">
    <div class="row mt-3 w-100">
      <h5>Checks</h5>
      <h6 :class="`p-2 w-25 rounded text-center fw-bold text-uppercase text-white ${this.fullValid ? 'bg-success' : 'bg-error'}`">{{ this.fullValid ? 'Checks passed' : 'Checks did not pass' }}</h6>
      <div class="col-12">
        <table class="table table-bordered table-hover">
          <thead>
            <th scope="col">Due Dates</th>
            <th class="text-end" scope="col">Diff in Amounts</th>
            <th class="text-end" scope="col">Diff between Old and New</th>
          </thead>
          <tbody>
            <tr v-for="(rowValue, rowDate) in checks" :key="rowDate">
              <td>{{ rowDate }}</td>
              <td :class="`text-end ${rowValue.amountValid ? 'text-success' : 'text-error fw-bold'}`">{{ rowValue.amount }}</td>
              <td :class="`text-end ${rowValue.diffValid ? 'text-success' : 'text-error fw-bold'}`">{{ rowValue.diffBetweenOldNew }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
    <hr class="mt-4">
    <div class="row mt-3 w-100">
      <h5>Actions</h5>
      <small v-if="(outputMapped || []).length > csvLimit"><i>Note: the files downloaded will be split into {{
          csvLimit
          }} records
          chunks.</i></small>
      <div class="row">
        <div class="col-3">
          <p class="py-1"></p>
          <button class="form-control btn btn-primary" @click="download">Download</button>
        </div>
        <div class="col-3">
          <p class="py-1"></p>
          <button class="form-control btn btn-light" @click="clear">Clear</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
  const CACHE_KEY = 'TENDOPAY_RESCHEDULER'
  const FORCE_CACHE = true
  const CSV_LIMIT = 50000
  const DISPLAY_LIMIT = 100
  const HEADERS = [
    'user_id',
    'employer_id',
    'schedule_id',
    'transaction_id',
    'as_is_due_date',
    'to_be_due_date',
    'amount',
    'repaid',
    'outstanding'
  ]

  export default {
    data() {
      return {
        data: {
          raw: null,
          employer: null,
          currentPayroll: [],
          newPayroll: []
        },
        cacheProcessed: false,
        headers: HEADERS,
        displayLimit: DISPLAY_LIMIT,
        csvLimit: CSV_LIMIT,
        output: null,
        outputMapped: null
      }
    },
    mounted() {
      this.restoreCache()
      this.recomputeOutput()
    },
    computed: {
      inputHeaders () {
        return (this.data.raw || '')
          .split(/\r?\n|\r|\n/g)[0].split(/\t/g)
      },
      payrollDates() {
        return (new Array(30)).fill(null).map((_, i) => i + 1)
      },
      asIsDueDateIndex() {
        return this.headers.indexOf('as_is_due_date')
      },
      toBeDueDateIndex() {
        return this.headers.indexOf('to_be_due_date')
      },
      outstandingIndex() {
        return this.headers.indexOf('outstanding')
      },
      checks() {
        const oldPayroll = {};
        const newPayroll = {};
        const result = {};
        (this.outputMapped || []).forEach(_ => {
          oldPayroll[`${_[this.asIsDueDateIndex]}`] = oldPayroll[`${_[this.asIsDueDateIndex]}`] || 0
          newPayroll[`${_[this.toBeDueDateIndex]}`] = newPayroll[`${_[this.toBeDueDateIndex]}`] || 0
          oldPayroll[`${_[this.asIsDueDateIndex]}`] += Math.round((parseFloat((_[this.outstandingIndex] || 0))) * 100) / 100
          newPayroll[`${_[this.toBeDueDateIndex]}`] += Math.round((parseFloat((_[this.outstandingIndex] || 0))) * 100) / 100
          if(_[this.asIsDueDateIndex] || _[this.toBeDueDateIndex]) {
            result[`${_[this.asIsDueDateIndex]} → ${_[this.toBeDueDateIndex]}`] = {
              old: _[this.asIsDueDateIndex],
              new: _[this.toBeDueDateIndex],
              diffBetweenOldNew: ((new Date(_[this.toBeDueDateIndex])) - (new Date(_[this.asIsDueDateIndex]))) / (60 * 60 * 24 * 1000)
            }
          }
        })
        Object.keys(result).forEach(_ => {
          result[_] = {
            ...result[_],
            amount: oldPayroll[result[_].old] - newPayroll[result[_].new],
            amountValid: (oldPayroll[result[_].old] - newPayroll[result[_].new]) == 0,
            diffValid: result[_].diffBetweenOldNew >= this.checkDiffDays.min && result[_].diffBetweenOldNew <= this.checkDiffDays.max
          }
        })
        return result
      },
      checkDiffDays() {
        try {
          const diff = Math.min(Math.abs(this.data.newPayroll[0] - this.data.currentPayroll[0]), Math.abs(this.data.newPayroll[1] - this.data.currentPayroll[1]))
          return {
            min: diff - 1,
            max: diff + 1
          }
        } catch (e) {
          return {
            min: 4,
            max: 6
          }
        }
      },
      fullValid() {
        return Object.values(this.checks).reduce((acc, _) => acc && _.amountValid && _.diffValid, true)
      }
    },
    methods: {
      onFileChange(e) {
        var file = (e.target.files || e.dataTransfer.files)[0];
        var reader = new FileReader()
        reader.readAsText(file, "UTF-8")
        reader.onload = (evt) => {
          this.data.raw = evt.target.result.replace(/,/g, '\t')
          this.recomputeOutput()
        }
      },
      recomputeOutput() {
        try {
          this.output = (this.data.raw || '')
            .split(/\r?\n|\r|\n/g)
            .filter(_ => _ != "")
            .map(_ => _.split(/\t/g))
            .slice(1)
            .filter(_ => _.length > 0)
          this.recomputeOutputMapped()
        } catch (e) {
          //
        }
      },
      recomputeOutputMapped() {
        try {
          this.outputMapped = this.output.map(_ => {
            return this.headers.map(__ => {
              let value = this.findValue(__, _)
              if (__ == 'to_be_due_date') {
                const date = new Date(this.findValue('as_is_due_date', _));
                const year = date.getFullYear();
                const month = date.getMonth();
                const day = date.getDate();

                if (day === this.data.currentPayroll[0]) {
                    if (this.data.newPayroll[0] < this.data.currentPayroll[0]) {
                      value = new Date(year, month + 1, this.data.newPayroll[0] + 1).toISOString().split('T')[0];
                    } else {
                      value = new Date(year, month, this.data.newPayroll[0] + 1).toISOString().split('T')[0];
                    }
                } else if (day === this.data.currentPayroll[1] || day === (new Date(year, month + 1, 0)).getDate()) {
                    if (this.data.newPayroll[1] < this.data.currentPayroll[1]) {
                      value = new Date(year, month + 1, this.data.newPayroll[1] + 1).toISOString().split('T')[0];
                    } else {
                      value = new Date(year, month, this.data.newPayroll[1] + 1).toISOString().split('T')[0];
                    }
                }
              }
              return value
            })
          })

          if (this.cacheProcessed) {
            this.storeCache()
          }
        } catch (e) {
          console.log(e)
        }
      },
      findValue(header, row) {
        try {
          const inputHeader = header
          if (!inputHeader) {
            return ''
          }
          const inputIndex = this.inputHeaders.indexOf(inputHeader)
          return row[inputIndex]
        } catch (e) {
          return ''
        }
      },
      storeCache() {
        const obj = {
          data: {
            ...this.data
          }
        }
        localStorage.setItem(CACHE_KEY, JSON.stringify(obj))
      },
      restoreCache() {
        const res = localStorage.getItem(CACHE_KEY)
        if (res) {
          let clear = {}
          try {
            clear = JSON.parse(res)
          } catch (err) {
            clear = {}
          }
          if (!FORCE_CACHE) {
            this.clear()
            console.log(`* ELS FORMATTER: Invalidated cache`)
          } else {
            const data = clear.data
            Object.keys(data || {}).forEach(_ => {
              if (data[_] != null) {
                this.data[_] = data[_]
              }
            })
            console.log(`* ELS FORMATTER: Restored cache`)
          }
        }
        this.cacheProcessed = true
      },
      clear() {
        Object.keys(this.data || {}).forEach(_ => {
          if (typeof this.data[_] === 'object' && this.data[_] !== null) {
            this.data[_] = []
          } else {
            this.data[_] = null
          }
        })
        localStorage.removeItem(CACHE_KEY)
        this.recomputeOutput()
      },
      download() {
        const date = new Date();
        const yyyy = date.getFullYear();
        const mm = String(date.getMonth() + 1).padStart(2, '0')
        const dd = String(date.getDate()).padStart(2, '0')
        const formattedDate = `${yyyy}_${mm}_${dd}`

        let data = []
        let files = []
        for (let i = 0; i < Math.ceil(this.outputMapped.length / CSV_LIMIT); i++) {
          data.push(this.outputMapped.slice(i * CSV_LIMIT, i * CSV_LIMIT + CSV_LIMIT))
        }
        data.forEach(_ => {
          let csvContent = "data:text/csv;charset=utf-8,"
            + [
              this.headers,
              ..._
            ].map(e => e.join(",")).join("\n");

          files.push(encodeURI(csvContent));
        })

        const link = document.createElement("a")
        document.body.appendChild(link)

        files.forEach((_, index) => {
          link.setAttribute('href', _)
          link.setAttribute('download', `${data.employer || ''}_rescheduling_${formattedDate}_${index + 1}.csv`)

          link.click()
        })
      }
    }
  }
</script>

<style scoped lang="scss">
.output {
  max-height: 50vh;
}

.row {
  margin-right: 0 !important;
  margin-left: 0 !important;
}
</style>