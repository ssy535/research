<template>
  <div>
    <el-select :value="value" @change="onChange" remote @focus="getCode" :loading="loading" clearable>
      <el-option
        v-for="item in options"
        :key="item.value"
        :label="item.name"
        :value="item.value"/>
    </el-select>
  </div>
</template>

<script>
  import request from '@/utils/request'

  const wether = [
    {name: 'Y', value: 'Y'},
    {name: 'N', value: 'N'}
  ]

  export default {
    props: ['value', 'cdGrpId'],
    data() {
      return {
        options: [],
        loading: false
      }
    },
    methods: {
      getCode() {
        if(!_.isUndefined(this.options) && !_.isEmpty(this.options)) {
          return Promise.resolve();
        }

        if (_.isEqual(this.cdGrpId, 'wether')) {
          this.options = wether;
          return ;
        }

        this.loading = true;

        return request({
          url: '/commoncode/' + this.cdGrpId + '/keyvalues',
          method: 'GET'
        }).then((res) => {
          this.options = res.data
          this.loading = false;
        })
      },
      onChange: function(value) {
        if (value === '') {
          value = null;
        }
        this.$emit('input', value);
      }
    },

  }
</script>

<style scoped>

</style>
