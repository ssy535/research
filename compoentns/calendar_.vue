<template>
  <div>
    <el-date-picker
      v-model="val"
      type="date"
      :picker-options="pickerOptions"
      placeholder="날짜 선택"
    />
  </div>
</template>

<script>
  export default {
    props: ['value'],
    data() {
      return {
        ts: '',
        pickerOptions: {

          disabledDate(time) {
            return time.getTime() <= Date.now();
          }
          ,
          shortcuts: [
            {
              text: '오늘',
              onClick(picker) {
                picker.$emit('pick', new Date());
              }
            },
            {
              text: '일주일 뒤',
              onClick(picker) {
                picker.$emit('pick', (new Date()).getTime() + 3600 * 1000 * 24 * 7);
              }
            }]
        }
      }
    },
    computed: {
      val: {
        get() {
          if (this.ts === '' || _.isUndefined(this.ts)) {
            this.ts = this.value;
          }

          this.$emit('input', this.ts);
          return this.ts;
        },

        set(v) {
          this.ts = v;
        }
      }
    }
  }
</script>

