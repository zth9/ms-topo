<template>
  <div>
    <el-row>
      <el-table :data="dubboInvokeInfoList" v-loading="tableLoading"
                element-loading-text="查询中，请稍等..."
                element-loading-spinner="el-icon-loading"
                element-loading-background="rgba(255,255,255,0.75)">
        <el-table-column prop="interfaceName" label="接口名"></el-table-column>
      </el-table>
    </el-row>
  </div>
</template>

<script>

export default {
  name: 'relation',
  components: {},
  props: {
    currentNode: {
      type: '',
      source: '',
      target: '',
      name: '',
    }
  },
  watch: {
    currentNode: {
      handler(newVal) {
        this.dubboInvokeInfoList = [];
        this.init();
      },
      deep: true
    }
  },
  data() {
    return {
      tableLoading: false,
      dubboInvokeInfoList: []
    }
  },
  created() {
  },
  mounted: function () {
    this.init();
  },

  methods: {
  }
}
</script>

<style>
</style>