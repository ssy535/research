<template>
  <div>
    <el-date-picker
      v-model="val"
      type="daterange"
      range-separator="-"
      :picker-options="pickerOptions"
      start-placeholder="시작일"
      end-placeholder="종료일">
    </el-date-picker>
  </div>
</template>

<script>
  export default {
    props: ['startDate', 'endDate', 'value'],
    data() {
      return {
        ts: '',
        te: '',
        pickerOptions: {

          disabledDate(time) {
            return time.getTime() <= Date.now();
          }
          ,
          shortcuts: [
            {
              text: '오늘',
              onClick(picker) {
                let start = new Date();
                if (!_.isEmpty(picker.selectedDate))
                  start = picker.selectedDate[0];

                const end = new Date();
                picker.$emit('pick', [start, end]);
              }
            },
            {
              text: '일주일 뒤',
              onClick(picker) {
                let start = new Date();
                if (!_.isEmpty(picker.selectedDate))
                  start = picker.selectedDate[0];

                let end = new Date();
                end.setTime((new Date(start)).getTime() + 3600 * 1000 * 24 * 7);
                console.log(picker)
                picker.$emit('pick', [start, end]);
              }
            }]
        }
      }
    },
    computed: {
      val: {
        get() {
          if (this.ts === '' || _.isUndefined(this.ts)) {
            this.ts = this.startDate;
          }

          if (this.te === '' || _.isUndefined(this.te)) {
            this.te = this.endDate;
          }

          let ret = '';

          if (!_.isUndefined(this.ts) && !_.isUndefined(this.te)) {
            ret = [this.ts, this.te];
          }

          this.$emit('input', ret);
          return ret;
        },

        set(v) {
          this.ts = v[0];
          this.te = v[1];
        }
      }
    }
  }
</script>

