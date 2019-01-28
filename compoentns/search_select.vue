<template>
  <div>
    <el-select :value="value" @change="onChange">
      <el-option
        v-for="item in pageSet"
        :key="item.value"
        :label="item.name"
        :value="item.value"/>
    </el-select>
  </div>
</template>


<script>
  export default {
    props: ['value'],
    data () {
      return {
        pageSet : [
          {'name' : '10개 보기', 'value' : 10},
          {'name' : '15개 보기', 'value' : 15},
          {'name' : '20개 보기', 'value' : 20},
          {'name' : '30개 보기', 'value' : 30},
          {'name' : '50개 보기', 'value' : 50},
          {'name' : '100개 보기', 'value' : 100}
        ]
      }
    }, methods : {
      onChange(value) {
        if (value === '') {
          value = null;
        }
        this.$emit('onChange', value);

      }
    }
  }
</script>
